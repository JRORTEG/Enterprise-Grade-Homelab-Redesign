# Complete Guide: Dual Minecraft Servers behind a Velocity Proxy
*A step-by-step guide to deploying a high-performance Vanilla and Modded server on a single Docker host.*

---

## Prerequisites
* A host machine (e.g., Proxmox LXC) with Docker installed.
* At least 8GB of RAM allocated to the host.

---

## Step 1: Create the Shared Docker Network
First, establish an external Docker network that all containers will share to communicate securely without exposing backend ports.
```bash
docker network create mc-network
```

---

## Step 2: Set Up Your Directory Structure
Create separate folders for each component of the project to ensure isolated maintenance and prevent conflicts.
```bash
mkdir -p ~/minecraft/vanilla
mkdir -p ~/minecraft/modded
mkdir -p ~/minecraft/velocity
```

---

## Step 3: Configure the Vanilla Server
Navigate to the vanilla directory (`cd ~/minecraft/vanilla`) and create a `docker-compose.yml` file:
```yaml
services:
  # Server 1: Vanilla Java Edition
  mc-vanilla:
    image: itzg/minecraft-server
    container_name: mc-vanilla
    environment:
      EULA: "TRUE"
      TYPE: "PAPER"
      VERSION: "LATEST"
      MEMORY: "2G"
      ONLINE_MODE: "FALSE" # Velocity handles the encryption/auth instead
    volumes:
      - ./vanilla-data:/data
    restart: unless-stopped

networks:
  default:
    name: mc-network
    external: true
```

---

## Step 4: Configure the Modded Server
Navigate to the modded directory (`cd ~/minecraft/modded`) and create a `docker-compose.yml` file:
```yaml
services:
  # Server 2: Modded Server (Using Fabric for mod loader support)
  mc-modded:
    image: itzg/minecraft-server
    container_name: mc-modded
    environment:
      EULA: "TRUE"
      TYPE: "FABRIC"
      VERSION: "1.20.1"
      MEMORY: "3G"
      ONLINE_MODE: "FALSE" # Velocity handles the encryption/auth instead
    volumes:
      - ./modded-data:/data
    restart: unless-stopped

networks:
  default:
    name: mc-network
    external: true
```

---

## Step 5: Configure the Velocity Proxy
Navigate to the velocity directory (`cd ~/minecraft/velocity`) and create a `docker-compose.yml` file:
```yaml
services:
  velocity-proxy:
    image: itzg/mc-proxy
    container_name: velocity-proxy
    ports:
      - "25565:25577" # Binds standard MC port 25565 on the host to Velocity's default 25577 port
    environment:
      TYPE: "VELOCITY"
      MEMORY: "512m"
    volumes:
      - ./data:/server
    restart: unless-stopped

networks:
  default:
    name: mc-network
    external: true
```

---

## Step 6: Initial Boot & Proxy Configuration
1. Start the Velocity proxy to generate its configuration files:
   ```bash
   cd ~/minecraft/velocity
   docker compose up -d
   ```
2. Open the newly generated configuration file (`~/minecraft/velocity/data/velocity.toml`).
3. Make the following modifications:
   * Set `player-info-forwarding-mode = "modern"`
   * Map your servers in the `[servers]` section and uncomment `try`:
     ```toml
     [servers]
     vanilla = "mc-vanilla:25565"
     modded = "mc-modded:25565"
     try = ["vanilla"]
     ```
   * Comment out the sample servers in the `[forced-hosts]` section by adding a `#` in front of them:
     ```toml
     [forced-hosts]
     #"lobby.example.com" = ["lobby"]
     #"factions.example.com" = ["factions"]
     #"minigames.example.com" = ["minigames"]
     ```
4. Restart Velocity to apply the changes: 
   ```bash
   docker compose restart
   ```
5. Retrieve your secret key. Copy the output of this command to your clipboard:
   ```bash
   cat ~/minecraft/velocity/data/forwarding.secret
   ```

---

## Step 7: Secure the Backend Servers
1. Boot both backend servers to generate their files, wait a minute for them to fully initialize, and then stop them:
   ```bash
   cd ~/minecraft/vanilla && docker compose up -d
   cd ~/minecraft/modded && docker compose up -d
   
   # Wait ~60 seconds for initial file generation to complete, then:
   cd ~/minecraft/vanilla && docker compose down
   cd ~/minecraft/modded && docker compose down
   ```
2. Open the Paper global config file for the **Vanilla** server (`~/minecraft/vanilla/vanilla-data/config/paper-global.yml`).
3. Find the `proxies:` section, enable velocity, and paste your secret key:
   ```yaml
   proxies:
     velocity:
       enabled: true
       online-mode: true
       secret: "PASTE_YOUR_FORWARDING_SECRET_HERE"
   ```
4. The Modded server runs Fabric, which has no `paper-global.yml`. Instead, configure equivalent Velocity modern-forwarding support through the Fabric mod loader setup, using the same forwarding secret (see `Future-Planning/Open Items.md` #16 — exact mod/config not yet documented).

---

## Step 8: Final Launch
Start both backend servers back up:
```bash
cd ~/minecraft/vanilla && docker compose up -d
cd ~/minecraft/modded && docker compose up -d
```

**Success!** Your network is now live. Players can connect to your host's IP address on the default port `25565`. They will automatically join the Vanilla server and can type `/server modded` in-game to switch dynamically.
