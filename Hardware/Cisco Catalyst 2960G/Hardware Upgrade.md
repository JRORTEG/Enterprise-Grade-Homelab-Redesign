## 1. Factory Reset
Was already done

## 2. Initial setup
Configured with IP: 10.0.10.12 /24
Hostname: SW1_2960G

## 3. Configure SSH
```
enable
configure terminal
ip domain-name home.arpa
username [username] privilege 15 secret [password]
crypto key generate rsa
```

The command block above wasn't enough on its own to get a working SSH session. Hit a chain of failures getting from "config applied" to "actually connects." Logging the real sequence here.

**RSA key size:** generated the key at a 2048-bit modulus. `ip ssh version 2` (below) rejects anything under 768 bits, and the default modulus is too small. If a low-bit key already exists, zeroize it first:
```
crypto key zeroize rsa
crypto key generate rsa
! modulus size: 2048
```

**Force SSH version 2:** the switch defaults to offering SSHv1, which modern OpenSSH clients refuse outright (`Protocol major versions differ: 2 vs. 1`):
```
ip ssh version 2
```

**VTY login method:** `line vty 0` needs `login local`. Without it the switch has no way to authenticate a session even with everything else right. Covered in the VTY section below.

**Reaching 10.0.10.12 from Trusted Clients (VLAN 60):** needed the switch's `ip default-gateway 10.0.10.1` (in `Build VLAN database and management SVI` above) plus a pfSense rule permitting Trusted Clients → 10.0.10.12, TCP 22 and ICMP. Without the gateway, the switch has no route back to VLAN 60 and the connection just times out.

**Client-side legacy crypto:** this switch is running a 2010-era IOS image (`c2960-lanbasek9-mz.122-50.SE5`), so it only offers KEX/cipher/MAC algorithms modern OpenSSH clients disable by default. Hit `no matching key exchange method found`, then `no matching cipher found`, then `no matching MAC found` in sequence as each was overridden. Fixed by pinning legacy algorithms for this one host in `~/.ssh/config` instead of weakening the client globally:
```
Host 10.0.10.12
    KexAlgorithms +diffie-hellman-group1-sha1
    HostKeyAlgorithms +ssh-rsa
    Ciphers aes128-cbc,aes256-cbc
    MACs +hmac-sha1
```
Underlying cause (outdated IOS, not just a config gap) logged as an open item, see `Open Items.md` #25.

## 4. Configure VTY line 0 | Disable the rest
 ```
 enable
 configure terminal
 line vty 0
 login local
 transport input ssh
 exit
 line vty 1 15
 no exec
 
 ```

## 5. Configure g0/21 as uplink

pfSense runs as a VM on the Proxmox host, not a separate box, so this port isn't a link to a standalone firewall. It's Proxmox's single trunk NIC, carrying pfSense's inter-VLAN routing legs plus any Proxmox-hosted VM/LXC traffic across VLANs. Proxmox's WAN NIC goes straight to the modem and never touches this switch.

```
interface g0/21
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,50,60,70,80
```

## Build VLAN database and management SVI

```
vlan 10
name MGMT
vlan 20
name CORE
vlan 30
name APPS
vlan 40
name DMZ
vlan 50
name LAB
vlan 60
name TRUSTED_CLIENTS
vlan 70
name GUEST
vlan 80
name IOT
exit

interface vlan 10
ip address 10.0.10.12 255.255.255.0
no shutdown
exit

ip default-gateway 10.0.10.1
```

## Port assignment

| Port    | Device                                           | VLAN                 | Mode   |
| ------- | ------------------------------------------------ | -------------------- | ------ |
| `g0/1`  | Main PC                                          | 60 (Trusted Clients) | access |
| `g0/2`  | NAS (TrueNAS)                                    | 20 (Core)            | access |
| `g0/3`  | PS5                                              | 70 (Guest)           | access |
| `g0/4`  | Docking Station                                  | 70 (Guest)           | access |
| `g0/5`  | Trusted Access Point                             | 60 (Trusted Clients) | access |
| TBD     | TP-Link EAP723 (Trusted/Guest/IoT SSIDs)         | 60, 70, 80           | trunk  |
| TBD     | Davolink Minions router (novelty Guest AP)       | 70 (Guest)           | access |
| `g0/21` | Proxmox PC (carries virtual pfSense + all VLANs) | all                  | trunk  |

PS5 and dock ports are configured for VLAN 70, which exists in this switch's VLAN database now, but VLAN 70 still isn't built on pfSense (no interface, no DHCP scope), see `Open Items.md` #11. Traffic on `g0/3`/`g0/4` won't pass until that's done.

VLAN 70 used to be a combined GUEST/IoT segment; split it into GUEST (70) and a new IOT (80) once I settled on the access point plan — see `Hardware/Access Point/Plan.md`. VLAN 80 needs its own switch VLAN database entry and pfSense build-out, same as 70, before either new AP port can pass IoT traffic — `Open Items.md` #29.

EAP723 and Minions router ports aren't assigned yet — no physical switch to wire into until the move. EAP723 gets a trunk carrying only the three VLANs it serves (`60,70,80`), narrower than the Proxmox trunk on `g0/21` which has to carry everything. Minions router gets a plain access port on VLAN 70 only, deliberately never trunked — see `Hardware/Access Point/Plan.md` for why.

```
interface g0/1
switchport mode access
switchport access vlan 60

interface g0/2
switchport mode access
switchport access vlan 20

interface g0/3
switchport mode access
switchport access vlan 70

interface g0/4
switchport mode access
switchport access vlan 70

interface g0/21
switchport mode trunk
switchport trunk allowed vlan 10,20,30,40,50,60,70,80

! EAP723 - port TBD
interface gX/X
switchport mode trunk
switchport trunk allowed vlan 60,70,80

! Davolink Minions router - port TBD
interface gX/X
switchport mode access
switchport access vlan 70
```

## Disable unused ports

Shut down every port not tied to a device. No reason to leave unassigned ports enabled.

```
interface range g0/6 - 20
shutdown
exit
interface range g0/22 - 24
shutdown
```

`show int status` confirms only `g0/1`-`g0/5` (assigned devices) and `g0/21` (trunk) are up; everything else disabled:

```
Port      Name               Status       Vlan       Duplex  Speed Type
Gi0/1                        connected    60         a-full a-1000 10/100/1000BaseTX
Gi0/2                        connected    20         a-full a-1000 10/100/1000BaseTX
Gi0/3                        notconnect   70           auto   auto 10/100/1000BaseTX
Gi0/4                        connected    70         a-full a-1000 10/100/1000BaseTX
Gi0/5                        connected    60         a-full a-1000 10/100/1000BaseTX
Gi0/6                        disabled     1            auto   auto 10/100/1000BaseTX
Gi0/7                        disabled     1            auto   auto 10/100/1000BaseTX
Gi0/8                        disabled     1            auto   auto 10/100/1000BaseTX
Gi0/9                        disabled     1            auto   auto 10/100/1000BaseTX
Gi0/10                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/11                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/12                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/13                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/14                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/15                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/16                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/17                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/18                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/19                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/20                       disabled     1            auto   auto 10/100/1000BaseTX
Gi0/21                       connected    trunk      a-full a-1000 10/100/1000BaseTX
Gi0/22                       disabled     1            auto   auto Not Present
Gi0/23                       disabled     1            auto   auto Not Present
Gi0/24                       disabled     1            auto   auto Not Present
```

## Commands
 `write` - Save running config to startup
 `show ip int brief` - show interface statuses



## Original Switchports
Port 5 - Main PC
Port 2 - Proxmox
Port 4 - NAS