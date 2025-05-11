# Rootstock Node Guide


## System Requirements

| Resource | Recommended  | Notes                                                  |
| -------- | ------------ | ------------------------------------------------------ |
| **CPU**  | 2 cores +    | Higher per‑core speed improves sync time               |
| **OS**   | Ubuntu 24.04 | Tested on LTS release                                  |
| **RAM**  | 8 GB +       |                                                        |
| **Disk** | ≥ 128 GB SSD | Rootstock main‑net archive size ≈ 132 GB on 2024‑09‑22 |


## Pre‑Requisites

Update, upgrade and clean the system, then install **ufw**:

```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt auto-remove -y
sudo apt install ufw -y
```

## Configure Firewall Settings

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential ports
sudo ufw allow 22/tcp   # SSH
sudo ufw allow 80       # HTTP (optional)
sudo ufw allow 443      # HTTPS (optional)
sudo ufw allow 4444     # Rootstock JSON‑RPC

sudo ufw enable
```

> **Security tip** If your node is on a public IP, restrict port 4444 to trusted addresses.


## Install Docker & Compose

### 1. Remove conflicting packages

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 \
           podman-docker containerd runc; do
  sudo apt-get remove $pkg
done
```

### 2. Add Docker’s official GPG key & repository

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

### 3. Install Docker Engine & Compose plugin

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Test
sudo docker run hello-world

docker compose version
```

## Set up the Rootstock Node

1. **Create a working directory**

   ```bash
   mkdir rootstock && cd rootstock
   ```

2. **Create the node configuration file** (`node.conf`):

   ```bash
   nano node.conf
   ```

   ```hocon
   blockchain.config.name = "main"

   database.dir = /var/lib/rsk/database/mainnet

   rpc {
     providers : {
       web: {
         cors: "localhost",
         http: {
           enabled: true,
           bind_address = "0.0.0.0",
           hosts = ["localhost"],
           port: 4444,
         },
         ws: {
           enabled: false,
           bind_address: "0.0.0.0",
           port: 4445,
         }
       }
     }
     modules = [
       { name: "eth",      version: "1.0", enabled: true },
       { name: "net",      version: "1.0", enabled: true },
       { name: "rpc",      version: "1.0", enabled: true },
       { name: "web3",     version: "1.0", enabled: true },
       { name: "evm",      version: "1.0", enabled: true },
       { name: "sco",      version: "1.0", enabled: false },
       { name: "txpool",   version: "1.0", enabled: true },
       { name: "debug",    version: "1.0", enabled: false },
       { name: "personal", version: "1.0", enabled: true }
     ]
   }
   ```

3. **Create the Docker Compose file** (`docker-compose.yml`):

   ```bash
   nano docker-compose.yml
   ```

   ```yaml
   services:
     rsk-node:
       image: rsksmart/rskj:ARROWHEAD-6.3.1
       container_name: rsk-node
       ports:
         - "5050:5050"   # Node p2p
         - "4444:4444"   # JSON‑RPC
       volumes:
         - rsk-data:/var/lib/rsk/.rsk
         - ./node.conf:/etc/rsk/node.conf
       restart: unless-stopped

   volumes:
     rsk-data:
   ```

   > **Note** If you prefer a custom data path, replace `rsk-data` with `./data`.

## Run the Node

```bash
docker compose up -d
```

## Monitor the Node

```bash
docker logs -f rsk-node
```

Initial sync logs look like this:

```text
2024-09-16 21:23:20 [INFO] blockHeight=1 309 986 imported IMPORTED_BEST
2024-09-16 21:23:20 [INFO] blockHeight=1 309 987 imported IMPORTED_BEST
```

## Query the Node

*Client version:*

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"web3_clientVersion","params":[],"id":67}' \
  http://localhost:4444
```

```
{"jsonrpc":"2.0","id":67,"result":"RskJ/6.3.1/Mac OS X/Java1.8/ARROWHEAD-202f1c5"}
```

```bash
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://localhost:4444
```

```
{"jsonrpc":"2.0","id":1,"result":"0x14144a"}
```

## References

* [Node Operators Overview – Rootstock Dev Portal](https://dev.rootstock.io/node-operators/)
* [Running Rootstock with Docker (rskj DockerHub)](https://hub.docker.com/r/rsksmart/rskj)
* [rskj GitHub repository](https://github.com/rsksmart/rskj)

