1. [General Information](#1-general-information)
2. [Playbooks](#2-playbooks)
3. [Inventory Files](#3-inventory-files)
4. [Ansible Roles](#4-ansible-roles)
5. [Role: `blackfort_besu`](#5-role-blackfort_besu)
6. [Deployment Instructions](#6-deployment-instructions)
7. [Variables](#7-variables)
8. [Running a Besu Node in an Existing Network using Ansible. Recommended](#8-running-a-besu-node-in-an-existing-network-using-ansible-recommended)
9. [Running a Besu Node in an Existing Network using Docker Compose](#9-running-a-besu-node-in-an-existing-network-using-docker-compose)
10. [Adding a Validator to a Besu Network](#10-adding-a-validator-to-a-besu-network)

# Besu Blockchain Deployment with Ansible

This repository contains Ansible playbooks for deploying and managing a Hyperledger Besu blockchain network. The setup supports both `testnet` and `mainnet` environments.
---
## 1. General information

Chain ID `testnet` **4888**

Chain ID `mainnet` **488**

### Main API URLs

https://rpc.blackfort.network/mainnet/rpc - node reader `mainnet`

https://rpc.blackfort.network/mainnet/validator-1 - validator `mainnet`

https://rpc.blackfort.network/testnet/rpc - node reader `testnet`

https://rpc.blackfort.network/testnet/validator-1 - validator `testnet`
---
## 2. Playbooks

- `mainnet-playbook.yaml` - Deploys a Besu mainnet node.

- `testnet-playbook.yaml` - Deploys a Besu testnet node.
---
## 3. Inventory Files

Define the network structure in the following inventory files:

- `testnet-inventory.ini`

- `mainnet-inventory.ini`

### Inventory Groups

The network is structured into the following groups:

- **[validators]** - Validator nodes

- **[bootnodes]** - Boot nodes

- **[members]** - Network participants

- **[rpc]** - RPC nodes

Example:

```ini
[validators]
validator-1 ansible_host=203.0.113.11
validator-2 ansible_host=203.0.113.12
validator-3 ansible_host=203.0.113.13
validator-4 ansible_host=203.0.113.14
validator-5 ansible_host=203.0.113.15

[bootnodes]
bootnode-1 ansible_host=203.0.113.16

[members]
member-1 ansible_host=203.0.113.17

[rpc]
rpc-1 ansible_host=203.0.113.18

[all:vars]
ansible_python_interpreter=/usr/bin/python3
blackfort_besu_node="{{ blackfort_besu_network }}-{{ inventory_hostname }}"
blackfort_besu_rpc_http_authentication_enabled="true"
```
---
## 4. Ansible Roles

The playbooks utilize the following roles:

- **geerlingguy.docker** - Installs Docker

- **buluma.fail2ban** - Installs Fail2Ban for security

- **blackfort_besu** - Configures and deploys Besu
---
## 5. Role: `blackfort_besu`

### Responsibilities

- Configures Besu nodes

- Sets up necessary directories

- Copies configuration files (e.g., `config.toml`, `genesis.json`)

- Deploys validator keys

- Imports blockchain state (if enabled)

- Starts Besu nodes via Docker Compose

### Required Files

Located in `roles/blackfort_besu/files/`:

- **exportblock-mainnet.bin** - Blockchain state for import

- **genesis-mainnet.json** - Genesis file for mainnet

- **genesis-testnet.json** - Genesis file for testnet

- **publicKey-mainnet.pem** - JWT public key for mainnet

- **publicKey-testnet.pem** - JWT public key for testnet

- **static-nodes.json** - Static node connections (initially empty)

### Validator and Node Keys

#### Validator Keys

Each **validator node** requires a pre-generated private and public key pair (`nodekey` and `nodekey.pub`).  
These keys are stored in an encrypted file using **Ansible Vault**. The file is named:

- `key_validators-testnet.yml` – For the **Testnet**.
- `key_validators-mainnet.yml` – For the **Mainnet** (if applicable).

Example contents of the `key_validators-testnet.yml` file (encrypted):

```yaml
key_validators:
  - name: validator-1
    nodekey: "private_key_content_here"
    nodekey_pub: "public_key_content_here"
  - name: validator-2
    nodekey: "private_key_content_here"
    nodekey_pub: "public_key_content_here"
  # Add all validators
```

The validator keys are only deployed to hosts listed in the `[validators]` group in the inventory.

#### Other Nodes

For **non-validator nodes** (e.g., bootnodes, members, rpc), the key pair (`nodekey` and `nodekey.pub`) is **automatically generated** when the playbook is executed for the first time.  
These keys are stored in:

```swift
/opt/besu/keys/nodekey /opt/besu/keys/nodekey.pub
```
---
## 6. Deployment Instructions

### Prerequisites

Ensure you have:

- Ansible installed (`ansible-playbook` command available)

- SSH access to the target nodes

### Running the Playbook

#### Deploy to Testnet

```bash
ansible-playbook -i testnet-inventory.ini testnet-playbook.yaml --ask-vault-pass
```

#### Deploy to Mainnet

```bash
ansible-playbook -i mainnet-inventory.ini mainnet-playbook.yaml --ask-vault-pass
```

#### Additional Options

When running the playbooks, you can pass extra options using the `-e` flag in the `ansible-playbook` command:

- **install_fail2ban** – Installs and configures Fail2Ban to protect the node.  
  Default: `false`.  
  Enable it with:

```bash
ansible-playbook -i testnet-inventory.ini testnet-playbook.yaml -e "install_fail2ban=true"
```

- **install_docker** – Installs Docker and Docker Compose.  
  Default: `false`.  
  Enable it with:

```bash
ansible-playbook -i testnet-inventory.ini testnet-playbook.yaml -e "install_docker=true"
```

- **blackfort_besu_blocks_import** – Enables blockchain data import from the `exportblock-mainnet.bin` file.  
  Default: `"false"`.  
  Enable it with:

```bash
ansible-playbook -i mainnet-inventory.ini mainnet-playbook.yaml -e "blackfort_besu_blocks_import=true"
```

These variables can also be combined:

```bash
ansible-playbook -i testnet-inventory.ini testnet-playbook.yaml -e "install_fail2ban=true install_docker=true blackfort_besu_blocks_import=true"
```
---
## 7. Variables

| Variable                                                     | Description                                  | Default Value                                                                                        |
| ------------------------------------------------------------ | -------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `blackfort_besu_network`                                     | Network type (`testnet` or `mainnet`).       | `testnet`                                                                                            |
| `blackfort_besu_hostname`                                    | Optional hostname override.                  | `""`                                                                                                 |
| `blackfort_besu_version`                                     | Besu version.                                | `25.2.0`                                                                                             |
| `blackfort_besu_image`                                       | Docker image for Besu.                       | `hyperledger/besu`                                                                                   |
| `blackfort_besu_keygen_image`                                | Image used for key generation.               | `consensys/quorum-k8s-hooks:qgt-0.2.20`                                                              |
| `blackfort_besu_base_dir`                                    | Base directory for Besu data.                | `/opt/besu`                                                                                          |
| `blackfort_besu_config_dir`                                  | Configuration directory.                     | `/opt/besu/config`                                                                                   |
| `blackfort_besu_data_dir`                                    | Data directory.                              | `/opt/besu/data`                                                                                     |
| `blackfort_besu_keys_dir`                                    | Keys directory.                              | `/opt/besu/keys`                                                                                     |
| `blackfort_besu_log_level`                                   | Logging level.                               | `INFO`                                                                                               |
| `blackfort_besu_p2p_enabled`                                 | Enable P2P networking.                       | `true`                                                                                               |
| `blackfort_besu_discovery_enabled`                           | Enable P2P discovery.                        | `true`                                                                                               |
| `blackfort_besu_static_nodes_file`                           | Path to `static-nodes.json`.                 | `/etc/besu/static-nodes.json`                                                                        |
| `blackfort_besu_p2p_host`                                    | Host for P2P interface.                      | `0.0.0.0`                                                                                            |
| `blackfort_besu_p2p_port`                                    | P2P port.                                    | `30303`                                                                                              |
| `blackfort_besu_max_peers`                                   | Max number of peers.                         | `25`                                                                                                 |
| `blackfort_besu_min_gas`                                     | Minimum gas price.                           | `1000000000000`                                                                                      |
| `blackfort_besu_tx_pool_min_gas_price`                       | Transaction pool minimum gas price.          | `1000000000000`                                                                                      |
| `blackfort_besu_host_allowlist`                              | Allowed hosts for HTTP RPC.                  | `["*"]`                                                                                              |
| `blackfort_besu_rpc_http_enabled`                            | Enable HTTP RPC.                             | `true`                                                                                               |
| `blackfort_besu_rpc_http_host`                               | Host for HTTP RPC.                           | `0.0.0.0`                                                                                            |
| `blackfort_besu_rpc_http_port`                               | HTTP RPC port.                               | `8545`                                                                                               |
| `blackfort_besu_rpc_http_api`                                | Enabled APIs for HTTP RPC.                   | `["DEBUG", "ETH", "ADMIN", "WEB3", "IBFT", "NET", "TRACE", "EEA", "PRIV", "QBFT", "PERM", "TXPOOL"]` |
| `blackfort_besu_rpc_http_cors_origins`                       | CORS origins for HTTP RPC.                   | `["all"]`                                                                                            |
| `blackfort_besu_rpc_http_authentication_enabled`             | Enable JWT authentication for HTTP RPC.      | `false`                                                                                              |
| `blackfort_besu_rpc_http_authentication_jwt_public_key_file` | JWT public key file for HTTP RPC.            | `/keys/publicKey.pem`                                                                                |
| `blackfort_besu_rpc_http_max_active_connections`             | Max active HTTP RPC connections.             | `80`                                                                                                 |
| `blackfort_besu_revert_reason_enabled`                       | Enable revert reason output.                 | `true`                                                                                               |
| `blackfort_besu_rpc_ws_enabled`                              | Enable WebSocket RPC.                        | `false`                                                                                              |
| `blackfort_besu_rpc_ws_host`                                 | Host for WebSocket RPC.                      | `0.0.0.0`                                                                                            |
| `blackfort_besu_rpc_ws_port`                                 | WebSocket RPC port.                          | `8546`                                                                                               |
| `blackfort_besu_rpc_ws_api`                                  | Enabled APIs for WebSocket RPC.              | Same as HTTP RPC                                                                                     |
| `blackfort_besu_rpc_ws_authentication_enabled`               | Enable JWT authentication for WebSocket RPC. | `false`                                                                                              |
| `blackfort_besu_metrics_enabled`                             | Enable Prometheus metrics.                   | `true`                                                                                               |
| `blackfort_besu_metrics_host`                                | Host for metrics.                            | `0.0.0.0`                                                                                            |
| `blackfort_besu_metrics_port`                                | Metrics port.                                | `9545`                                                                                               |
| `blackfort_besu_xdns_enabled`                                | Enable experimental DNS features.            | `true`                                                                                               |
| `blackfort_besu_blocks_import`                               | Enable importing blockchain state from file. | `false`                                                                                              |
| `blackfort_besu_bonsai_historical_block_limit`               | Bonsai historical block limit.               | 512                                                                                                  |
| `blackfort_besu_besu_opts`                                   | Additional Besu options.                     | `""`                                                                                                 |
| `blackfort_besu_sync_mode`                                   | Synchronization mode.                        | `FAST`                                                                                               |
| `blackfort_besu_data_storage_format`                         | Data storage format.                         | `BONSAI`                                                                                             |
---
## 8. Running a Besu Node in an Existing Network using Ansible. Recommended

This guide explains how to deploy a Hyperledger Besu node into an existing network using Ansible.

### Prerequisites

Before proceeding, ensure you have the following:

1. Ansible installed on your control machine. If not installed, follow the [official Ansible installation guide](https://docs.ansible.com/ansible/latest/installation_guide/index.html).
2. SSH access to the target machine where Besu will run.
3. Proper network connectivity between your control machine and the target node.
4. Installation has been tested on Ubuntu 24.04.

### Clone the Repository

Begin by cloning the repository containing the necessary configuration files:

```bash
git clone https://github.com/BlackFortGroup/bxn-ansible
```

### Deployment

#### Inventory File Setup

Create or edit the `new-nodes-inventory.ini` file to include your new node(s) in the `new_nodes` group:

```ini
[new_nodes]
pp-node-1 ansible_host=203.0.113.19
```

Replace 203.0.113.19 with the actual IP address of your new node.

#### For Testnet

To add a new node to the test network, run:

```bash
ansible-playbook -i new-nodes-inventory.ini add-new-node-testnet.yaml -e "install_docker=true"
```

#### For Mainnet

To add a new node to the main network, run:

```bash
ansible-playbook -i new-nodes-inventory.ini add-new-node-mainnet.yaml -e "install_docker=true"
```
---
## 9. Running a Besu Node in an Existing Network using Docker Compose

To successfully run a Hyperledger Besu node in an existing network using Docker Compose v2, follow the steps outlined below.

### Prerequisites

Ensure that both Docker and Docker Compose are installed on your system.

- **Docker**: Installation instructions are available on the official website:
  
  - For Ubuntu: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

- **Docker Compose**: Installation guide can be found here:
  
  - For Linux: [Install Docker Compose on Linux](https://docs.docker.com/compose/install/linux/)

### Clone the Repository

Begin by cloning the repository containing the necessary configuration files:

```bash
git clone https://github.com/BlackFortGroup/bxn-ansible
cd bxn-ansible/docker
```

### Start the Node

- **For Mainnet**:

```bash
HOSTNAME=$HOSTNAME docker compose -f docker-compose-mainnet.yml up -d
```

- **For Testnet**:

The process for starting a node on the testnet is analogous to the mainnet.

```bash
HOSTNAME=$HOSTNAME docker compose -f docker-compose-testnet.yml up -d
```

### Docker Compose Configuration (`docker-compose-mainnet.yml`)

The `docker-compose-mainnet.yml` file defines the configuration for running a Besu node in the Mainnet network. Below is the content of the file with explanations:

```yaml
services:
  bxn-node-mainnet:
    container_name: bxn-node-mainnet
    image: hyperledger/besu:latest
    restart: always
    ports:
      - "8545:8545"
      - "30303:30303/tcp"
      - "30303:30303/udp"
    command: [ "--config-file=/etc/besu/config.toml",
               "--identity=${HOSTNAME}" ]
    volumes:
      - ../roles/blackfort_besu/files/genesis-mainnet.json:/etc/besu/genesis.json:ro
      - ./static-nodes-mainnet.json:/etc/besu/static-nodes.json:ro
      - ./config.toml:/etc/besu/config.toml:ro
      - bxn-node-mainnet-data:/opt/besu/database

volumes:
  bxn-node-mainnet-data:
```

#### Explanation of Ports and Volumes

- **Ports**:
  
  - `8545:8545`: Maps port 8545 in the container to port 8545 on the host, enabling access to the JSON-RPC API for interacting with the node.
  
  - `30303:30303/tcp` and `30303:30303/udp`: Map the container's TCP and UDP ports 30303 to the host, facilitating peer-to-peer communication with other nodes.

- **Volumes**:
  
  - `../roles/blackfort_besu/files/genesis-mainnet.json:/etc/besu/genesis.json:ro`: Mounts the genesis file into the container in read-only mode, defining the initial state of the blockchain.
  
  - `./static-nodes-mainnet.json:/etc/besu/static-nodes.json:ro`: Mounts the static nodes file in read-only mode, specifying peers for the node to connect with.
  
  - `./config.toml:/etc/besu/config.toml:ro`: Mounts the main configuration file in read-only mode, containing various node settings.
  
  - `bxn-node-mainnet-data:/opt/besu/database`: Defines a named volume for storing blockchain data, ensuring data persistence across container restarts.

### Configuration File (`config.toml`)

The `config.toml` file specifies the node's runtime settings. Below are the key options with their purposes:

```toml
data-path="/opt/besu/database"
genesis-file="/etc/besu/genesis.json"
static-nodes-file="/etc/besu/static-nodes.json"
rpc-http-enabled=true
Xdns-enabled=true
Xdns-update-enabled=true
nat-method="DOCKER"
rpc-http-api=["ETH","NET","WEB3"]
sync-mode="FAST"
data-storage-format="BONSAI"
```

#### Explanation of Options

- `data-path`: Specifies the directory for storing blockchain data.

- `genesis-file`: Indicates the path to the genesis file, which defines the initial state and parameters of the blockchain.

- `static-nodes-file`: Points to the file containing a list of static nodes for the node to connect with.

- `rpc-http-enabled`: Enables the HTTP API for remote procedure calls, allowing external applications to interact with the node.

- `Xdns-enabled`: Activates the experimental DNS feature.

- `Xdns-update-enabled`: Permits updates to the experimental DNS feature.

- `nat-method`: Sets the Network Address Translation (NAT) method. The `DOCKER` value indicates that the node is operating within a Docker container, facilitating proper NAT handling.

- `rpc-http-api`: APIs exposed over HTTP RPC. The available API options are: ADMIN, CLIQUE, DEBUG, EEA, ETH, IBFT, MINER, NET, PERM, PLUGINS, PRIV, QBFT, TRACE, TXPOOL, and WEB3.

- `sync-mode`: Synchronization mode, possible values are FULL

- `data-storage-format`: Data storage format (options: 'FOREST', 'BONSAI')

### Accessing the JSON-RPC API

With `rpc-http-enabled` set to `true`, the JSON-RPC API is accessible at `http://localhost:8545`. This API allows for interaction with the Ethereum network through various methods.

#### Example Usage with `curl`

- **Retrieve the Current Block Number**:
  
  To get the current block number, use the `eth_blockNumber` method:

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' http://localhost:8545
```

This command returns the current block number in hexadecimal format.

- **Retrieve the Number of Peers**:
  
  To find out how many peers are connected to your node, use the `net_peerCount`

```bash
curl -X POST --data '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}' http://localhost:8545
```

This command returns the number of connected peers in hexadecimal format.

For a comprehensive list of available JSON-RPC methods and their usage, refer to the [Welcome | Besu documentation](https://besu.hyperledger.org/stable/public-networks/reference/api)

### Verifying the Node

To check if the node is running:

```bash
docker ps
```

To view logs for debugging:

```bash
docker compose -f docker-compose-mainnet.yml logs -f
```

### Stopping the Node

To stop the node, execute:

```bash
docker compose -f docker-compose-mainnet.yml down
```
---
## 10. Adding a Validator to a Besu Network

This guide explains how to add a validator to an existing Hyperledger Besu network.

### Prerequisites

1. A running Besu node deployed using one of these methods:
   - [Running a Besu Node in an Existing Network using Ansible. Recommended](#8-running-a-besu-node-in-an-existing-network-using-ansible-recommended)
   - [Running a Besu Node in an Existing Network using Docker Compose](#9-running-a-besu-node-in-an-existing-network-using-docker-compose)

2. The node must be fully synchronized with the network
   - Check for Full Synchronization
   
     The node has completed synchronization when the log contains the message:
     ```Node is in sync, enabling transaction handling```
    ```bash
    docker logs <besu_container_name> | grep "Node is in sync, enabling transaction handling" 
    ```
   - Monitor Synchronization Progress
   
     During synchronization, the node imports blocks and logs entries in the format:
     ```Imported empty block #<block_number> ...```

     _Example log entry:_

     ```EthScheduler-Workers-0 | INFO  | PersistBlockTask | Imported empty block #529,913 / 0 tx / 0 om / 0 (0.0%) gas / (0x4e46...d3a) in 0.000s. Peers: 8```

    ```bash
    docker logs <besu_container_name> | grep "PersistBlockTask"  
    ```

### Get Node Address

```bash
docker exec <besu_container_name> besu public-key export-address
```

Example output:

```
0xfe3b557e8fb62b89f4916b721be55ceb828dbd73
```

### Submit Validator Request

1. Send your node address to the network administrator for validator registration. Provide:
   * The address obtained from the previous step
   * Your node's enode URL (optional)
   * Any additional information required by the network's governance

2. The administrator will:
   * Submit your address to the validator management contract
   * Coordinate with existing validators for approval
   * Notify you when the registration is complete