# Private-RPC-for-Aztec-Node-Run
## Create Your Own RPC For Aztec Node

## Hardware Requirements

### OS: Ubuntu 20.04 or later
### RAM: -16 GB	| CPU	84-6 cores | Disk 1000 Gb+ 
---

## Set Root first
```
sudo -i
```

## Install Dependecies

```bash
sudo apt-get update && sudo apt-get upgrade -y
```
```bash
sudo apt install curl iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev  -y
```

**Docker:**
```bash
sudo apt update -y && sudo apt upgrade -y
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y && sudo apt upgrade -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test Docker
sudo docker run hello-world

sudo systemctl enable docker
sudo systemctl restart docker
```

## Create Directories
```
mkdir -p /root/ethereum/execution
mkdir -p /root/ethereum/consensus
```

## Generate the JWT secret:
```
openssl rand -hex 32 > /root/ethereum/jwt.hex
```
**Verify JWT secrets exist**:
```
cat /root/ethereum/jwt.hex
```

## Configure `docker-compose.yml`
```
cd /root/ethereum
```
```
nano docker-compose.yml
```
* Replace the following code into your *docker-compose.yml* file:
```
services:
  geth:
    image: ethereum/client-go:stable
    container_name: geth
    network_mode: host
    restart: unless-stopped
    ports:
      - 30303:30303
      - 30303:30303/udp
      - 8545:8545
      - 8546:8546
      - 8551:8551
    volumes:
      - /root/ethereum/execution:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    command:
      - --sepolia
      - --http
      - --http.api=eth,net,web3
      - --http.addr=0.0.0.0
      - --authrpc.addr=0.0.0.0
      - --authrpc.vhosts=*
      - --authrpc.jwtsecret=/data/jwt.hex
      - --authrpc.port=8551
      - --syncmode=snap
      - --datadir=/data
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  prysm:
    image: gcr.io/prysmaticlabs/prysm/beacon-chain
    container_name: prysm
    network_mode: host
    restart: unless-stopped
    volumes:
      - /root/ethereum/consensus:/data
      - /root/ethereum/jwt.hex:/data/jwt.hex
    depends_on:
      - geth
    ports:
      - 4000:4000
      - 3500:3500
    command:
      - --sepolia
      - --accept-terms-of-use
      - --datadir=/data
      - --disable-monitoring
      - --rpc-host=0.0.0.0
      - --execution-endpoint=http://127.0.0.1:8551
      - --jwt-secret=/data/jwt.hex
      - --rpc-port=4000
      - --grpc-gateway-corsdomain=*
      - --grpc-gateway-host=0.0.0.0
      - --grpc-gateway-port=3500
      - --min-sync-peers=3
      - --checkpoint-sync-url=https://checkpoint-sync.sepolia.ethpandaops.io
      - --genesis-beacon-api-url=https://checkpoint-sync.sepolia.ethpandaops.io
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```


## Port conflict checking
### The setup uses ports 30303, 8545, 8546, 8551, 4000, and 3500. Make sure they’re free:

```
sudo apt update

sudo apt install net-tools
```

```
netstat -tuln | grep -E '30303|8545|8546|8551|4000|3500'
```


## Step 6. Run Geth & Prysm Nodes
* Start Geth & Prysm Nodes:
```
docker compose up -d
```

### Node Logs
```
# All lines of logs:
docker compose logs -f

# Last 100 lines of logs:
docker compose logs -fn 100
```
### Prysm will Sync Quickly & Geth will take time to sync, After checking logs you can use 'Ctrl + C' to stop the the logs.


## Setup VPS Firewall
```
sudo apt install ufw
```

```
 sudo ufw allow 22

sudo ufw allow ssh

sudo ufw enable
```

```
sudo ufw allow 8545/tcp    # Geth HTTP RPC

sudo ufw allow 3500/tcp    # Prysm HTTP API

sudo ufw allow 30303/tcp   # Geth P2P

sudo ufw allow 30303/udp   # Geth P2P
```
---


## Checking If Nodes Are Synced
### Geth (Sepolia)
```
curl -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"eth_syncing","params":[],"id":1}' http://localhost:8545
```
### ✅If fully synced👇
```json
{"jsonrpc":"2.0","id":1,"result":false}
```

### Prysm (Beacon)

```
curl http://localhost:3500/eth/v1/node/syncing
```
### ✅If fully synced👇
```json
{"data":{"head_slot":"12345","sync_distance":"0","is_syncing":false}}
```

## Step 8. Getting the RPC Endpoints
### Geth (Sepolia)

Geth provides an HTTP RPC endpoint for interacting with the execution layer of Ethereum. Based on `docker-compose.yml` setup, Geth exposes port `8545` for HTTP RPC. The endpoints are:
* Inside the VPS: `http://localhost:8545`
* Outside the VPS: `http://<your-vps-ip>:8545` (replace `<your-vps-ip>` with your VPS’s public IP address, e.g., `http://101.1.101.1:8545`)
* **Aztec Sequencer Execution RPC (Running by CLI)**: `http://<your-vps-ip>:8545`
* **Aztec Sequencer Execution RPC (Running by `docker-compose.yml`)**: `http://127.0.0.1:8545` or `http://localhost:8545`

### Beacon Node (Prysm)
Prysm, as the beacon node, offers an HTTP gateway on port `3500`. the endpoints are:
* Inside the VPS: `http://localhost:3500`
* Outside the VPS: `http://<your-vps-ip>:3500` (e.g., `http://203.0.113.5:3500`).
* **Aztec Sequencer Consensus Beacon RPC (Running by CLI)**: `http://<your-vps-ip>:3500`
* **Aztec Sequencer Consensus Beacon RPC (Running by `docker-compose.yml`)**: `http://127.0.0.1:3500` or `http://localhost:3500`

* ---

