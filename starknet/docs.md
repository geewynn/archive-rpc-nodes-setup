# Starknet Juno Full Node — Docker Guide

## Overview

[Juno](https://juno.nethermind.io) is a Go implementation of a Starknet full‑node client created by Nethermind. Running your own Juno node lets you query Starknet directly, contribute to the network’s resilience, and power downstream services such as indexers and dApps.

## System Requirements

| Resource | Minimum                                              |
| -------- | ---------------------------------------------------- |
| **CPU**  | 4 cores                                              |
| **OS**   | Debian 12 / Ubuntu 22.04                             |
| **RAM**  | ≥ 8 GB                                               |
| **Disk** | ≥ 500 GB |

## Pre‑Requisites

### Update & essential packages

```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y
sudo apt install -y wget curl screen git ufw
```

### Firewall configuration

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80       # HTTP
sudo ufw allow 443      # HTTPS
```

Enable the firewall:

```bash
sudo ufw enable
```

## Install Docker Engine & Compose

Remove any conflicting packages:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove $pkg
done
```

Add Docker’s GPG key & repository, then install:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Smoke tests
sudo docker run hello-world
docker compose version
```

## Directory Layout

```bash
mkdir starknet && cd starknet
```

> **Important:** Juno requires an **Ethereum mainnet WebSocket endpoint** (e.g. `wss://your-l1-endpoint`).

## `docker-compose.yml`

Create and open the file:

```bash
nano docker-compose.yml
```

Paste the following:

```yaml
version: '3.9'

networks:
  monitor-net:
    driver: bridge

volumes:
  juno_data: {}

services:
  juno:
    image: nethermind/juno:v0.14.2
    container_name: juno
    user: root
    restart: unless-stopped
    volumes:
      - "/var/lib/juno-data:/data"
    command:
      - --db-path=/data
      - --network=mainnet
      - --http
      - --http-port=6060
      - --http-host=0.0.0.0
      - --metrics
      - --metrics-host=0.0.0.0
      - --metrics-port=9090
      - --rpc-cors-enable
      - --ws
      - --ws-port=6061
      - --ws-host=0.0.0.0
      - --log-level=trace
      - --eth-node=wss://<l1-endpoint>
    expose:
      - 6060
      - 5050
      - 9090
      - 8545
      - 6061
    ports:
      - "5050:5050"   # P2P
      - "6060:6060"   # HTTP RPC
      - "9090:9090"   # Metrics
      - "8545:8545"   # JSON‑RPC (Ethereum)
      - "6061:6061"   # WebSocket
    networks:
      - monitor-net
```

## Launch the Node

```bash
docker compose up -d
```

## Monitor Logs

```bash
docker logs juno -f --tail 100
```

Typical sync output:

```
10:17:08.292 INFO   migration/migration.go:110  Applying database migration     {"stage": "18/18"}
...
TRACE jsonrpc/server.go:451   Received request  {"method":"starknet_blockNumber"}
```

## Health Checks

### Block Number

```bash
curl -s -X POST http://localhost:6060 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"starknet_blockNumber","params":[],"id":1}'
```

### Sync Status

```bash
curl -s -X POST http://localhost:6060 \
  -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"starknet_syncing","params":[],"id":1}'
```

A response of `false` means your node is fully synced. Expect a **\~5‑day** sync time from genesis on recommended hardware.

## References

* [Starknet Juno documentation](https://juno.nethermind.io)
* [InfraDAO – Archive Nodes 101 – Starknet / Juno / Docker](https://docs.infradao.com/archive-nodes-101/starknet/juno/docker)
