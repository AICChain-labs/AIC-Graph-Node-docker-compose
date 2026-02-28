# AIC Graph Node Container Deployment Instructions

This document is responsible for creating and running the basic AIC Graph Node services (`postgres`, `ipfs`, `graph-node`). It does not include subgraph deployment steps.

Applicable Environment:
- AIC RPC: `YOUR_RPC_URL`


---

## 1. Install Docker and Compose

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
docker --version
docker compose version
```

---

## 2. Prepare Directory and Container Orchestration File

```bash
sudo mkdir -p /opt/graph-node
sudo chown -R $USER:$USER /opt/graph-node
cd /opt/graph-node
```

Copy the `graph-node/docker-compose.yml` from this repository to `/opt/graph-node/docker-compose.yml`.

Or you can create it directly:

```bash
cp /path/to/v2-subgraph-main/graph-node/docker-compose.yml /opt/graph-node/docker-compose.yml
```

---

## 3. Start Containers

```bash
cd /opt/graph-node
docker compose up -d
docker compose ps
```

View logs:

```bash
docker compose logs -f graph-node
```

---

## 4. Fix Postgres Locale for the First Time (Graph Node Requires C)

If the following appears in the logs:
- `Database does not use C locale`

Execute the following commands to fix it:

```bash
cd /opt/graph-node
docker compose stop graph-node

docker exec -it graph-postgres psql -U graph -d postgres -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='graph';"
docker exec -it graph-postgres psql -U graph -d postgres -c "DROP DATABASE IF EXISTS graph;"
docker exec -it graph-postgres psql -U graph -d postgres -c "CREATE DATABASE graph WITH OWNER=graph TEMPLATE=template0 ENCODING='UTF8' LC_COLLATE='C' LC_CTYPE='C';"
docker exec -it graph-postgres psql -U graph -d postgres -c "SELECT datname, datcollate, datctype FROM pg_database WHERE datname='graph';"

docker compose up -d graph-node
docker compose logs -f graph-node
```

Expect to see the `graph` database as `C / C`.

---

## 5. Firewall and Port Recommendations

When exposing externally only through Nginx, it is recommended to only allow:
- `22/tcp`
- `80/tcp`
- `443/tcp`

The `graph-node`, `ipfs`, and `admin` ports are already bound to `127.0.0.1` and will not be directly exposed to the public network.

---

## 6. Run Status Verification (Independent of Subgraphs)

### 6.1 Check index-node Locally

```bash
curl -s http://127.0.0.1:8030/ | head
```

### 6.2 Check RPC Connectivity

```bash
curl -s -X POST http://YOUR_RPC_URL:8545 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
```

### 6.3 Check graph-node Log Keywords

```bash
docker compose logs --tail=200 graph-node
```

Should include:
- `Starting GraphQL HTTP server at: http://localhost:8000`
- `Starting JSON-RPC admin server at: http://localhost:8020`
- `Starting index node server at: http://localhost:8030`
- `Starting block ingestor for network, network_name: aic-mainnet`

---

## 7. Common Operations Commands

```bash
cd /opt/graph-node

# Start/Stop/Restart
docker compose up -d
docker compose stop
docker compose restart

# View status and logs
docker compose ps
docker compose logs -f graph-node

# Update images
docker compose pull
docker compose up -d
```
