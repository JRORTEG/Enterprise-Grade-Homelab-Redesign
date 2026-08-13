# WAN Interface Firewall Rules

Interface: WAN (external-facing). No internal subnet. Source is the public internet.

Default posture: **implicit block-all inbound** (pfSense default). Nothing reaches internal hosts from WAN unless explicitly permitted via a NAT port forward.

## How pfSense WAN rules work

pfSense evaluates firewall rules at the **ingress interface**: the interface where a packet enters the firewall. For traffic originating from the internet, that interface is WAN. This is a general pfSense behavior, not WAN-specific. See the guiding principle in `Firewall.md` and its fuller treatment in `VLAN_40.md`, which apply the same logic to inter-VLAN traffic.

Most internet-facing services in this homelab are reached via NAT (the pfSense WAN IP is publicly routable; DMZ hosts are RFC1918). The workflow is:

1. **Create a NAT Port Forward** in `Firewall > NAT > Port Forward`. Map WAN IP:port → DMZ host IP:port.
2. **Check "Add associated filter rule"** at creation time. pfSense auto-generates the corresponding WAN filter rule under `Firewall > Rules > WAN`. Do **not** also create the WAN filter rule manually. Duplicates cause rule-order issues and hard-to-diagnose behavior.

The filter rules in the "Auto-Generated WAN Filter Rules" section below are created by pfSense from the NAT rules; they are documented here for auditability, not for manual entry.

## NAT Port Forward Rules

Configure in `Firewall > NAT > Port Forward`. Each rule: Interface = WAN, Destination = WAN address.

| #  | WAN Port | Protocol | Redirect to (DMZ)           | Purpose                                                       |
|----|----------|----------|-----------------------------|---------------------------------------------------------------|
| W1 | 25565    | TCP      | Minecraft 10.0.40.10:25565  | Velocity proxy, player connections                           |
| W2 | 443      | TCP      | nginx 10.0.40.12:443        | HTTPS entry. nginx reverse proxies all DMZ services          |
| W3 | 51820    | UDP      | PiVPN 10.0.40.13:51820      | WireGuard VPN endpoint (see PiVPN scope note in `VLAN_40.md`) |
| W4 | any      | any      | Torrenting 10.0.40.14       | P2P incoming peer connections ⚠ broad. See risk note         |

> **nginx port 80 (deferred, see open items):** Rule W5 (TCP 80 → nginx 10.0.40.12:80) is not added until the ACME renewal method is confirmed. HTTP-01 challenge requires inbound port 80; DNS-01 and TLS-ALPN-01 do not.

## Auto-Generated WAN Filter Rules

These rules appear in `Firewall > Rules > WAN` automatically when NAT rules W1–W4 are created with "Add associated filter rule" checked. Listed here for audit purposes only. Do not create them manually.

| #  | Source    | Destination           | Proto/Port | Generated from |
|----|-----------|-----------------------|------------|----------------|
| F1 | WAN (any) | Minecraft 10.0.40.10  | TCP 25565  | NAT rule W1    |
| F2 | WAN (any) | nginx 10.0.40.12      | TCP 443    | NAT rule W2    |
| F3 | WAN (any) | PiVPN 10.0.40.13      | UDP 51820  | NAT rule W3    |
| F4 | WAN (any) | Torrenting 10.0.40.14 | any/any    | NAT rule W4 ⚠  |

## Anti-Spoofing and WAN Interface Defaults

These are interface-level settings, not rule-table entries. Verify both are enabled under `System > Interfaces > WAN`:

- **Block private networks:** Drops RFC1918 addresses (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) arriving on WAN. Prevents spoofed internal-address traffic from the internet.
- **Block bogon networks:** Drops unallocated/reserved IP ranges arriving on WAN. Both are enabled by default in pfSense; confirm they have not been disabled.

## WAN Management Lockout

pfSense's built-in anti-lockout rule only protects the LAN interface. WAN management access is blocked by the implicit deny-all, but confirm no legacy rules exist:

- No WAN filter rule should allow TCP 443 or TCP 22 destined to the pfSense WAN IP itself.
- If remote management access is ever needed, reach the Mgmt VLAN via PiVPN/WireGuard (W3) first, then access pfSense at 10.0.10.1. Never expose the management UI directly to WAN.

## Enterprise Hardening: Future Candidates

Not yet implemented; logged here for future planning:

- **GeoIP blocking (pfBlockerNG):** Restrict Minecraft (W1) and nginx (W2) to expected geographic regions. Reduces scanning and brute-force attack surface without affecting legitimate users.
- **IDS/IPS (Snort or Suricata):** Inline WAN traffic inspection. Particularly relevant given the broad Torrenting rule (W4/F4).
- **Rate limiting on WAN rules:** Useful for Minecraft (W1) against connection floods and nginx (W2) against volumetric attacks.

## Risk note: Torrenting any/any (W4, F4)

No port/protocol restriction in either direction, required for P2P trackers and peers whose ports are unpredictable. This is the single largest open attack surface on the WAN interface. Not resolved in this pass; revisit once the torrent client's configured port range is known. Most clients support pinning to a specific range, which would convert W4/F4 from any/any to a scoped TCP/UDP rule.

## Open items

- **nginx port 80 inbound (W5, deferred):** Confirm ACME renewal method in use. HTTP-01 requires inbound TCP 80 → nginx 10.0.40.12:80; DNS-01 and TLS-ALPN-01 do not. Add W5 and corresponding F5 only if HTTP-01 is confirmed.
- **Torrenting port range:** Replace any/any (W4/F4) with a specific port range once the client's configuration is known.
- **GeoIP and IDS/IPS:** See enterprise hardening section above, tracked as future candidates.

## Cross-references

- `VLAN_40.md`: outbound (DMZ-initiated) rules for all DMZ hosts; sole DMZ→internal exception is rule 12 (New Jellyfin → TrueNAS SMB)
- `VLAN_60.md` rule 15: Trusted Clients → Old Jellyfin direct access (lives on VLAN 60 tab, not WAN)
- `VLAN_30.md` rules 11–15: Apps → DMZ hosts Grafana scrape (lives on VLAN 30 tab, not WAN)
