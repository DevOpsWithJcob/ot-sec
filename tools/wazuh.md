# Unlocking Threat Detection with Wazuh: A Beginner's Guide

As a network security expert, I've deployed Wazuh countless times to create unified security monitoring across diverse environments. Wazuh acts like a central command center for your security operations—it collects data from endpoints, correlates events in real-time, and helps you respond to threats swiftly. Unlike single-purpose tools, it combines SIEM (Security Information and Event Management) for log analysis with XDR (Extended Detection and Response) for proactive hunting, all open-source and free. Perfect for beginners building a security stack or pros scaling to thousands of agents.

## What is Wazuh?

Wazuh is a free, open-source cybersecurity platform that unifies XDR and SIEM capabilities to protect endpoints and cloud workloads across on-premises, virtualized, and containerized environments. It focuses on threat prevention, detection, and response by analyzing logs, monitoring file integrity, and detecting vulnerabilities, with no vendor lock-in or licensing costs.

Key highlights include:
- **Unified Protection**: Real-time correlation and remediation for endpoints, plus comprehensive event monitoring and alerting.
- **Endpoint and Cloud Security**: Deploys a lightweight agent to collect data from hosts, containers, and clouds like AWS or Azure, supporting over 15 million endpoints worldwide.
- **Flexibility and Integrations**: Customizable source code, API connections to tools like VirusTotal or TheHive, and a vibrant community for contributions.
- **Scalability**: Handles everything from small labs to enterprises with over 100,000 users, boasting 30 million downloads annually.
- **Managed Option**: Wazuh Cloud for hassle-free, scalable deployments with a free trial.

It's trusted for use cases like compliance auditing (e.g., PCI-DSS), vulnerability scanning, and incident response. Dive deeper at the [official Wazuh site](https://wazuh.com/) or [documentation](https://documentation.wazuh.com/current/index.html).

## Setting Up Wazuh in a Docker Lab Using Docker Compose

Docker lets you spin up Wazuh's full stack—manager, indexer, and dashboard—in isolated containers, ideal for testing without host pollution. This single-node setup uses official images for a quick lab.

### Prerequisites for Docker
- Docker Engine or Desktop (latest stable; AMD64 architecture).
- Docker Compose (latest stable; included in Desktop).
- Git for cloning the repo.
- Host specs: 4+ CPU cores, 8 GB+ RAM, 50 GB+ disk; Linux host needs `vm.max_map_count=262144` (run `sysctl -w vm.max_map_count=262144`).
- Optional: Add your user to the `docker` group for non-root runs.

### Docker Compose File Content
Clone the official repo to get the pre-built `docker-compose.yml`:
```
git clone https://github.com/wazuh/wazuh-docker.git -b v4.13.1
cd wazuh-docker/single-node/
```
The file in `single-node/` defines services like `wazuh.manager`, `wazuh.indexer`, and `wazuh.dashboard`, with volumes for data persistence and environment vars for configs (e.g., passwords in `.env`).

### Step-by-Step Instructions
1. **Clone the Repository**: Run the git command above and `cd` into `single-node/`.
2. **Generate Certificates**: Create self-signed certs for secure comms:
   ```
   docker compose -f generate-indexer-certs.yml run --rm generator
   ```
   Certs land in `config/wazuh_indexer_ssl_certs/`.
3. **Launch the Stack**: Start all components:
   ```
   docker compose up -d
   ```
   Wait ~1-2 minutes for initialization.
4. **Access the Dashboard**: Open `https://<YOUR_DOCKER_HOST_IP>` (e.g., localhost). Login: `admin` / `SecretPassword`. Ignore self-signed cert warnings.
5. **Verify**: Check container health: `docker compose ps`. View logs: `docker compose logs -f wazuh.dashboard`.
6. **Customize/Stop**: Edit `.env` or `docker-compose.yml`, then `docker compose down` to stop (add `-v` to nuke volumes).

**Important Notes on Configuration**: Use the `.env` file for passwords—change defaults post-setup. For production, swap self-signed certs with trusted ones. If behind a proxy, tweak `generate-indexer-certs.yml`. Scale by editing for multi-node later.

## Installing and Configuring Wazuh on a Virtual Machine with Ubuntu or Debian

For a native, persistent install, use the all-in-one script on a VM—it bundles the manager, indexer, and dashboard in minutes. This is great for learning the full platform without Docker overhead.

### Prerequisites for VM Setup
- Ubuntu 16.04+ (e.g., 22.04/24.04) or Debian equivalent; 64-bit (x86_64/ARM64).
- Root/sudo access; internet.
- Hardware: 4 vCPUs, 8 GB RAM, 50-200 GB disk (scales with agents).
- Update system: `sudo apt update && sudo apt upgrade -y`.

### Steps for VM Setup
1. **Download and Run the Installer**:
   ```
   curl -sO https://packages.wazuh.com/4.13/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
   ```
   This handles repo setup, package installs, cert generation, and service starts. Note the output: IP, admin password (e.g., `<ADMIN_PASSWORD>`).
2. **Access the Dashboard**: Browse to `https://<YOUR_VM_IP>`. Login: `admin` / `<ADMIN_PASSWORD>`. Accept cert exception.
3. **View Passwords (Optional)**: Extract full creds:
   ```
   sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
   ```
4. **Disable Auto-Updates (Recommended)**: Edit `/etc/apt/sources.list.d/wazuh.list` to comment out the repo line, preventing surprises.
5. **Verify Services**: Check status:
   ```
   sudo systemctl status wazuh-manager
   sudo systemctl status wazuh-indexer
   sudo systemctl status wazuh-dashboard
   ```
   All should be "active (running)".
6. **Add an Agent (Test)**: From another host, download/install the agent package and register it via the dashboard.

**Important Notes on Configuration**: The script sets defaults—tweak `/var/ossec/etc/ossec.conf` for custom rules. For multi-node, use distributed guides. Secure your VM (e.g., UFW: `sudo ufw allow 443`). Update manually: `sudo apt update && sudo apt upgrade`.

There you have it—Wazuh ready to monitor and alert! Start by adding a test agent and watching alerts roll in. Hit any roadblocks? The [docs](https://documentation.wazuh.com/current/) have your back. What's your setup goal?