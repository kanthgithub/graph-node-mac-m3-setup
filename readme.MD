# Graph Node Local Setup - Complete Guide

This guide covers the complete, working, debugged setup of a local Graph Node using Postgres and IPFS in Docker, along with building the graph-node binary from source.

⸻

✅ Prerequisites
	•	macOS (Tested on M3 MacBook Pro)
	•	Docker & Docker Compose
	•	Rust & Cargo (with Rustup)
	•	Git

⸻

1. Clone and Build Graph Node

# Clone graph-node source
mkdir ~/Developer
cd ~/Developer
https://github.com/graphprotocol/graph-node.git
cd graph-node

# Build the release binary
cargo build --release

⸻

2. config.toml 

```toml
[graph]
ipfs = "http://192.168.0.41:5001"

[store]
[store.primary]
connection = "postgresql://graph-node:graph-node@localhost:5432/graph-node"
pool_size = 10

[chains]
ingestor = "mainnet"

[chains.mainnet]
shard = "primary"
provider = [
  { label = "local", url = "http://localhost:8545", features = ["archive"] }
]

[[deployment.rule]]
indexers = ["default"]

[[deployment.rule.pattern]]
kind = "ethereum/contract"
network = "mainnet"

[indexer]
id = "default"

[log]
level = "info"

[metrics]
port = 8040
```

3. Docker Compose Setup

Create a docker-compose.yml file under a sibling examples/ directory:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15
    container_name: examples-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: root
      LANG: C
      LC_ALL: C
    command: postgres -c client_encoding=UTF8
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U root"]
      interval: 10s
      timeout: 5s
      retries: 5

  ipfs:
    image: ipfs/kubo:latest
    container_name: examples-ipfs
    restart: unless-stopped
    ports:
      - "4001:4001"
      - "5001:5001"
      - "8180:8080"
      - "8081:8081"
    volumes:
      - ipfs_staging:/export
      - ipfs_data:/data/ipfs
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5001/api/v0/id"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  pgdata:
  ipfs_staging:
  ipfs_data:
```

⸻

4. init.sql - Postgres Role & DB Setup

 - Create init.sql in same examples/ directory:

```sql
- Drop and recreate the graph-node role
DROP ROLE IF EXISTS "graph-node";
CREATE ROLE "graph-node" WITH LOGIN PASSWORD 'graph-node';
ALTER ROLE "graph-node" WITH SUPERUSER;

-- Drop and recreate the graph-node database with correct settings
DROP DATABASE IF EXISTS "graph-node";
CREATE DATABASE "graph-node"
  WITH OWNER = "graph-node"
  ENCODING = 'UTF8'
  LC_COLLATE = 'C'
  LC_CTYPE = 'C'
  TEMPLATE = template0;
```

⸻

5. Start Docker

```sh
cd ~/Developer/examples
```

# Recreate containers and volumes

```sh
rm -rf ./pgdata
```

# Start services

```sh
docker compose up -d --force-recreate
```

# Wait until Postgres and IPFS are healthy

⸻

5. Validate DB Encoding and Locale

```sh
docker exec -it examples-postgres psql -U graph-node -d graph-node -c "SHOW LC_COLLATE;"
docker exec -it examples-postgres psql -U graph-node -d graph-node -c "SHOW SERVER_ENCODING;"
```

 - Expected output:
	•	LC_COLLATE: C
	•	SERVER_ENCODING: UTF8

 - If not:
	•	Destroy the volume: docker volume rm examples_pgdata
	•	Recreate containers

⸻

6. Run Graph Node Locally

```sh
cd ~/Developer/graph-node
./target/release/graph-node \
  --config ../examples/config.toml \
  --ipfs 192.168.0.41:5001
```

Replace IPFS IP with docker inspect examples-ipfs if needed.

⸻

7. Health Checks (No Subgraphs Yet)

```sh
curl http://localhost:8040/metrics
```

- Key values:
  - deployment_count 0 ✅ OK
  - eth_rpc_status{provider="local"} 0 ✅ Healthy
  - store_connection_error_count 0 ✅ Good

⸻

🛠️ Troubleshooting

- Issue: SQL_ASCII encoding
    - Happens when Postgres creates DB with default LC/ENCODING
	- Solution: Use TEMPLATE = template0 in init.sql

- Issue: Role doesn’t exist
	- Check logs: docker logs examples-postgres
	- Ensure init.sql ran (check docker-compose.yml volume mount)

Issue: permission denied to create extension "postgres_fdw"
	- Your graph-node role must be SUPERUSER

Issue: Graph node panics
	- Usually caused by invalid Postgres config, bad locale, or encoding mismatch

⸻

✅ Final Output on Success

```
INFO Successfully connected to IPFS RPC API at: '192.168.0.41:5001'
INFO Pool successfully connected to Postgres
INFO Migrations finished
INFO Starting GraphQL HTTP server at: http://localhost:8000
INFO Metrics server at: http://localhost:8040
```

⸻

📌 Next Steps
	- Deploy a subgraph using graph deploy ...
	- Point a frontend dApp to http://localhost:8000/subgraphs/name/...

⸻

For any issues, recheck:
	- Docker logs (docker logs examples-postgres)
	- Health output (curl localhost:8040/metrics)
	- Encoding (SHOW SERVER_ENCODING;)
	- Locale (SHOW LC_COLLATE;)