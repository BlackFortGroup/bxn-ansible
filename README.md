# Besu Blockchain Deployment with Ansible

This repository contains Ansible playbooks for deploying and managing a Hyperledger Besu blockchain network. The setup supports both `testnet` and `mainnet` environments.

## Playbooks

- `mainnet-playbook.yaml` - Deploys a Besu mainnet node.

- `testnet-playbook.yaml` - Deploys a Besu testnet node.

## Inventory Files

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

## Ansible Roles

The playbooks utilize the following roles:

- **geerlingguy.docker** - Installs Docker

- **buluma.fail2ban** - Installs Fail2Ban for security

- **blackfort_besu** - Configures and deploys Besu

## Role: `blackfort_besu`

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

## Deployment Instructions

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

### Variables

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

