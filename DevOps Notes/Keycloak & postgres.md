## Keycloak Deployment & Configuration

Environment: Windows (Hyper-V) → Multipass → Ubuntu VM → Docker  
Mode: Single-node (Dev / Lab / Internal Auth)  
Primary Integration: OliveTin (OIDC)

---

# 1) VM Provisioning (Multipass on Windows)

Create VM:

```bash
multipass launch \
  --name keycloak-vm \
  --cpus 2 \
  --memory 4G \
  --disk 10G
```

Verify:

```bash
multipass list
```

Enter VM:

```bash
multipass shell keycloak-vm
```

---

# 2) Install Docker Engine (Inside VM)

Update system:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
```

Add Docker GPG:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add repo:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install:

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add user to docker group:

```bash
sudo usermod -aG docker $USER
exit
```

Re-enter shell:

```bash
multipass shell keycloak-vm
```

Verify:

```bash
docker version
```

---

# 3) Persistent Folder Structure (Host Inside VM)

Create production directory layout:

```bash
sudo mkdir -p /mit/eng/postgres/data
sudo chmod -R 777 /mit
```

Final structure:

```
/mit/eng/
├── docker-stack.yml
├── postgres/
│   └── data/
└── olivetin/
    └── configs/
        └── config.yaml
```

Important:

- All Keycloak realm data lives in Postgres
- Postgres persists via bind mount
- Deleting containers does NOT remove realm configuration

---

# 4) Docker Stack (Keycloak + Postgres)

docker-stack.yml:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: mit123
      POSTGRES_DB: keycloak
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - /mit/eng/postgres/data:/var/lib/postgresql/data
    deploy:
      restart_policy:
        condition: on-failure

  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    command: start-dev --http-relative-path /auth
    environment:
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME: admin
      KC_DB_PASSWORD: mit123
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT: "false"
      KC_HOSTNAME_URL: http://eng.mitesp.com:8080/auth
      KC_PROXY_HEADERS: "xforwarded"
    ports:
      - "8080:8080"
    depends_on:
      - postgres
```

Deploy:

```bash
docker compose up -d
```

Access:

[http://eng.mitesp.com:8080/auth](http://eng.mitesp.com:8080/auth)  
Login: admin / admin

---

# 5) Realm Setup

Create new realm:
Name: engineering
Reason:
- Isolates OliveTin from master realm
- Clean OIDC namespace

---

# 6) Client Configuration (olivetin)

Create client:

Client ID: olivetin  
Client authentication: ON  
Access Type: Confidential

Critical settings:

Valid Redirect URIs:

```
http://eng.mitesp.com:1337/*
```

Web Origins:

```
*
```

Save → Go to Credentials → Copy clientSecret  
This must match OliveTin config.yaml.

---

# 7) OpenID Scope Requirement (Keycloak 26 Fix)

Problem:  
Fresh installs may not include required openid scope.

Solution:

Create Client Scope:  
Name: openid  
Protocol: OpenID Connect

Assign to client:  
Clients → olivetin → Client Scopes → Add as Default

Without this:  
userinfo endpoint will not return group data.

---

# 8) Group Configuration (Standard Mapping Method)

Create group:
guest
Make default group:
Realm Settings → Default Groups → Add guest
Now every new user automatically receives guest group.

---

## Group Membership Mapper
Under:
Clients → olivetin → Dedicated Scopes → Add Mapper

Type: Group Membership  
Name: groups  
Token Claim Name: groups  
Add to userinfo: ON  
Add to ID token: ON  
Add to access token: ON  
Full group path: OFF

Expected userinfo output:

```json
{
  "preferred_username": "user1",
  "groups": "guest"
}
```

---

# 9) OliveTin Compatibility Fix (Hardcoded Claim)

Problem:  
OliveTin v3000.9.4 cannot parse:

```
"groups": ["guest"]
```

Error:  
Field groups is not a string

Solution: Hardcoded Claim

Delete all Group Membership mappers.

Add new mapper:

Type: Hardcoded Claim  
Name: group-hardcoded  
Token Claim Name: groups  
Claim value: guest  
Claim JSON Type: String  
Add to userinfo: ON  
Add to access token: ON  
Add to ID token: ON

Result:

```
"groups": "guest"
```

OliveTin ACL now works correctly.

---

# 10) Google Identity Provider Setup

Add provider:
Identity Providers → Google

Enter:
- Google Client ID
- Google Client Secret

Keycloak generates Redirect URI:

```
http://eng.mitesp.com:8080/auth/realms/engineering/broker/google/endpoint
```

Copy this into:

Google Cloud Console → Credentials → Authorized Redirect URIs

Without exact match → login fails.

---

# 11) Common Production Errors & Fixes

401 Unauthorized  
→ Client secret mismatch

Invalid Token Issuer  
→ Ensure both OliveTin and Keycloak use eng.mitesp.com consistently

Google 400 redirect_uri_mismatch  
→ Fix redirect URI in Google Console

Field groups is not a string  
→ Use Hardcoded Claim mapper

userinfo returns empty groups  
→ Ensure openid scope is Default  
→ Ensure mapper Add to userinfo = ON

---

# 12) Token Debugging

Get access token via login → then:

```bash
curl http://eng.mitesp.com:8080/auth/realms/engineering/protocol/openid-connect/userinfo \
  -H "Authorization: Bearer <token>"
```

Verify:

- preferred_username
- groups

If missing → mapper or scope misconfigured.

---

# 13) Production Considerations

Current mode: start-dev  
Not suitable for internet exposure.

For production:
- Use start (not start-dev)
- Enable HTTPS
- Set KC_HOSTNAME_STRICT=true
- Disable wildcard Web Origins
- Use reverse proxy (NGINX / Traefik)
- Secure Postgres password
- Remove chmod 777 in real deployments