# Unlocking Network Insights with Zeek: A Beginner's Guide

As a network security expert, I've relied on Zeek for years to dissect traffic patterns and uncover hidden threats without disrupting operations. Think of Zeek as your network's storyteller—it doesn't just flag alerts like a traditional intrusion detection system; instead, it generates detailed logs of what's happening across your connections, protocols, and applications. This makes it perfect for security analysis, compliance checks, and troubleshooting. Whether you're monitoring a home lab or an enterprise setup, Zeek's open-source nature keeps it flexible and cost-free.

## What is Zeek?

Zeek, formerly known as Bro, is an open-source framework for network analysis and security monitoring. Developed originally in the 1990s by Vern Paxson and now maintained by the Zeek Project with support from Corelight, it excels at passively observing network traffic to produce rich, structured logs rather than blocking threats outright. This "analyze first" approach helps you understand behaviors, like unusual file transfers or protocol anomalies, for deeper investigations.

Key highlights include:
- **Rich Logging**: Over 70 default log types (e.g., connection details, HTTP requests, DNS queries) tracking more than 3,000 events, with options for custom outputs like JSON for easy integration with tools like ELK Stack.
- **Protocol Awareness**: Decodes complex protocols (e.g., TLS, SSH) to reveal encrypted content metadata without decryption.
- **Scalability**: Handles high-speed networks via clustering, with over 10,000 global deployments.
- **Extensibility**: Write scripts in Zeek's domain-specific language to tailor detection, plus a package manager (zkg) for community plugins.

It's ideal for security teams wanting visibility into "what's normal" versus suspicious activity, feeding into SIEMs for automated alerts. Unlike firewalls, Zeek focuses on forensics and intelligence, making it a staple for threat hunting. For more, visit the [official Zeek site](https://zeek.org/).

## Setting Up Zeek in a Docker Lab Using Docker Compose

Docker simplifies Zeek deployment by packaging everything into a container, great for isolated labs without messing up your host system. We'll use Docker Compose to orchestrate a single-node setup, monitoring traffic on your host's interface.

### Prerequisites for Docker
- Docker Engine installed (version 20+ recommended; get it from [docker.com](https://www.docker.com/)).
- Docker Compose (v2+; often bundled with Docker now).
- Host networking enabled (for real traffic capture) and sufficient privileges (e.g., run as root or add user to docker group).
- Basic terminal access; tested on Linux/Mac hosts.

### Docker Compose File Content
Create a file named `docker-compose.yml` in a new directory (e.g., `mkdir zeek-lab && cd zeek-lab`). Paste this simple config for a standalone Zeek instance:

```yaml
version: '3.8'
services:
  zeek:
    image: zeek/zeek:latest  # Or 'zeek/zeek:lts' for stable LTS
    container_name: zeek-monitor
    network_mode: host  # Allows capturing host traffic
    cap_add:
      - NET_ADMIN
      - NET_RAW  # Needed for packet capture
    volumes:
      - ./logs:/opt/zeek/logs:delegated  # Persist logs locally
      - ./etc:/opt/zeek/etc:delegated  # Optional: Mount configs
    environment:
      - ZEEK_INTERFACE=eth0  # Replace with your interface (e.g., enp0s3)
    command: >
      bash -c "
        zeekctl install && 
        zeekctl deploy &&
        zeekctl start
      "
    restart: unless-stopped
```

This pulls the official image, mounts logs for easy access, and auto-deploys Zeek.

### Step-by-Step Instructions
1. **Prepare Your Directory**: Create the folder and `docker-compose.yml` as above. Make a `logs` subfolder: `mkdir logs`.
2. **Find Your Network Interface**: Run `ip link show` to identify it (e.g., `eth0` or `en0` on Mac). Update the `ZEEK_INTERFACE` in the YAML.
3. **Pull and Start**: Run `docker compose up -d` to download the image and launch in detached mode.
4. **Verify Setup**: Check container status with `docker compose ps`. View logs: `tail -f logs/current/conn.log` (generate traffic by browsing a site).
5. **Customize (Optional)**: Edit configs in `./etc` (copy defaults from container if needed: `docker cp zeek-monitor:/opt/zeek/etc ./etc`), then restart: `docker compose restart`.
6. **Stop and Clean**: `docker compose down` to halt; add `-v` to remove volumes.

**Important Notes on Configuration**: Start with default scripts for broad logging. Tune `networks.cfg` in `./etc` for your subnets to reduce noise. For production, add GeoIP plugins via zkg inside the container. Always test in passive mode first.

## Installing and Configuring Zeek on a Virtual Machine with Ubuntu or Debian

For a more persistent setup, install natively on a VM (e.g., via VirtualBox or Proxmox). This gives full system integration but requires more setup. Instructions are similar for Ubuntu 24.04 and Debian 12; adjust the repo URL accordingly.

### Prerequisites for VM Setup
- A VM with Ubuntu 24.04 LTS or Debian 12, at least 2GB RAM, 20GB disk, and 2 vCPUs.
- Root or sudo access; internet connectivity for packages.
- Update your system first: `sudo apt update && sudo apt upgrade -y`.

### Steps for VM Setup
1. **Add Zeek Repository**:
   - Download GPG key: `curl -fsSL https://download.opensuse.org/repositories/security:zeek/xUbuntu_24.04/Release.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/security_zeek.gpg > /dev/null` (For Debian 12, replace `xUbuntu_24.04` with `Debian_12`).
   - Add repo: `echo 'deb http://download.opensuse.org/repositories/security:/zeek/xUbuntu_24.04/ /' | sudo tee /etc/apt/sources.list.d/security:zeek.list` (Adjust for Debian as above).
   - Update: `sudo apt update`.

2. **Install Zeek**:
   - Install: `sudo apt install zeek -y` (or `zeek-lts` for stable).
   - Add to PATH: `echo 'export PATH=$PATH:/opt/zeek/bin' >> ~/.bashrc && source ~/.bashrc`.
   - Verify: `zeek --version` (should show v7.x or similar).

3. **Configure Networks**:
   - Edit `/opt/zeek/etc/networks.cfg`: Add your local subnets, e.g.:
     ```
     10.0.0.0/8     Private IP space
     172.16.0.0/12  Private IP space
     192.168.0.0/16 Private IP space
     ```
   - Save and exit.

4. **Set Up Node Configuration** (for cluster mode; simpler than standalone):
   - Edit `/opt/zeek/etc/node.cfg`: Comment out standalone lines with `#` and add (replace `YOUR_VM_IP` with your VM's IP from `ip a`):
     ```
     [zeek-logger]
     type=logger
     host=YOUR_VM_IP

     [zeek-manager]
     type=manager
     host=YOUR_VM_IP

     [zeek-proxy]
     type=proxy
     host=YOUR_VM_IP

     [zeek-worker]
     type=worker
     host=YOUR_VM_IP
     interface=eth0  # Your interface

     [zeek-worker-lo]
     type=worker
     host=localhost
     interface=lo
     ```
   - Save.

5. **Deploy and Start**:
   - Run `sudo zeekctl install`.
   - Validate: `sudo zeekctl check` (all "ok").
   - Deploy: `sudo zeekctl deploy`.
   - Start: `sudo zeekctl start`.
   - Status: `sudo zeekctl status` (should show running).

6. **Verify and Explore Logs**:
   - Check logs: `ls /opt/zeek/logs/current/` (e.g., `conn.log` for connections).
   - Tail a log: `tail -f /opt/zeek/logs/current/conn.log`.
   - Stop: `sudo zeekctl stop`.

**Important Notes on Configuration**: Use cluster mode for scalability, but standalone works for small VMs—uncomment the `[zeek]` section in `node.cfg` if preferred. Regularly update Zeek: `sudo apt update && sudo apt upgrade zeek`. For JSON logs, add `@load policy/misc/json-logs` to `local.zeek`. Secure your VM with a firewall (e.g., ufw allow from trusted IPs).

There you go—Zeek up and running in minutes! Start by generating some traffic and reviewing `conn.log` to see the magic. If you hit snags, the [Zeek docs](https://docs.zeek.org/) are gold. What's your first analysis goal?