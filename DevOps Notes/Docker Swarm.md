
# 1️ Swarm Infrastructure

## Cluster Initialization

```bash
sudo docker swarm init --advertise-addr <STATIC_IP>
sudo docker swarm init --advertise-addr 172.26.24.71
```

Get join tokens:

```bash
sudo docker swarm join-token manager
sudo docker swarm join-token worker
```

Get a new manager join token:

```bash
sudo docker swarm join-token manager
```

---

## Node Management

List nodes:

```bash
docker node ls
```

Check tasks running on a specific node:

```bash
docker node ps <node_name>
docker node ps manager1
```

Remove dead/unreachable node:

```bash
sudo docker node demote <node_id>
sudo docker node demote manager1

sudo docker node rm <node_id>
sudo docker node rm manager1
```

Leave swarm (reset node):

```bash
sudo docker swarm leave --force
```

---

# 2️ Stacks & Services

## Stack Deployment

Deploy / Update stack:

```bash
docker stack deploy -c <file>.yml <stack_name>
docker stack deploy -c docker-compose.yml portal
```

List stacks:

```bash
docker stack ls
```

List services in stack:

```bash
docker stack services <stack_name>
docker stack services portal
```

Verify stack / task failures:

```bash
docker stack ps <stack_name>
docker stack services portal
```

Remove stack:

```bash
docker stack rm <stack_name>
docker stack rm superset_test
```

---

## Services (Direct Management)

Create service:

```bash
sudo docker service create --name <name> --replicas 3 -p 80:80 <image>
sudo docker service create --name web-test --replicas 3 -p 80:80 nginx
```

List services:

```bash
sudo docker service ls
```

Scale service:

```bash
sudo docker service scale <name>=<number>
sudo docker service scale web-test=5
```

Inspect service:

```bash
docker service inspect <service_name> --pretty
docker service inspect <name> --pretty
```

Where are replicas running?

```bash
sudo docker service ps <service_name>
sudo docker service ps web-test
```

---

# 3️ Networks

List networks:

```bash
docker network ls
```

Inspect network:

```bash
docker network inspect <network_name>
docker network inspect portal-net
```

---

# 4️ Logs & Monitoring

Live service logs:

```bash
docker service logs -f <service_name>
docker service logs -f portal_http-fs
```

Container logs:

```bash
docker logs -f --tail 100 <container_id>
```

Check images:

```bash
docker images
docker images | grep qube-docs
```

---

# 5️ Maintenance & Cleanup

Remove stopped containers:

```bash
docker container prune
```

Prune everything (stopped containers + unused images):

```bash
docker system prune
```

Delete volume:

```bash
docker volume rm <volume_name>
docker volume rm superset_test_postgres_data
```

Limit old task history (reduce disk usage):

```
docker swarm update --task-history-limit 1
```

---

# 6️ Quick Diagnostics Flow (When Something Fails)

Check nodes
```
docker node ls
```
Check stack
```
docker stack ps <stack_name>
```
Check service
```
docker service ps <service_name>
```
Check logs
```
docker service logs -f <service_name>
```
Inspect service
```
docker service inspect <service_name> --pretty
```
