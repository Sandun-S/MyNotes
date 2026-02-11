# SSH Access Reference – Personal Infra Notes

Owner: Sandun  
Purpose: Central reference for SSH targets, port forwards, and DB tunnels  
Config File: ~/.ssh/config  

---

# 1. SSH Config File (DO NOT MODIFY)

Path:

```bash
~/.ssh/config
````

Content:

```ssh
Host github.com
        HostName github.com
        User git
        IdentityFile ~/.ssh/id_ed25519
 
Host gitlab.com
        HostName gitlab.com
        User git
        IdentityFile ~/.ssh/id_ed25519
 
Host ora-test
        Hostname 68.233.117.44
        User sanduns
 
Host shysha
        Hostname 172.25.164.136
        User sanduns
 
Host local
        Hostname 172.25.164.140
        User sanduns
 
Host qube-ccs
        Hostname 124.43.225.45
        User mitiot
        Port 2002
        IdentityFile ~/.ssh/mit-iot-key
        LocalForward 1880 127.0.0.1:1880

Host sandun
        Hostname vm.mshome.net
        User ubuntu
```

---

# 2. Git Hosting

## GitHub

```bash
ssh -T git@github.com
```

Key:

```
~/.ssh/id_ed25519
```

---

## GitLab

```bash
ssh -T git@gitlab.com
```

Key:

```
~/.ssh/id_ed25519
```

---

# 3. Qube CCS Server

**Public IP:** 124.43.225.45  
**Port:** 2002  
**User:** mitiot  
**Key:** ~/.ssh/mit-iot-key

---

## Using SSH Config (Preferred)

```bash
ssh qube-ccs
```

Automatically:
- Uses port 2002
- Uses mit-iot-key
- Creates local forward for Node-RED

### Local Port Forward

```
LocalForward 1880 → 127.0.0.1:1880
```

Access Node-RED locally at:

```
http://localhost:1880
```

---

## Manual Equivalent Command

```bash
ssh -i ~/.ssh/mit-iot-key \
    -p 2002 \
    mitiot@124.43.225.45 \
    -L 1880:127.0.0.1:1880
```

---

# 4. Oracle Cloud – ora-test

**Public IP:** 68.233.117.44  
**Alias:** ora-test  
**User:** sanduns

---

## Basic SSH

```bash
ssh ora-test
```

---

## PostgreSQL Tunnel

Forward remote PostgreSQL to local machine:

```bash
ssh ora-test -L 5432:127.0.0.1:5432
```

Then connect locally:

```bash
psql -h localhost -p 5432 -U admin -d qube
```

---

## Port 8090 Tunnel

```bash
ssh ora-test -L 8090:127.0.0.1:8090
```

Access locally:

```
http://localhost:8090
```

---

# 5. Shysha Server (Shishyadara)

**IP:** 172.25.164.136  
**User:** sanduns

---

## SSH Access

```bash
ssh shysha
```

OR manual:

```bash
ssh sanduns@172.25.164.136
```

Password:

```
mit123
```

---

## Web Services

SQL Lab:

```
http://172.25.164.136/dash/sqllab/
```

Credentials:

```
admin / admin
```

Meta Page:

```
http://172.25.164.136/meta/getting-started
```

---

# 6. Local MIT ESP VM

**IP:** 172.25.164.140  
**DNS:** eng.mitesp.com  
**User:** sanduns

---

## SSH Access

```bash
ssh local
```

OR

```bash
ssh sanduns@eng.mitesp.com
```

Password:

```
mit123
```

---

# 7. Hyper-V / Multipass VM (sandun)

**Hostname:** vm.mshome.net  
**User:** ubuntu

---

## SSH

```bash
ssh sandun
```

Equivalent:

```bash
ssh ubuntu@vm.mshome.net
```

Used for:

- Local lab VMs
- Nexus
- Keycloak
- Swarm testing

---

# 8. Common Tunnel Patterns

## Generic Local Forward

```bash
ssh <host> -L <local_port>:127.0.0.1:<remote_port>
```

Example:

```bash
ssh ora-test -L 5432:127.0.0.1:5432
```

---

## Background Tunnel

```bash
ssh -N -L 5432:127.0.0.1:5432 ora-test
```

---

## Kill Tunnel

Find process:

```bash
ps aux | grep ssh
```

Kill:

```bash
kill <pid>
```

---

# 9. Security Notes

- Private key used for CCS: `~/.ssh/mit-iot-key`
- Default key for Git: `~/.ssh/id_ed25519`
- Some internal systems use passwords (should migrate to key-based auth)
- Avoid exposing 5432 publicly — always tunnel via SSH

---

# 10. Quick Command Summary

| Target       | Command                 |
| ------------ | ----------------------- |
| GitHub       | `ssh -T git@github.com` |
| GitLab       | `ssh -T git@gitlab.com` |
| Qube CCS     | `ssh qube-ccs`          |
| Oracle       | `ssh ora-test`          |
| Shysha       | `ssh shysha`            |
| Local MIT VM | `ssh local`             |
| Hyper-V VM   | `ssh sandun`            |
