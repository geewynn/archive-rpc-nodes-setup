## Optimism Sepolia Node Setup

## System Requirements

| Resource | Recommended      |                                  |
| -------- | ---------------- | --------------------------------------- |
| **CPU**  | 8 cores 
| **OS**   | Ubuntu 24.04 LTS |
| **RAM**  | 16 GB+           |                                         |
| **Disk** | 2 TB SSD         


```text
In this guide, we cover docker installation of op-erigon and op-nodeto facilitate the node's synchronization on Sepolia Testnet Network. This method is expected to sync an archive node successfully in days or weeks using the snapshot provided by Testinprod-io

Before you start, make sure that you have your own synced Ethereum Sepolia L1 RPC URL and L1 Consensus Layer Beacon endpoint (e.g. Lighthouse Sepolia) ready
```

## Pre requisites
```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt autoremove -y
​
sudo apt install -y wget curl screen git ufw zstd
```

## Setting up Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp
sudo ufw allow 80
sudo ufw allow 443
```

## Enable Firewall
```bash
sudo ufw enable
```

## Install Docker

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test docker is working
sudo docker run hello-world

#Install docker compose

sudo apt-get update
sudo apt-get install docker-compose-plugin

# Test the docker version
docker compose version
```

## Create .env file
```bash
sudo nano .env
```

## Create JWT secret file

```bash
mkdir -p /root/data/optimism-sepolia/op-erigon/ && cd /root/data/optimism-sepolia/op-erigon/

openssl rand -hex 32 | tr -d "\n" > "./jwt.hex"
```

### Optional/Recommended: Download OptimismSnapshot
To sync from a snapshot, visit Testinprod-io Node Snapshots page to find a latest OptimismSepolia archive for Op-Erigon: https://snapshot.testinprod.io/.

As downloading a snapshot takes some time it is good idea to run it in a screen session

```bash
screen -S erigon
```

Use aria2c to download the most recent Optimism Sepolia Op-Erigon Archive Snapshot

```bash
cd /root/optimism-sepolia

aria2c --file-allocation=none -c -x 15 -s 15 https://datadirs.testinprod.io/op-sepolia-db-24726201.zst
```
press ctrl+A and D to return to previous screen and continue installation

```bash
screen -r erigon #will bring you back to monitor downloading progress
```
You'll need to extract the downloaded snapshot and move its contents to the op-erigon-data directory, where Docker stores persistent data.

```bash
# check sha256sum
sha256sum op-sepolia-db-24726201.zst
# decompress first
zstd --decompress op-sepolia-db-24726201.zst -o mdbx.dat
```
If you initially tried to sync the node from scratch and are now trying with a snapshot make sure to empty the destination directory first:

```bash
cd /var/lib/docker/volumes/op-sepolia-erigon_op-erigon_data/_data 

ls 

#if directory isn't empty remove contents of chaindata directory

rm -rf ~/chaindata/*

mv /root/optimism-sepolia/mdbx.dat /var/lib/docker/volumes/op-sepolia-erigon_op-erigon_data/_data/chaindata

cd /var/lib/docker/volumes/op-sepolia-erigon_op-erigon_data/_data/chaindata

ls #to check if archive has moved properly
```

```bash
docker volume create op-sepolia-erigon_op-erigon_data

cd /var/lib/docker/volumes/op-sepolia-erigon_op-erigon_data/_data

mv /root/optimism-sepolia/mdbx.dat /var/lib/docker/volumes/op-sepolia-erigon_op-erigon_data/_data/

ls
```
### Launch Optimism Sepolia

```bash
cd /root/optimism-sepolia

sudo nano docker-compose.yml
```