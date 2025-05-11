# Moonriver Node Setup

## System Requirements

CPU: 8 Cores (Fastest per core speed)
OS: Ubuntu 24.04
RAM: 16GB
DISK: 2TB

### Run a tracing node
Geth's debug and txpool APIs and OpenEthereum's trace module provide non-standard RPC methods for getting a deeper insight into transaction processing. Supporting these RPC methods is important because many projects, such as The Graph, rely on them to index blockchain data.

To use the supported RPC methods, you need to run a tracing node. This guide covers the steps on how to setup and sync a tracing node on Moonbeam using Docker.


### Pre-Requisties
```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y
sudo apt install -y wget curl screen git ufw
```

### Setting up Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80
sudo ufw allow 443
```

### Enable Firewall
```bash
sudo ufw enable
```

### Install Docker

Run this command to remove any conflicting docker
```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

Add Docker's official GPG key:
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add the repository to ppt sources:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```
Install docker
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test docker is working
sudo docker run hello-world

#Install docker compose

sudo apt-get update
sudo apt-get install docker-compose-plugin

# Test the docker version
docker compose version
```

### Make Database directory and set necessary permissions
```bash
mkdir /var/lib/moonriver-data/

sudo chown -R $(id -u):$(id -g) /var/lib/moonriver-data
```

### Create a directory for WASM overrides:
```bash
mkdir /var/lib/moonriver-data/moonriver

sudo chown -R $(id -u):$(id -g) /var/lib/moonriver-data/moonriver
```

### Create Docker compose
Instead of the standard moonbeamfoundation/moonbeam docker image, you will use the moonbeamfoundation/moonbeam-tracing image. 
The latest supported version can be found on the Docker Hub for the moonbeam-tracing image from these repos: https://hub.docker.com/r/moonbeamfoundation/moonbeam-tracing/tags

```bash
sudo nano docker-compose.yml
```

```bash
version: '3.9'


networks:
  monitor-net:
    driver: bridge
  
volumes:
  moonriver-data: {}

services:
  moonriver:
    image: moonbeamfoundation/moonbeam-tracing:v0.42.1-3401-latest
    user: root
    container_name: moonriver
    volumes:
      - "/var/lib/moonriver-data:/data"
    restart: unless-stopped
    command:
      - --base-path=/data
      - --chain=moonriver
      - --name="babayaga"
      - --rpc-port=9944
      - --rpc-cors=all
      - --unsafe-rpc-external
      - --state-pruning=archive
      - --trie-cache-size=1073741824
      - --db-cache=64000
      - --ethapi=debug,trace,txpool
      - --wasm-runtime-overrides=/moonriver/moonriver-substitutes-tracing
      - --runtime-cache-size=64
      - --
      - --name=" (Embedded Relay)"
    expose:
      - 9944 # rpc + ws parachain
      - 9945 # rpc + ws relay chain
      - 30333 # p2p parachain
      - 30334 # p2p relay chain
      - 9615 # prometheus parachain
      - 9616 # prometheus relay chain
    ports:
      - "9944:9944" # rpc + ws parachain
      - "9945:9945" # rpc + ws relay chain
      - "30333:30333"
      - "30334:30334"
    networks:
      - monitor-net
```

### Monitor logs for errors
```bash
docker logs moonriver -f --tail 100
```

The expected output after successful launch should look like this:
```
2025-01-26 11:58:08 [🌗] Found wasm override. version=moonriver-3102 (moonriver-0.tx2.au3) file=/moonbeam/moonriver-substitutes-tracing/moonriver-runtime-3102-substitute-tracing.wasm
2025-01-26 11:58:09 [🌗] Found wasm override. version=moonriver-1002 (moonriver-0.tx2.au3) file=/moonbeam/moonriver-substitutes-tracing/moonriver-runtime-1002-substitute-tracing.wasm
2025-01-26 11:58:09 [Relaychain] 🔨 Initializing Genesis block/state (state: 0xb000…ef6b, header-hash: 0xb0a8…dafe)    
2025-01-26 11:58:09 [Relaychain] 👴 Loading GRANDPA authority set from genesis on what appears to be first startup.    
2025-01-26 11:58:09 [Relaychain] 👶 Creating empty BABE epoch changes on what appears to be first startup.    
2025-01-26 11:58:09 [Relaychain] 🏷  Local node identity is: 12D3KooWNcthaQ9c8SByj2A5uJdtPvU5bJaGNJwVzGDDkdNkn9UW    
2025-01-26 11:58:09 [Relaychain] Running libp2p network backend    
2025-01-26 11:58:09 [Relaychain] 💻 Operating system: linux    
2025-01-26 11:58:09 [Relaychain] 💻 CPU architecture: x86_64    
2025-01-26 11:58:09 [Relaychain] 💻 Target environment: gnu  
```

```bash
2025-01-28 21:21:49 [🌗] ⚙️  Syncing 175.0 bps, target=#10068541 (38 peers), best: #9577251 (0x28f7…734e), finalized #3716815 (0x28cc…7e57), ⬇ 3.5MiB/s ⬆ 0.5kiB/s    
2025-01-28 21:21:50 [Relaychain] ⚙️  Syncing 44.0 bps, target=#26848175 (41 peers), best: #16826836 (0x96a2…eabe), finalized #16826616 (0xba2c…df4e), ⬇ 1.6MiB/s ⬆ 140.9kiB/s    
2025-01-28 21:21:54 [🌗] ⚙️  Syncing 214.8 bps, target=#10068541 (38 peers), best: #9578325 (0xfe6c…a27c), finalized #3716959 (0xff38…fd86), ⬇ 4.0MiB/s ⬆ 1.6kiB/s    
2025-01-28 21:21:55 [Relaychain] ⚙️  Syncing 36.6 bps, target=#26848176 (41 peers), best: #16827019 (0x205c…f14d), finalized #16826912 (0x05d9…fbde), ⬇ 1.1MiB/s ⬆ 110.8kiB/s    
2025-01-28 21:21:59 [🌗] ⚙️  Syncing 140.6 bps, target=#10065658 (39 peers), best: #9579028 (0xc181…c7a0), finalized #3716959 (0xff38…fd86), ⬇ 2.7MiB/s ⬆ 1.1kiB/s    
2025-01-28 21:22:00 [Relaychain] ⚙️  Syncing 43.4 bps, target=#26848176 (41 peers), best: #16827236 (0xb843…e9bd), finalized #16826912 (0x05d9…fbde), ⬇ 1.8MiB/s ⬆ 215.0kiB/s 
```

### Test Moonriver RPC
You can call the JSON-RPC API methods to confirm the node is running. For example, call eth_syncing to return the synchronization status. It will return the starting, current, and highest block, or false if not synchronizing (or if the head of the chain has been reached)
Replace https://{YOUR_DOMAIN} with actual domain name or host ip:  
```bash
curl https://{YOUR_DOMAIN} \
        -X POST \
        -H "Content-Type: application/json" \
        -d '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}'
```
Expected output:
```bash
{"jsonrpc":"2.0","result":{"startingBlock":"0x0","currentBlock":"0x920bb1","highestBlock":"0x99a23d","warpChunksAmount":null,"warpChunksProcessed":null},"id":1}
```
### References
Moonbeam Tracing Node Docs: https://docs.moonbeam.network/node-operators/networks/tracing-node/

Moonbeam Full Node Docs: https://docs.moonbeam.network/node-operators/networks/run-a-node/docker/

Moonbeam Tracing Node Docker: https://hub.docker.com/r/moonbeamfoundation/moonbeam-tracing/tags