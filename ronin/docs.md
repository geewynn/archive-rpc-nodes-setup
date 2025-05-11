# Docker — Ronin Chain (CIP Docker Guide Template)

*Author: **godwin**  
Last updated: **14 November 2024***  

---

## System Requirements

| Resource | Minimum |
|----------|---------|
| **CPU**  | 8 cores + |
| **OS**   | Ubuntu 24.04 |
| **RAM**  | 32 GB + |
| **Disk** | ≥ 1.2 TB (archive size on 14 Nov 2024) |

---

## Pre-Requisites

Update, upgrade and clean the system:

```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt auto-remove -y
sudo apt install ufw -y
```
## Configure Firewall Settings
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 8545
sudo ufw allow 8546
sudo ufw allow 30303
sudo ufw allow 6060
```
## Install Docker
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove $pkg
done
```
Add Docker’s official GPG key:

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
Add the repository to apt sources and install Docker Engine, CLI & Compose:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

# Verify docker-compose
docker compose version
```

## Setup Ronin Node
Create working directories
```bash
mkdir -p ~/ronin/docker
cd ~/ronin
mkdir -p chaindata/data/ronin
```

Create docker compose files
```bash
version: "3"
services:
  node:
    image: ${NODE_IMAGE}
    stop_grace_period: 5m
    stop_signal: SIGINT
    hostname: node
    container_name: node
    ports:
      - 127.0.0.1:8545:8545
      - 127.0.0.1:8546:8546
      - 30303:30303
      - 30303:30303/udp
      - 6060:6060
    volumes:
      - ~/ronin/chaindata:/ronin
    environment:
      - SYNC_MODE=full
      - PASSWORD=${PASSWORD}
      - NETWORK_ID=${NETWORK_ID}
      - RONIN_PARAMS=${RONIN_PARAMS}
      - VERBOSITY=${VERBOSITY}
      - MINE=${MINE}
      - GASPRICE=${GASPRICE}
      - ETHSTATS_ENDPOINT=${INSTANCE_NAME}:${CHAIN_STATS_WS_SECRET}@${CHAIN_STATS_WS_SERVER}:443
```
Create .env file
```ini
# Display name on https://ronin-stats.roninchain.com/
INSTANCE_NAME=<INSTANCE_NAME>

# Latest image tag (see docs)
NODE_IMAGE=<NODE_IMAGE>

# Password encrypting the node’s keystore
PASSWORD=<PASSWORD>

MINE=false
NETWORK_ID=2020
GASPRICE=20000000000
VERBOSITY=3

CHAIN_STATS_WS_SECRET=WSyDMrhRBe111
CHAIN_STATS_WS_SERVER=ronin-stats-ws.roninchain.com

RONIN_PARAMS=--http.api eth,net,web3,consortium --txpool.pricelimit 20000000000 --txpool.nolocals --cache 4096 \
--discovery.dns enrtree://AIGOFYDZH6BGVVALVJLRPHSOYJ434MPFVVQFXJDXHW5ZYORPTGKUI@nodes.roninchain.com
```

## Run the Node
```bash
cd ~/ronin/docker && docker-compose up -d
```

## Monitor the Node
```bash
docker logs node -f --tail 100
```

Result
```bash
INFO [11-14|21:24:17.122] Imported new chain segment   blocks=1 txs=11 mgas=1.280 elapsed=16.911ms mgasps=75.657 number=39,923,835 hash=1afff2..417928
```

##  Query the Node

Client version:
```bash
curl http://localhost:8545 -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":67}'
```

Result
```bash
{"jsonrpc":"2.0","id":67,"result":"ronin/v2.8.3-d27eb42e/linux-amd64/go1.20.10"}
```

Current block number
```bash
curl -X POST http://localhost:8545/ -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0", "method":"eth_blockNumber","params":[],"id":1}'
```
Result
```bash
curl -X POST http://localhost:8545/ -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0", "method":"eth_blockNumber","params":[],"id":1}'
```

## References
[Run a mainnet RPC node — Ronin Docs](https://docs.roninchain.com/rpc/mainnet-rpc)

[Run an archive node — Ronin Docs](https://docs.roninchain.com/validators/setup/mainnet/run-archive)
