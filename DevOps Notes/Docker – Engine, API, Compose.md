# 1) Docker Installation (Ubuntu)

### Manual Install (APT – Recommended)

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker repo:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

Install Engine + Plugins:

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add user to docker group:

```bash
sudo usermod -aG docker $USER
exit
```

---

### Script Version (Docker-install.sh)

```bash
#!/bin/bash
# run as root

sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

# 2) Docker Remote API (Port 2375 – No TLS)

⚠ Exposes Docker daemon over TCP. Use only in trusted networks or behind firewall/VPN.

## Enable Docker TCP API (Manager Node)

Edit daemon config:

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}
```

Create systemd override:

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/override.conf
```

Paste:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```

Apply changes:

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
systemctl status docker
```

---

## Remote Control from Windows PowerShell

Point terminal to Swarm VIP:

```powershell
$env:DOCKER_HOST="tcp://172.28.192.100:2375"
docker node ls
```

Alternate VIP:

```powershell
$env:DOCKER_HOST="tcp://172.26.24.100:2375"
docker node ls
docker service ls
```

---

## Docker API – curl Playbook

Health check:

```bash
curl.exe http://<VIP>:2375/_ping
```

List nodes (JSON):

```bash
curl.exe -s http://<VIP>:2375/nodes | jq
```

Swarm info:

```bash
curl.exe -s http://<VIP>:2375/swarm | jq
```

Filtered task query:

```bash
curl.exe -G "http://<VIP>:2375/tasks" --data-urlencode 'filters={\"service\":[\"web-test\"]}' | jq
```

---

# 3) Docker Compose (Non-Swarm)

## Core Commands

Start full stack (background):

```bash
docker compose up -d
```

Check running services:

```bash
docker compose ps
```

Live logs (all services):

```bash
docker compose logs -f
```

Stop & remove stack (keep volumes):

```bash
docker compose down
```

Restart single service:

```bash
docker compose restart <name>
docker compose restart nodered
```

---

# 4) Container-Level Operations

List containers:

```bash
docker ps
docker ps -a | grep <name>
```

Stop container:

```bash
docker stop <id>
```

Container logs:

```bash
docker logs -f --tail 100 <container_id>
```

Shell into container:

```bash
docker exec -it qube-docs-dev bash
docker exec -u root -it qube-docs-dev bash
```

---

# 5) Images & Volumes

List volumes:

```bash
docker volume ls
```

Inspect volume:

```bash
docker volume inspect nexus-data
```

Remove unused images:

```bash
docker image prune
```

System cleanup:

```bash
docker system prune
```

---

# 6) Quick Operational Set (IIoT Context)

Deploy Compose stack:

```bash
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Investigate issue:

```bash
docker compose logs -f
docker ps -a | grep <name>
docker logs -f --tail 100 <container_id>
```

Disk cleanup on edge device (Pi / VM):

```bash
docker image prune
docker system prune
```

---

This now cleanly separates:

- Engine installation
    
- Remote API configuration
    
- Compose usage (non-Swarm)
    
- Container-level debugging
    
- Storage + image maintenance
    

No duplicated Swarm-only commands included.

If you want, I can next produce a **single consolidated “Docker + Swarm + API master reference”** structured for production ops.