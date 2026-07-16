# AWS Docker Swarm Orchestration

A three-tier application (React/Nginx frontend, Flask backend, PostgreSQL database) deployed locally with Docker Compose, then migrated to a **3-node Docker Swarm cluster** running on AWS EC2 (1 manager + 2 workers).

This README documents the full journey from local development to a working multi-node Swarm deployment, including the real issues hit along the way and how they were fixed.

---

## Architecture

### Application layer

- **Frontend**: Nginx serving static files + reverse-proxying `/api/` calls to the backend, so the browser only ever talks to one origin (avoids CORS).
- **Backend**: Flask + Gunicorn API on port 8000.
- **Database**: Postgres 16 (Alpine), initialized from `database/init.sql`.

### Infrastructure layer — 3-node Docker Swarm cluster on EC2

![AWS architecture diagram — 3-node Docker Swarm cluster](/Users/thantzinmin/Devops Practice/devops-handbook/aws-docker-swarm-orchestration/architecture-diagram.png)

```

Docker Swarm's **routing mesh** means any node's public IP on port `3000` will correctly route the request to wherever the `frontend` service replica actually lives — the app doesn't need to be running on the node you connect to.

</details>

---

## Stage 1 — Local development with Docker Compose

### `docker-compose.yaml`
Defines all 3 services (`frontend`, `backend`, `db`) on a single `bridge` network, using `build:` to build images locally from each service's `Dockerfile`.

Key details:
- `db` has **no `ports:` mapping** — it's only reachable inside the Docker network (`backend` connects via the internal hostname `db:5432`), so it never conflicts with a local Postgres install on the host machine.
- `backend` uses `depends_on: db: condition: service_healthy`, gated by a Postgres healthcheck (`pg_isready`).
- `.env` supplies `FRONTEND_PORT`, `BACKEND_PORT`, `DB_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`, etc.

### Commands used

```bash
# Build and run all 3 services locally
docker compose up --build

# Check running containers
docker ps
```

### Issues hit and fixed
- **Port conflict on 3000**: `http://localhost:3000` was showing a raw directory listing instead of the app. Root cause: a VS Code "Live Preview" process (`Code Helper`) was already bound to port 3000, intercepting all requests before Docker's Nginx container ever saw them.
  ```bash
  lsof -i :3000        # found the conflicting process (Code Helper)
  kill <PID>            # freed the port
  docker ps             # confirmed ldn-frontend now owns 0.0.0.0:3000->80/tcp
  ```

---

## Stage 2 — Understanding `nginx.conf`

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri /index.html;
    }

    location /api/ {
        proxy_pass http://backend:8000/api/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

- `try_files $uri /index.html` — SPA fallback so client-side routing doesn't 404 on refresh.
- `location /api/` — reverse proxy to the backend using Docker's internal DNS (`backend` resolves to the backend container/service on the same network).
- The `proxy_set_header` lines preserve the original `Host`, client IP, and forwarding chain, since the backend otherwise only sees requests coming from Nginx.

---

## Stage 3 — Preparing for Docker Swarm

Swarm mode (`docker stack deploy`) has key differences from Compose:

| Compose feature | Swarm support |
|---|---|
| `build:` | ❌ Not supported — must use pre-built `image:` |
| `depends_on: condition: service_healthy` | ❌ Ignored — app must handle DB-not-ready itself |
| `container_name:` | ❌ Ignored — Swarm manages naming/replicas itself |
| `networks: driver: bridge` | 🔁 Must become `driver: overlay` for cross-node networking |
| `deploy:` (replicas, restart_policy) | ✅ New in Swarm, has no effect in plain Compose |

### Build and push images to Docker Hub

```bash
docker login

docker build -t tommyzizii/ldn-frontend:latest ./frontend
docker build -t tommyzizii/ldn-backend:latest ./backend

docker push tommyzizii/ldn-frontend:latest
docker push tommyzizii/ldn-backend:latest
```

### `docker-stack.yml`

```yaml
services:
  frontend:
    image: tommyzizii/ldn-frontend:latest
    ports:
      - "${FRONTEND_PORT}:80"
    networks:
      - app-network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

  backend:
    image: tommyzizii/ldn-backend:latest
    environment:
      FLASK_ENV: ${FLASK_ENV}
      DB_HOST: ${DB_HOST}
      DB_PORT: ${DB_PORT}
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "${BACKEND_PORT}:8000"
    networks:
      - app-network
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure

networks:
  app-network:
    driver: overlay

volumes:
  postgres_data:
```

### Handling `.env` in Swarm

`docker stack deploy` does **not** read `.env` automatically the way `docker compose up` does. Variables must be exported into the shell first:

```bash
export $(cat .env | xargs)

# Verify
echo $POSTGRES_DB
echo $POSTGRES_USER

# Validate the fully-resolved stack file before deploying
docker stack config -c docker-stack.yml
```

### Test locally in single-node Swarm first

```bash
docker swarm init
docker stack deploy -c docker-stack.yml ldn

docker stack services ldn
docker service logs ldn_backend
docker service logs ldn_db

# Tear down before moving to EC2
docker stack rm ldn
docker swarm leave --force
```

---

## Stage 4 — Building the 3-node EC2 Swarm cluster

### 4.1 Security group (`swarm-cluster-sg`)

Created once, attached to all 3 EC2 instances in the default VPC.

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | My IP | SSH access |
| Custom TCP | TCP | 2377 | This SG (self) | Swarm cluster management |
| Custom TCP | TCP | 7946 | This SG (self) | Node gossip/discovery |
| Custom UDP | UDP | 7946 | This SG (self) | Node gossip/discovery |
| Custom UDP | UDP | 4789 | This SG (self) | Overlay network data (VXLAN) |
| Custom TCP | TCP | 3000 | Anywhere | Frontend access |
| Custom TCP | TCP | 8000 | My IP | Backend debugging (optional) |

> **Note**: the self-referencing rules (2377, 7946, 4789) can only be set *after* the security group exists, since you need its own ID as the source. Create the SG with the other rules first, then edit it to add these.

### 4.2 Launch 3 EC2 instances

- 1× `swarm-manager`, 2× `swarm-worker`
- Ubuntu 24.04 LTS, `t3.small`, same VPC/subnet, same key pair, `swarm-cluster-sg` attached
- Public IP auto-assigned for SSH access

### 4.3 Install Docker on all 3 nodes

```bash
sudo apt update
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# log out and back in
docker --version
docker ps
```

### 4.4 Initialize the Swarm

On the **manager**:
```bash
docker swarm init --advertise-addr <MANAGER_PRIVATE_IP>
```

Retrieve the join command any time with:
```bash
docker swarm join-token worker
```

On **each worker**:
```bash
docker swarm join --token SWMTKN-1-xxxx... <MANAGER_PRIVATE_IP>:2377
```

Verify from the manager:
```bash
docker node ls
```

### 4.5 Get project files onto the manager

```bash
scp -i swarm-key.pem docker-stack.yml .env ubuntu@<MANAGER_PUBLIC_IP>:~/
scp -i swarm-key.pem database/init.sql ubuntu@<MANAGER_PUBLIC_IP>:~/database/init.sql
```

### 4.6 Deploy the stack

```bash
cd ~
export $(cat .env | xargs)
docker stack config -c docker-stack.yml     # sanity check
docker stack deploy -c docker-stack.yml ldn
docker stack services ldn
docker service ps ldn_backend
docker service ps ldn_frontend
docker service ps ldn_db
```

---

## Issue: ARM64 vs x86_64 architecture mismatch

**Symptom:**
```
"no suitable node (unsupported platform)"
```

**Cause:** Images were built on an Apple Silicon (ARM64) Mac using a plain `docker build`, which defaults to the host's architecture. The EC2 instances are x86_64 — Swarm refused to schedule ARM64 images onto them.

**Fix:** rebuild explicitly for the target platform and push directly:

```bash
docker buildx create --use   # one-time setup if buildx isn't active yet

docker buildx build --platform linux/amd64 -t tommyzizii/ldn-backend:latest ./backend --push
docker buildx build --platform linux/amd64 -t tommyzizii/ldn-frontend:latest ./frontend --push
```

Then force the running services to re-pull the corrected images:

```bash
docker service update --force ldn_backend
docker service update --force ldn_frontend
```

---

## Final verification

```bash
docker stack services ldn        # all services show full replica counts (e.g. 2/2, 1/1)
docker service ps ldn_backend     # confirm "Running", spread across different nodes
docker service ps ldn_frontend
docker service ps ldn_db
```

App reachable at `http://<ANY_NODE_PUBLIC_IP>:3000` — thanks to Swarm's **routing mesh**, any node's public IP serves the app correctly on port 3000, regardless of which physical node the frontend container actually landed on.

---

## Key takeaways

- `db` with no `ports:` mapping keeps Postgres isolated from the host — safer, and avoids conflicts with local Postgres/MySQL installs.
- Swarm mode requires pre-built, registry-hosted images (`build:` doesn't work) and ignores `depends_on` health conditions — apps must handle "dependency not ready yet" themselves.
- `overlay` networking (vs `bridge`) is what allows containers to discover each other by service name across different physical EC2 nodes.
- `.env` files must be manually exported into the shell before `docker stack deploy` — Swarm doesn't read them automatically like Compose does.
- Cross-architecture builds (Apple Silicon dev machine → x86_64 cloud servers) require `docker buildx --platform linux/amd64 --push` instead of a plain `docker build` + `docker push`.
- Self-referencing security group rules (source = the security group itself) are what let Swarm nodes talk to each other on ports 2377/7946/4789 without hardcoding IPs.