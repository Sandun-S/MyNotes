
# Nexus Repository Manager Setup Guide (Docker)

**Date:** 2026-02-10  
**Environment:** Ubuntu VM (Multipass / WSL)  
**Tools:** Docker, Sonatype Nexus 3

---

## 1. Host Preparation (Docker Installation)

Before running Nexus, install the latest Docker Engine from the official Docker repository.

---

### A. Install Docker Engine

Run the following commands to add Docker’s GPG key and repository:

```bash
# 1. Add Docker's official GPG key
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 2. Add the Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. Install Docker packages
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
````

---

### B. Configure User Permissions

Allow running Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
```

**Action Required:**
Log out and log back in (or run `newgrp docker`).
Verify with:

```bash
docker ps
```

---

## 2. Initial Nexus Deployment

Launch Nexus once to initialize the data volume and generate the admin password.

---

### A. Create Storage Volume

Nexus requires persistent storage:

```bash
docker volume create nexus-data
```

---

### B. First Run (Web UI Only)

Expose only the management port (8081):

```bash
docker run -d -p 8081:8081 --name nexus -v nexus-data:/nexus-data sonatype/nexus3
```

Wait **2–3 minutes** for startup.

---

### C. Retrieve Initial Admin Password

```bash
docker exec nexus cat /nexus-data/admin.password
```

**Example Output:**

```
1ea0542c-4d51-4597-8c33-cdc4e12239a4
```

**Action:** Copy this password.

---

### D. Initial Login

* Open: `http://<VM_IP>:8081`
* Username: `admin`
* Password: value from Step **2C**
* Follow the wizard and **change the password**

  * Example: `1qaz2wsx`

---

## 3. Enable Docker Registry Support (Port 8082)

Nexus does not expose Docker registry ports by default.

---

### A. Configure via Web UI

1. Go to **Settings (⚙️) → Repositories**
2. Click **Create Repository**
3. Select **docker (hosted)**

**Configuration:**

* **Name:** `docker-private`
* **HTTP Connector:**

  * Enable **HTTP**
  * Port: `8082`

Save the repository.

---

### B. Redeploy Container with Port 8082

The existing container only exposes 8081.

```bash
# Stop & remove existing container
# (Safe: data persists in the volume)
docker rm -f nexus

# Start Nexus with both ports exposed
docker run -d \
  -p 8081:8081 \
  -p 8082:8082 \
  --name nexus \
  -v nexus-data:/nexus-data \
  sonatype/nexus3
```

---

## 4. Client Configuration (Insecure Registry)

Docker blocks HTTP registries by default.

---

### Configure Docker Daemon

**File:** `/etc/docker/daemon.json`

```json
{
  "insecure-registries": [
    "localhost:8082",
    "172.26.96.170:8082"
  ]
}
```

> Replace `172.26.96.170` with your actual VM IP if accessing remotely.

Restart Docker:

```bash
sudo systemctl restart docker
```

---

## 5. Usage Workflow (Push & Pull)

---

### A. Login

```bash
docker login localhost:8082
```

* **Username:** `admin`
* **Password:** (new password set in Step 2D)

> Use `localhost` when inside the VM to avoid network issues.

---

### B. Tagging & Pushing an Image

```bash
# 1. Pull a public image
docker pull alpine

# 2. Tag for private registry
docker tag alpine localhost:8082/my-alpine

# 3. Push to Nexus
docker push localhost:8082/my-alpine
```

---

### C. Verification (Pull Test)

```bash
# Remove local copy
docker rmi localhost:8082/my-alpine

# Pull from Nexus
docker pull localhost:8082/my-alpine
```

---

## 6. Troubleshooting

**Connection Refused**

* Check container status: `docker ps`
* Ensure port `8082` is listed under `PORTS`

**server gave HTTP response to HTTPS client**

* Missing `/etc/docker/daemon.json` configuration

**access denied / unauthorized**

* Wrong credentials
* Try:

  ```bash
  docker logout
  docker login localhost:8082
  ```