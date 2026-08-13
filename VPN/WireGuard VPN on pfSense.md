# Setup WireGuard VPN on pfSense

Steps for setting up a WireGuard remote-access tunnel on pfSense: server config on the firewall, then matching config on the client device.

## 1. Install the WireGuard Package
* Navigate to System → Package Manager → Available Packages.
* Search for `wireguard`, click install, and wait for it to finish.

## 2. Create the Tunnel (Server Configuration)
* Go to VPN → WireGuard → Tunnels and click + Add Tunnel.
* Check Enable and give it a description (e.g., "WG_RemoteAccess").
* In the Interface Keys section, click Generate to create the pfSense server's public and private key pair.
* Copy the Public Key. It's needed for the client device later.
* Under Interface Addresses, assign a new, unused private subnet for the VPN (e.g., `10.100.0.1/24`). This is the gateway IP for VPN clients and has to be set for routing to work.
* Save the tunnel.
* Go to the Settings tab in the WireGuard menu, check Enable WireGuard, and Save.

## 3. Assign and Enable the Interface
* Navigate to Interfaces → Assignments.
* Next to "Available network ports," select the new tunnel (e.g., `tun_wg0`) from the dropdown and click + Add.
* Click the newly created interface name (usually OPT1 by default).
* Check Enable Interface, rename it to something recognizable (e.g., WG_VPN).
* Leave the IPv4 and IPv6 configuration types as "None" since the WireGuard tunnel settings already handle the IP.
* Click Save, then Apply Changes.

## 4. Configure Firewall Rules
Two rules are needed for the VPN to work:
* **Allow inbound connection:** Firewall → Rules → WAN. Add a rule to pass UDP traffic where the Destination is "WAN Address" and the Destination Port matches the tunnel's port (default 51820).
* **Allow internal routing:** Firewall → Rules → WG_VPN (the interface named above). Add a rule to pass any protocol from the WireGuard subnet to LAN net (or to Any for full internet access routed through the VPN).
* Save and Apply Changes.

## 5. Generate Keys on the Client Device
Before adding the client into pfSense, the client device needs its own configuration.
* Open the WireGuard app on the client device and add a new empty tunnel.
* The app generates a public/private key pair automatically.
* Copy the client's Public Key.
* In the client's interface configuration, set its address to an IP within the tunnel's subnet (e.g., `10.100.0.2/32`).
* Under the Peer section of the client app, input the pfSense server's Public Key (from Step 2).
* Set the endpoint to the pfSense WAN IP and port (e.g., `203.0.113.1:51820`), and set Allowed IPs to `0.0.0.0/0` for full tunneling or the LAN subnet for split tunneling.

## 6. Add the Client to pfSense as a Peer
* Back in pfSense, go to VPN → WireGuard → Peers and click + Add Peer.
* Select the tunnel (`tun_wg0`) from the dropdown.
* Uncheck Dynamic Endpoint, since a mobile client or laptop's IP changes while traveling.
* Paste the client's Public Key (from Step 5) into the Peer Keys section.
* Under Allowed IPs, enter the exact IP assigned to the client (e.g., `10.100.0.2`).
* Use a `/32` subnet mask here so pfSense knows precisely which device gets this address.
* Save and Apply Changes.

Toggle the WireGuard connection on the client device once the config is applied. With internet access on the client and the correct pfSense WAN IP, the handshake completes and the tunnel is up.
