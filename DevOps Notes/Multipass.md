# Multipass VM Notes (Hyper-V Backend – Windows)

**Date:** Friday, January 30, 2026  
**Host OS:** Windows 10 / 11 Pro  
**Backend:** Microsoft Hyper-V  
**Shells:** PowerShell (Admin), WSL 2  
**Guest OS:** Ubuntu (cloud images)

---

## 1. What Multipass Is (Short)

- Lightweight VM manager by Canonical
- Uses **cloud images** (headless Ubuntu by default)
- On Windows Pro, runs on **Hyper-V**
- Designed for **fast dev / infra / lab environments**
- Not meant to replace full Hyper-V Manager workflows

Official install docs:  
https://canonical.com/multipass/install

---

## 2. Verify Hyper-V Backend

Multipass supports multiple backends.  
On Windows **you must explicitly use Hyper-V**.

```powershell
multipass set local.driver=hyperv
````

Verify:

```powershell
multipass get local.driver
```

Expected output:

```
hyperv
```

---

## 3. Basic VM Lifecycle Management

### List All Instances

```bash
multipass list
```

---

### Launch a New VM

```bash
multipass launch \
  --name manager1 \
  --cpus 1 \
  --memory 1G
```

Recommended for servers (Docker / Nexus / CI):

```bash
multipass launch \
  --name nexus-server \
  --cpus 2 \
  --memory 4G \
  --disk 20G
```

---

### Access VM Shell

```bash
multipass shell manager1
```

> This opens a rootless Ubuntu shell (`ubuntu` user).

---

### Stop / Start VM

```bash
multipass stop manager1
multipass start manager1
```

---

### Delete VM

```bash
multipass delete manager1
```

⚠️ This only **marks the VM for deletion**.

To actually free disk space:

```bash
multipass purge
```

---

### Find Available Images

```bash
multipass find
```

Examples:
- `22.04` → Ubuntu Jammy
- `20.04` → Ubuntu Focal
- `daily` → bleeding edge (not recommended)

---

## 4. Networking Model (CRITICAL)

### Hyper-V Default Switch Behavior

Multipass on Windows **always uses the Hyper-V Default Switch**.

Key characteristics:
- NAT-based
- Subnet **changes on every Windows reboot**
- IP addresses are **dynamic**
- Managed entirely by Hyper-V

---

### 🚨 CRITICAL WARNING (Read This)

❌ **DO NOT configure static IPs inside the VM**  
❌ **DO NOT modify Netplan when using Hyper-V Default Switch**

#### Why?

- Hyper-V regenerates the subnet dynamically
- A static IP will **break connectivity permanently**
- Result: SSH timeout, unreachable VM

---

## 5. Netplan (Why You Usually Should NOT Touch It)

Default cloud-init netplan file:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Typical content:

```yaml
network:
  version: 2
  ethernets:
    default:
      match:
        macaddress: "52:54:00:71:5b:70"
      dhcp-identifier: mac
      dhcp4: true
```

### ✅ Correct State (Hyper-V)

- `dhcp4: true`
- No static addresses
- No routes modified

### ❌ When Static IPs Are Allowed

Only when:

- Using **bridged networking**
- Or running Multipass on **Linux**
- Or using **custom Hyper-V switches**

Not with Default Switch.

---

## 6. Correct Way to Access VMs (DNS-Based)

### Magic DNS Name (Recommended)

Hyper-V provides automatic DNS resolution:

```
<vm-name>.mshome.net
```

Example:

```bash
ssh ubuntu@nexus-server.mshome.net
```

✅ Stable  
✅ No IP tracking  
✅ Works across reboots

---

## 7. WSL 2 Integration (Smart SSH)

### Problem

- WSL cannot resolve `*.mshome.net`
- SSH fails even though VM is running

---

### Solution: SSH Wrapper in WSL

Add this to `~/.bashrc` in WSL:

```bash
ssh() {
    if [[ "$1" == "manager1" || "$1" == "nexus-server" ]]; then
        multipass.exe shell "$1"
    else
        /usr/bin/ssh "$@"
    fi
}
```

Result:

```bash
ssh nexus-server
```

➡️ Transparently opens `multipass shell`

This preserves:

- GitHub SSH
- Remote servers
- Normal SSH behavior

---

## 8. Troubleshooting

---

### Problem: IPv4 shows `N/A` or SSH Timeout

**Cause:**
- Hyper-V Default Switch reset
- Windows sleep / reboot

**Fix (Admin PowerShell):**

```powershell
Restart-Service multipassd
```

If still broken:

```powershell
multipass restart <name>
```

---

### Problem: VM "Already Running" but inaccessible

**Fix: Hard reset Multipass services**

```powershell
multipass stop --all
taskkill /IM multipassd.exe /F
Start-Service multipassd
```

---

## 9. Advanced Configuration

---

### Disable VM Auto-Start on Windows Boot

By default, Hyper-V restarts VMs automatically.

Steps:
1. Open **Hyper-V Manager**
2. Right-click VM → **Settings**
3. Go to **Management → Automatic Start Action**
4. Select **Nothing**

---

### Resize an Existing VM (Windows Limitation)

Multipass CLI cannot resize running VMs on Windows.

Steps:

```bash
multipass stop nexus-server
```

Then:

1. Open **Hyper-V Manager**
2. VM → Settings
3. Adjust:
    - **Processor** → vCPUs
    - **Memory** → RAM
        - Disable **Dynamic Memory** for servers (Nexus, Docker)

Then:

```bash
multipass start nexus-server
```

---

## 10. File Sharing (Mounting)

Mount a Windows folder into the VM:

```bash
# Syntax
multipass mount <Windows_Path> <VM_Name>:<Linux_Path>
```

Example:

```bash
multipass mount C:\Users\Sandun\Code nexus-server:/home/ubuntu/code
```

Inside VM:

```bash
ls ~/code
```

✅ Persistent  
✅ No SCP needed  
⚠️ Path permissions follow Windows ACLs

---

## 11. Recommended Usage Patterns

|Use Case|Recommendation|
|---|---|
|Docker / Nexus|2 CPU, 4GB RAM, static name|
|HA / Swarm labs|DNS access only|
|CI testing|Short-lived VMs|
|Prod-like infra|Avoid Default Switch|

---

## 12. Summary (Rules to Remember)

- Always use **Hyper-V backend**
- Never assign static IPs with Default Switch
- Use `*.mshome.net` DNS
- Use `multipass purge` to reclaim disk
- Treat Multipass VMs as **ephemeral infrastructure**