
## Project Goal
The objective of this project was to deploy a dual-server Minecraft environment (one Vanilla, one Modded) running simultaneously on a single Proxmox LXC container with 8GB of allocated memory. The architecture required a secure, centralized entry point using a reverse proxy to route player traffic seamlessly to either server.

## Key Architectural Decisions
* **Memory Allocation:** To prevent out-of-memory (OOM) crashes, the 8GB of LXC memory was distributed strategically: 3GB for the Modded server, 2GB for the Vanilla server, and 3GB reserved for host/LXC overhead.
* **Decoupled Architecture:** Instead of using a single `docker-compose.yml`, the servers and proxy were split into separate directories. This prevents restarts or mod troubleshooting on one server from impacting players on the other.
* **Shared Networking:** An external Docker network (`mc-network`) was established, allowing the isolated containers to communicate securely by their internal hostnames without exposing their individual backend ports to the outside network.
* **Proxy Selection:** Velocity (`itzg/mc-proxy`) was chosen as the high-performance reverse proxy to handle authentication and traffic routing.
* **Server Software:** The Vanilla backend runs Paper (`TYPE: "PAPER"`), providing full vanilla compatibility plus the backend support Velocity's secure "modern" player forwarding requires. The Modded backend runs Fabric (`TYPE: "FABRIC"`) to support the Fabric mod loader ecosystem.

## Implementation Steps
1. **Initial Deployment Strategy:** Created the foundational `docker-compose.yml` configurations utilizing the `itzg/minecraft-server` image.
2. **Network Resolution:** Created the shared Docker network and updated the compose syntax to correctly utilize `external: true` to resolve Docker deprecation warnings.
3. **Velocity Configuration:** Deployed the Velocity proxy container. Resolved a volume mapping mismatch by directing the host directory to `/server` internally, allowing configuration files (`velocity.toml`, `forwarding.secret`) to generate and sync properly.
4. **Proxy Hardening:** Updated Velocity's `player-info-forwarding-mode` to `"modern"` and cleared out default dummy configurations in the `[forced-hosts]` block to fix a proxy crash loop.
5. **Backend Security Setup:** Configured both backend game servers to run in offline mode (`ONLINE_MODE: "FALSE"`), ensuring players can only join via the proxy.
6. **Secret Handshake Integration:** Copied Velocity's `forwarding.secret` string and injected it into the Vanilla server's `paper-global.yml` file (native Paper support). Configured equivalent Velocity modern-forwarding support on the Modded server through its Fabric mod loader setup, allowing secure transmission of player profiles on both backends.
7. **World Generation Fix:** Resolved a chunk loading error (`4903 > 4790`) on the vanilla server by wiping the originally generated world folders, allowing Paper to generate a clean, version-matching map.
8. **Final Validation:** Successfully connected to the Vanilla server via the proxy port (`25565`) and verified the usage of the `/server <name>` in-game command to dynamically switch between instances.
