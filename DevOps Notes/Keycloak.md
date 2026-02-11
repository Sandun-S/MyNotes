Below is a **complete, production-oriented Keycloak setup note**, rewritten clearly, structured properly, and with missing operational details added.

This assumes:

- Windows Host
- Hyper-V backend
- Ubuntu VM via Multipass
- Docker-based Keycloak deployment

---

# Keycloak Deployment on Multipass (Docker-Based)

**Host:** Windows 10/11 Pro  
**Backend:** Hyper-V (Multipass)  
**Guest OS:** Ubuntu (Cloud Image)  
**Deployment Mode:** Docker (Single Node)  
**Purpose:** Dev / Lab / Internal Auth Server  

---

# 1. Create the VM

Create a dedicated VM for Keycloak:

```bash
multipass launch \
  --name keycloak-vm \
  --cpus 2 \
  --memory 4G \
  --disk 10G
````

Verify:

```bash
multipass list
```

Shell into the VM:

```bash
multipass shell keycloak-vm
```

---

# 2. Install Docker Engine (Inside VM)

## 2.1 Install Required Dependencies

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

---

## 2.2 Add Docker Official GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

---

## 2.3 Add Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

---

## 2.4 Install Docker Packages

```bash
sudo apt-get update
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Verify installation:

```bash
docker --version
docker compose version
```

---

# 3. Allow Docker Without sudo

Add `ubuntu` user to docker group:

```bash
sudo usermod -aG docker ubuntu
newgrp docker
```

Verify:

```bash
docker ps
```

---

# 4. Create Persistent Volume

Keycloak stores realm data inside `/opt/keycloak/data`.

Create named volume:

```bash
docker volume create keycloak_data
```

Verify:

```bash
docker volume ls
```

---

# 5. Deployment Options

---

# OPTION 1 — Direct `docker run` (Quick Dev Mode)

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=admin \
  -v keycloak_data:/opt/keycloak/data \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

---

### Important Notes

- `start-dev` → Uses embedded H2 database
- Not production-safe
- Admin credentials are hardcoded (change later)
- HTTP only (no TLS)

---

# OPTION 2 — Docker Compose (Recommended Even for Dev)

Create file:

```bash
nano docker-compose.yml
```

Content:

```yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    container_name: keycloak
    ports:
      - "8080:8080"
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
    volumes:
      - keycloak_data:/opt/keycloak/data
    command: start-dev
    restart: unless-stopped

volumes:
  keycloak_data:
```

Start service:

```bash
docker compose up -d
```

---

# 6. Verify Container

Check running containers:

```bash
docker ps
```

Follow logs:

```bash
docker logs -f keycloak
```

Wait until you see:

```
Keycloak <version> started in ...
```

---

# 7. Access Keycloak from Host

Open new terminal on Windows host:

```bash
multipass info keycloak-vm
```

Look for:

```
IPv4: 172.xx.xx.xx
```

Open browser:

```
http://<VM_IP>:8080
```

OR use DNS (recommended):

```
http://keycloak-vm.mshome.net:8080
```

---

# 8. First Login

1. Click **Administration Console**
2. Login:
    - Username: `admin`
    - Password: `admin`
3. Immediately change password

---

# 9. Production Considerations (IMPORTANT)

The above setup is DEV ONLY.

For production, you MUST:

### 9.1 Use External Database (PostgreSQL Recommended)

Example Compose Snippet:

```yaml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: strongpassword
    volumes:
      - pg_data:/var/lib/postgresql/data

  keycloak:
    image: quay.io/keycloak/keycloak:latest
    command: start
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: strongpassword
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: strongpassword
```

---

### 9.2 Enable HTTPS (Reverse Proxy Recommended)

Recommended approach:

- Run NGINX or Traefik
- Terminate TLS
- Forward traffic to Keycloak on 8080
---

### 9.3 Set Proper Hostname

Keycloak requires hostname config in production:

```
KC_HOSTNAME=auth.example.com
```

Without this:

- Redirect issues
- Token issuer mismatch

---

# 10. Common Problems

---

## Problem: Page loads but login fails

Cause:
- Wrong hostname
- Mixed HTTP/HTTPS

Fix:
- Set `KC_HOSTNAME`
- Use consistent protocol

---

## Problem: Container restarts repeatedly

Check logs:

```bash
docker logs keycloak
```

Common causes:

- DB not reachable
- Port already in use

---

## Problem: Cannot Access from Windows

Check:

```bash
multipass list
```

If IP missing:

```powershell
Restart-Service multipassd
```

---

# 11. Recommended Resource Sizing

|Use Case|CPU|RAM|Disk|
|---|---|---|---|
|Dev Lab|2|4GB|10GB|
|Small Internal Auth|2–4|8GB|20GB|
|Production|4+|8–16GB|50GB+|

---

# 12. Operational Commands Quick Reference

```bash
# Stop Keycloak
docker stop keycloak

# Start Keycloak
docker start keycloak

# Restart
docker restart keycloak

# Remove container (data persists)
docker rm -f keycloak

# Remove volume (DESTROYS DATA)
docker volume rm keycloak_data
```

---

# 13. Summary Architecture (Dev Mode)

Windows Host  
→ Hyper-V  
→ Multipass Ubuntu VM  
→ Docker  
→ Keycloak Container  
→ H2 DB (Embedded)

---

# 14. Final Recommendations

For serious environments:

- Do NOT use `start-dev`
- Do NOT use embedded H2 DB
- Always use PostgreSQL
- Always use HTTPS
- Always set hostname explicitly
- Backup volume regularly