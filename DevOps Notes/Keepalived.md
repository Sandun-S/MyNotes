
# Keepalived High Availability (Floating IP) – Operational Notes

**Date:** Monday, February 2, 2026  
**Purpose:**  
Provide High Availability (HA) using a Floating / Virtual IP (VIP) via **Keepalived (VRRP)**.  
Used together with **Docker Swarm self-healing logic**.

---

## 1. What Keepalived Does (Short)

- Uses **VRRP (Virtual Router Redundancy Protocol)**
- One node is **MASTER**, others are **BACKUP**
- MASTER owns the **Virtual IP (VIP)**
- If MASTER fails, a BACKUP takes over the VIP automatically
- Optional **notify scripts** allow automation on state change

---

## 2. Installation

```bash
sudo apt update
sudo apt install -y keepalived
````

Verify installation:

```bash
keepalived --version
```

---

## 3. Service Management

```bash
# Check status
systemctl status keepalived

# Start service
sudo systemctl start keepalived

# Stop service
sudo systemctl stop keepalived

# Restart service
sudo systemctl restart keepalived

# Enable at boot
sudo systemctl enable keepalived
```

---

## 4. Main Configuration File

**File:** `/etc/keepalived/keepalived.conf`

Edit:

```bash
sudo nano /etc/keepalived/keepalived.conf
```

---

## 5. Basic VRRP Example (Minimal)

```conf
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 100
    advert_int 1

    virtual_ipaddress {
        172.26.100.100/20
    }
}
```

### Key Parameters Explained

|Parameter|Meaning|
|---|---|
|`state`|Initial role (`MASTER` or `BACKUP`)|
|`interface`|Network interface used for VRRP heartbeats|
|`virtual_router_id`|VRRP group ID (must match on all nodes)|
|`priority`|Election weight (higher = MASTER)|
|`advert_int`|Heartbeat interval in seconds|
|`virtual_ipaddress`|Floating IP with subnet|

---

## 6. Full Production Configuration (Recommended)

```conf
! Configuration File for keepalived

global_defs {
   router_id ri-1161                 # Unique ID per node (must differ per machine)

   vrrp_skip_check_adv_addr          # Skip sender IP validation for VRRP packets
                                     # Improves stability in complex networks

   vrrp_garp_interval 0.1            # Gratuitous ARP interval (IPv4)
   vrrp_gna_interval 0.1             # Gratuitous Neighbor Advertisement (IPv6)
}

vrrp_instance qube-1161 {
    state MASTER                     # MASTER on primary, BACKUP on others
    interface wlan0                  # Interface carrying the VIP
    virtual_router_id 141            # VRRP group ID (same on all nodes)
    priority 100                     # MASTER: 100, BACKUP: 90 / 80
    advert_int 1                     # Heartbeat every 1 second

    notify "/etc/keepalived/notify.sh"

    authentication {
        auth_type PASS
        auth_pass 1161               # Shared password across nodes
    }

    virtual_ipaddress {
        192.168.37.5/22              # Floating IP + subnet
    }
}
```

---

## 7. MASTER / BACKUP Priority Strategy

|Node Role|Priority|
|---|---|
|Primary MASTER|100|
|Backup Node 1|90|
|Backup Node 2|80|

> Highest priority **always becomes MASTER** if healthy.

---

## 8. Notify Script (Advanced – Self-Healing)

**Purpose:**  
Triggered automatically when a node changes VRRP state (MASTER / BACKUP / FAULT).

**Use Case:**

- If a node becomes MASTER
    
- And Docker Swarm has **no leader**
    
- Re-initialize the Swarm automatically
    

---

### 8.1 Notify Script Location

```bash
sudo nano /etc/keepalived/notify.sh
```

---

### 8.2 Notify Script Content

```bash
#!/bin/bash
#
# /etc/keepalived/notify.sh

NODE="$2"
STATE="$3"

# Extract interface name from keepalived config
IFACE=$(grep interface /etc/keepalived/keepalived.conf | awk '{print $2}')

# Get local IP of that interface
MYIP=$(ifconfig "$IFACE" | grep 'inet ' | awk '{print $2}')

LOGFILE="/var/log/qube/kpld.log"

echo "$(date) - EVENT: $1 NODE: $NODE STATE: $STATE" >> "$LOGFILE"

if [ "$STATE" == "MASTER" ]; then

    # Check if a Swarm Leader exists
    docker node ls | grep Leader > /dev/null

    if [ "$?" != "0" ]; then
        echo "$(date) - No Swarm Leader found. Reinitializing Swarm on $MYIP" >> "$LOGFILE"

        while true; do
            docker swarm init \
                --force-new-cluster \
                --advertise-addr "$MYIP"

            if [ "$?" == "0" ]; then
                echo "$(date) - Swarm successfully reinitialized" >> "$LOGFILE"
                break
            fi
        done
    fi
fi
```

---

### 8.3 Permissions

```bash
sudo chmod +x /etc/keepalived/notify.sh
```

Create log directory:

```bash
sudo mkdir -p /var/log/qube
sudo chown root:root /var/log/qube
```

---

## 9. How the Self-Healing Works (Flow)

1. MASTER node fails
    
2. BACKUP node wins VRRP election
    
3. VIP moves to BACKUP
    
4. `notify.sh` triggers
    
5. Script checks Docker Swarm leader
    
6. If no leader → `docker swarm init --force-new-cluster`
    
7. Services recover on new MASTER
    

---

## 10. Validation & Testing

### Check VIP Ownership

```bash
ip addr show wlan0
```

or

```bash
ip a | grep 192.168.37.5
```

---

### Simulate Failover

```bash
sudo systemctl stop keepalived
```

On BACKUP node, verify VIP appears and Swarm recovers.

---

## 11. Common Pitfalls

|Issue|Cause|
|---|---|
|VIP not moving|Firewall blocking VRRP (protocol 112)|
|Split-brain|Same priority on multiple nodes|
|Swarm not recovering|Docker not installed / permissions issue|
|Notify script not running|Script not executable|

---

## 12. Firewall Notes (If Enabled)

Allow VRRP traffic:

```bash
sudo ufw allow proto vrrp
```

Or for iptables:

```bash
iptables -A INPUT -p vrrp -j ACCEPT
```

---

## 13. Summary

- Keepalived provides **IP-level HA**
    
- VRRP ensures **automatic failover**
    
- Notify scripts enable **application-level recovery**
    
- Combined with Docker Swarm → **fully self-healing control plane**
    

---

```

If you want next:
- A **MASTER/BACKUP diff config**
- A **systemd + keepalived HA diagram**
- Or a **Swarm + Keepalived reference architecture**

Just say the word.
```