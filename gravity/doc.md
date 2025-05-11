# Gravity Node Setup

## System requirements

| Resource | Recommendation                                             |
| -------- | ---------------------------------------------------------- |
| **CPU**  | 2 – 4 cores (highest per‑core speed)                       |
| **OS**   | Debian 12 / Ubuntu 22.04                                   |
| **RAM**  | 8 – 16 GB                                                  |
| **Disk** | 286 GB SSD  *(node size observed at 268 GB on 2025‑01‑28)* |

## Offchain Labs ⛓️

See the [official Nitro documentation](https://docs.arbitrum.io/).

## Pre‑requisites

```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y

sudo apt install -y git make wget gcc pkg-config libusb-1.0-0-dev \
  libudev-dev jq gcc g++ curl libssl-dev screen apache2-utils \
  build-essential pkg-config
```

## Setting up the firewall (UFW)

1. **Set default policies**

   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   ```

2. **Allow required ports**

   ```bash
   # SSH
   sudo ufw allow 22/tcp

   # Gravity JSON‑RPC
   sudo ufw allow 8546
   sudo ufw allow 8547
   ```

   > *Tip  Avoid exposing the RPC ports to unknown IP addresses.*

3. **Enable the firewall**

   ```bash
   sudo ufw enable
   ```

## Install Docker (Ubuntu/Debian)

Remove any conflicting packages:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 \
           podman-docker containerd runc; do
  sudo apt-get remove $pkg
done
```

### Add the Docker GPG key

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Add the Docker repository

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

### Install Docker Engine & Compose

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Test
sudo docker run hello-world
```

## Install Go 1.23.5

```bash
wget https://go.dev/dl/go1.23.5.linux-amd64.tar.gz && \
  sudo rm -rf /usr/local/go && \
  sudo tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz && \
  rm go1.23.5.linux-amd64.tar.gz
```

Add Go to your `PATH` (e.g. in `~/.profile`):

```bash
export PATH=$PATH:/usr/local/go/bin
```

Verify:

```bash
go version
```

## Build **Nitro** with Docker

```bash
git clone --recurse-submodules https://github.com/OffchainLabs/nitro
cd nitro
docker build -t nitro .
```

### Copy the Nitro binary out of the image

```bash
docker cp $(docker run -it -d nitro):/usr/local/bin/nitro /root/nitro/build/bin/
/root/nitro/build/bin/nitro -h   # test
```

## Create a systemd service (Gravity Mainnet)

1. **Create data directories**

   ```bash
   mkdir -p /root/.gravity/share/nitro/datadir
   ```

2. **Create the service file**

   ```bash
   sudo nano /etc/systemd/system/gravity.service
   ```

   Paste the following (adjust `<ethereum_mainnet_rpc>` and `<ethereum_beacon_chain_rpc>`):

   ```ini
   [Unit]
   Description=Gravity Alpha Mainnet Node
   After=network.target

   [Service]
   Type=simple
   Restart=on-failure
   RestartSec=5
   TimeoutSec=900
   User=root
   Nice=0
   LimitNOFILE=200000
   ExecStart=/root/nitro/build/bin/nitro \
     --parent-chain.connection.url=<ethereum_mainnet_rpc> \
     --persistent.chain=/root/.gravity/share/nitro/datadir/ \
     --persistent.global-config=/root/.gravity/share/nitro/ \
     --chain.id=1625 \
     --chain.name=conduit-orbit-deployer \
     --chain.info-json='[{"chain-id":1625,"parent-chain-id":1,"chain-name":"conduit-orbit-deployer", ...}]' \
     --http.api=net,web3,eth \
     --http.corsdomain=* \
     --http.addr=0.0.0.0 \
     --http.port=8547 \
     --http.vhosts=* \
     --node.data-availability.enable \
     --node.data-availability.rest-aggregator.enable \
     --node.data-availability.rest-aggregator.urls=https://das-gravity-mainnet-0.t.conduit.xyz \
     --execution.forwarding-target=https://rpc.gravity.xyz \
     --node.feed.input.url=wss://relay-gravity-mainnet-0.t.conduit.xyz \
     --parent-chain.blob-client.beacon-url=<ethereum_beacon_chain_rpc>
   KillSignal=SIGHUP

   [Install]
   WantedBy=multi-user.target
   ```

3. **Enable & manage the service**

   ```bash
   sudo systemctl daemon-reload      # reload unit files
   sudo systemctl enable gravity.service
   sudo systemctl start gravity.service
   # To stop
   sudo systemctl stop gravity.service
   ```

4. **Follow logs**

   ```bash
   journalctl -f -u gravity.service
   ```

   Sample output while syncing:

   ```text
   INFO [01-28|23:32:36.178] created block  l2Block=36,676,753 l2BlockHash=f84024..8ee71c
   INFO [01-28|23:32:37.179] created block  l2Block=36,676,756 l2BlockHash=1e0d92..5f8afb
   INFO [01-28|23:32:38.180] created block  l2Block=36,676,760 l2BlockHash=e097e7..079758
   ```

   > Expect your node to reach chain head within about a week.

## References

* [Running an Ethereum archive node (Arbitrum Docs)](https://docs.arbitrum.io/node-running/how-tos/running-an-archive-node)
* [Run a Gravity Alpha Mainnet (L2) Node – Gravity Docs](https://docs.gravity.xyz)
* [Running an Orbit node (Arbitrum Docs)](https://docs.arbitrum.io)
* [Run an Arbitrum Orbit node – Conduit](https://docs.conduit.xyz)
