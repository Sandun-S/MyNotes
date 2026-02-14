## OliveTin + Keycloak – Production Deployment Notes

Environment: Ubuntu (Docker Swarm / Multipass)  
Status: Final Working Configuration

---

# 1) System Architecture

### Overlay Network

All services run inside a Docker Swarm overlay network:

eng-net (driver: overlay)
### Services

**Postgres (v15)**
- Database backend for Keycloak
- Persistent host bind mount
- Prevents realm/user data loss

**Keycloak (v26)**
- OpenID Connect Identity Provider
- Manages realms, users, groups
- Bridges Google login → internal identity

**OliveTin**
- Web-based command executor
- Uses OIDC tokens for authentication
- Authorization based on:
    - preferred_username
    - groups
    - email substring match

---

# 2) Docker Stack Deployment

File: docker-stack.yml

```yaml
version: "3.8"

networks:
  eng-net:
    driver: overlay

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: mit123
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - /mit/eng/postgres/data:/var/lib/postgresql/data
    networks:
      - eng-net
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
    networks:
      eng-net:
        aliases:
          - eng.mitesp.com
    depends_on:
      - postgres

  olivetin:
    image: jamesread/olivetin:latest
    ports:
      - "1337:1337"
    volumes:
      - /mit/eng/olivetin/configs:/config
    networks:
      - eng-net
```

Deploy:

docker stack deploy -c docker-stack.yml eng-stack

---

# 3) OliveTin Configuration

File: /mit/eng/olivetin/configs/config.yaml

### Core Runtime Settings

```yaml
listenAddressSingleHTTPFrontend: 0.0.0.0:1337
authRequireGuestsToLogin: true
logLevel: "INFO"
authOAuth2RedirectURL: "http://eng.mitesp.com:1337/oauth/callback"
```

Important:
- authOAuth2RedirectURL fixes Google 400 redirect mismatch errors
- eng.mitesp.com alias resolves 401 token validation issues inside overlay network

---

## Actions

```yaml
actions:
  - title: "Check System Uptime"
    icon: "clock"
    shell: "uptime"
    acls: ["acl_guest", "acl_google", "acl_admin"]
```

---

## Access Control Lists

```yaml
accessControlLists:
  - name: acl_admin
    matchUsergroups: ["admins", "/admins"]
    permissions: { view: true, exec: true, logs: true }

  - name: acl_guest
    matchUsergroups: ["guest", "/guest"]
    permissions: { view: true, exec: true }

  - name: acl_google
    matchUsernameSubstrings: ["@gmail.com"]
    permissions: { view: true, exec: true }
```

Critical:
- exec: true required → prevents blurred buttons
- matchUsergroups requires groups claim from Keycloak
- matchUsernameSubstrings allows direct Google ACL

---

## OIDC Providers

### Keycloak Provider

```yaml
authOAuth2Providers:
  keycloak:
    title: "Login with Keycloak"
    clientId: "olivetin"
    clientSecret: "<secret>"
    authUrl: "http://eng.mitesp.com:8080/auth/realms/engineering/protocol/openid-connect/auth"
    tokenUrl: "http://eng.mitesp.com:8080/auth/realms/engineering/protocol/openid-connect/token"
    whoamiUrl: "http://eng.mitesp.com:8080/auth/realms/engineering/protocol/openid-connect/userinfo"
    usernameField: "preferred_username"
    userGroupField: "groups"
```

### Google Direct Provider (Optional)

```yaml
  google:
    title: "Login with Google (Direct)"
    clientId: "<google-client-id>"
    clientSecret: "<google-secret>"
    authUrl: "https://accounts.google.com/o/oauth2/v2/auth"
    tokenUrl: "https://oauth2.googleapis.com/token"
    whoamiUrl: "https://www.googleapis.com/oauth2/v3/userinfo"
    usernameField: "email"
```

---

# 4) Critical Keycloak Configuration

For group-based ACL to work:
## Client Scope
- Create or use openid scope
- Assign as Default to client: olivetin

## Mapper (Mandatory)

Mapper Type: Group Membership  
Name: groups  
Add to userinfo: ON  
Token Claim Name: groups

Important:
- Multivalued OFF (recommended for OliveTin parsing)
- Must appear in /userinfo response
- Validate with:

```
curl [http://eng.mitesp.com:8080/auth/realms/engineering/protocol/openid-connect/userinfo](http://eng.mitesp.com:8080/auth/realms/engineering/protocol/openid-connect/userinfo) -H "Authorization: Bearer "
```

Expected:
```
{  
"preferred_username": "...",  
"groups": "guest"  
}
```

If groups returns as array and OliveTin errors:  
```
"[guest] is not a string" → adjust mapper formatting.
```

---

# 5) Known Production Fixes

Problem → Fix
```
Google 400 redirect error  
→ Set authOAuth2RedirectURL correctly

401 invalid token inside cluster  
→ Use network alias eng.mitesp.com

Blurred buttons  
→ Set exec: true in ACL

Group parsing error  
→ Disable multivalued OR adjust claim formatting

```

---

# 6) Operational Verification

Check services:
```
docker service ls
```
Check logs:
```
docker service logs -f eng-stack_olivetin  
docker service logs -f eng-stack_keycloak
```

Test OIDC flow:
1. Login via Keycloak
2. Inspect userinfo
3. Confirm groups claim
4. Validate ACL execution enabled
