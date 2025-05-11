# Optimism Sepolia – **Op‑Geth**  Guide


## System Requirements

| Resource | Recommended      | Notes                                   |
| -------- | ---------------- | --------------------------------------- |
| **CPU**  | 4 – 8 cores      | Faster cores reduce sync time           |
| **OS**   | Ubuntu 24.04 LTS | Tested on LTS release                   |
| **RAM**  | 16 GB+           |                                         |
| **Disk** | 5 TB SSD         


## Pre‑Requisites

### Update the operating system

```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y
sudo apt install -y git gcc make --fix-missing
```

### Install Docker & Compose

```bash
# Update package lists
sudo apt-get update && sudo apt-get upgrade -y

# Prerequisites
sudo apt-get install -y curl gnupg ca-certificates lsb-release

# Add Docker’s GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add the Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine + Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add current user to the docker group
sudo usermod -aG docker $(whoami)

# Test
sudo docker run hello-world
```

## Firewall Configuration (UFW)

```bash
# Default policy
sudo ufw default deny incoming && sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow 22/tcp

# Allow OP‑Node & OP‑Geth ports
sudo ufw allow 8546   # op-node RPC
sudo ufw allow 9545   # op-geth RPC (Bedrock)
sudo ufw allow 8545   # l2geth RPC (legacy)
sudo ufw allow 3000   # Grafana dashboard

# Enable firewall
sudo ufw enable

# Verify
sudo ufw status verbose
```

> **Tip** Restrict the RPC ports to trusted IP ranges if the host has a public address.


## Clone the Optimism Docker setup

```bash
git clone https://github.com/smartcontracts/simple-optimism-node.git
cd simple-optimism-node
```

Create a working copy of the environment file:

```bash
sudo cp .env.template .env
```

## Configure the `.env` file (sample)

```ini
###############################################################################
#                                ↓ REQUIRED ↓                                 #
###############################################################################
# Network to run the node on ("op-mainnet" or "op-sepolia")
NETWORK_NAME=op-sepolia

# Node type ("full" or "archive") – archive ≈10× larger
NODE_TYPE=archive

###############################################################################
#                            ↓ REQUIRED (BEDROCK) ↓                           #
###############################################################################
# L1 RPC endpoint for op‑node (Bedrock)
OP_NODE__RPC_ENDPOINT=<l1-endpoint>

# L1 Beacon endpoint (for blob data)
OP_NODE__L1_BEACON=<l1-beacon-endpoint>

# RPC type used by op‑node – see repo README
OP_NODE__RPC_TYPE=basic

# Reference L2 node for health‑checks
HEALTHCHECK__REFERENCE_RPC_PROVIDER=https://sepolia.optimism.io

###############################################################################
#                            ↓ OPTIONAL (BEDROCK) ↓                           #
###############################################################################
# Provider to serve legacy RPC requests
OP_GETH__HISTORICAL_RPC=https://mainnet.optimism.io

# Force op‑geth to use a specific sync mode (leave blank = snap‑sync)
OP_GETH__SYNCMODE=

###############################################################################
#                                ↓ OPTIONAL ↓                                 #
###############################################################################
# Custom image tags – defaults to `latest`
IMAGE_TAG__OP_GETH=
IMAGE_TAG__OP_NODE=

# Exposed ports – override if the defaults clash
PORT__OP_GETH_HTTP=8545
PORT__OP_NODE_P2P=
PORT__OP_NODE_HTTP=
PORT__GRAFANA=3000
```

> See the repo README for **all** available variables: [https://github.com/smartcontracts/simple-optimism-node#mandatory-configurations](https://github.com/smartcontracts/simple-optimism-node#mandatory-configurations)

## Operating the Node

```bash
docker compose up -d --build
```

### View logs

```bash
docker compose logs <CONTAINER_NAME> -f --tail 10
```

### Monitoring

Run the helper script to gauge sync progress:

```bash
./progress.sh
```

Expected output example:

```text
Chain ID: 11155420
Sampling, please wait…
Blocks per minute: 30
Hours until sync completed: 42
```

**Grafana dashboard** – browse to [http://localhost:3000](http://localhost:3000) (or `http://<host‑IP>:3000`) and log in:

| Field    | Value      |
| -------- | ---------- |
| Username | `admin`    |
| Password | `optimism` |

The pre‑loaded *Simple Node Dashboard* visualises sync status and resource use.


## Query the Node

### Check sync status (Ethereum RPC)

```bash
curl -H "Content-Type: application/json" \
  -X POST --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' \
  http://localhost:8545
# Response when fully synced:
# {"jsonrpc":"2.0","id":1,"result":false}
```

### Check Optimism Bedrock sync status

```bash
curl -X POST -H "Content-Type: application/json" --data \
  '{"jsonrpc":"2.0","method":"optimism_syncStatus","params":[],"id":1}' \
  http://localhost:9545
```


## Useful Links

* **simple-optimism-node** repository – [https://github.com/smartcontracts/simple-optimism-node](https://github.com/smartcontracts/simple-optimism-node)
* **docker‑compose.yml** reference – [https://github.com/smartcontracts/simple-optimism-node/blob/main/docker-compose.yml](https://github.com/smartcontracts/simple-optimism-node/blob/main/docker-compose.yml)
* Optimism **Bedrock** docs – [https://docs.optimism.io](https://docs.optimism.io)


*Converted automatically from the [original GitBook page](https://docswip.infradao.com/docs-in-progress/optimism-sepolia/docker/op-geth) on May 11 2025.*
