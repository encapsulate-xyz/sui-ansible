# Ansible Playbook to deploy Sui Validator, Fullnode and Bridge Node 

This Ansible playbook automates the deployment and configuration of Sui Validator, Fullnode and Bridge Node. It ensures that the necessary dependencies, configuration files, and services are properly set up and running.

## Table of Contents

- [Requirements](#requirements)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Variables](#variables)
- [Usage](#usage)

## Requirements

Before using this playbook, ensure the following requirements are met:

1. **Ansible version**: Make sure you have Ansible 2.15+ installed.
2. **SSH Access**: Passwordless SSH access to all target servers.
3. **Python**: Python 3.x installed on the control node and all target hosts.
4. **Privileges**: The user running the playbook must have sudo privileges on the target machines.

## Prerequisites

**Install HashiCorp Vault**

This playbook relies on HashiCorp Vault to securely retrieve sensitive files, such as validator and node keys. Follow the [HashiCorp Vault Installation Guide](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-install) to set up Vault on your infrastructure.

**Note on Secrets Management**

The playbook dynamically retrieves private validator keys and node keys from HashiCorp Vault. The keys are expected to follow a structured path format:
`<environment>/<project>/<organization>/<type>/<file_name>`
For example:
`testnet/sui/encapsulate/validator/network.key`

This structure ensures easy organization and secure retrieval of secrets.

## Setup

### 1. Install Ansible

If Ansible is not installed, visit the official documentation for detailed instructions on how to install Ansible on various Linux distributions:

[Ansible Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/installation_distros.html)

### 2. Clone the repository

Clone this repository to your Ansible control node:

```bash
git clone https://github.com/encapsulate-xyz/sui-ansible.git
cd sui-ansible
```

### 3. Inventory

Define your target servers' IP address or DNS in the inventory folder, and select either `mainnet.yml` or `testnet.yml` to update.

Example for `testnet.yml`

```yaml
---
all:
  vars:
    env: testnet
  children:
    sui:
      hosts:
        bridge.sui.testnet.encapsulate.xyz:
          type: bridge
        validator.sui.testnet.encapsulate.xyz:
          type: validator
        fullnode.sui.testnet.encapsulate.xyz:
          type: fullnode
```

## Variables

This playbook allows customization through several variables. You can define these variables in the following locations:

- **`group_vars/all.yml`**: Contains all the port, source url configurations.
- **`group_vars/mainnet.yml`** or **`group_vars/testnet.yml`**: Contains version specific variables.
- **`group_vars/vault.yml`**: Store secret variables, such as `jwtsecret`, in this file.
- There are role specific variables defined in each roles `default/main.yml` and `vars/main.yml`.

**Note**: Make sure to set the `VAULT_TOKEN` environment variable, as it enables logging in and fetching secrets from HashiCorp Vault.

Create a `group_vars/vault.yml` with your preferred ansible-vault password:

```
ansible-vault create group_vars/vault.yml
```

Example `group_vars/vault.yml`:

```
vault:
  default:
    hcp:
      vault:
        url:
  mainnet:
    bridge:
      sui:
        rpc_url:
      eth:
        rpc_url:
  testnet:
    bridge:
      sui:
        rpc_url:
      eth:
        rpc_url:
    validator:
      aws_access_key:
      aws_secret_key:
```

### Usage

1. First, install the dependencies:

  ```
  ansible-galaxy collection install -r collections/requirements.yml
  ```

2. Create a `ansible_vault_password` file containing ansible-vault password

3. Configure your remote server username and private key file path in the `ansible.cfg` file. Additionally, set the SSH port for your server by adjusting the `ansible_port` variable in `group_vars/all.yml`.

4. Then run the playbook:

- To deploy validator or fullnode:

```
ansible-playbook setup_node.yml -l validator.sui.testnet.encapsulate.xyz -e "fetch_validator_keys=true"
```

- To deploy bridge node:

```bash
ansible-playbook setup_bridge.yml -l bridge.sui.testnet.encapsulate.xyz -e "fetch_validator_keys=true"
```

**Note**: The default value for `fetch_validator_keys` is false, which disables fetching keys from Hashicorp Vault.

After you run the playbook, it will ask for confirmation, displaying all the variables and the IP address or DNS of the server you are going to deploy.

Example output:

```bash
TASK [Display environment being deployed to] ***************************************************************************************************
ok: [validator.sui.testnet.encapsulate.xyz] => {
    "msg": [
        "Deploying to Host: validator.sui.testnet.encapsulate.xyz",
        "Groups: ['sui']",
        "Project: sui",
        "Environment: testnet",
        "Type: validator",
        "Version: 44dc210152dda9c6f64a91573b1f66e9f481bd9e",
        "Service Name: sui",
        "Operating System: linux",
        "System Architecture: amd64"
    ]
}

TASK [Confirm deployment details] **************************************************************************************************************
[Confirm deployment details]
Press 'Enter' to continue with the deployment or Ctrl+C to cancel:
ok: [validator.sui.testnet.encapsulate.xyz]

TASK [Please confirm again] ********************************************************************************************************************
ok: [validator.sui.testnet.encapsulate.xyz] => {
    "msg": [
        "Deploying to Host: validator.sui.testnet.encapsulate.xyz",
        "Project: sui",
        "Environment: testnet",
        "Type: validator"
    ]
}

TASK [Confirm deployment details] **************************************************************************************************************
[Confirm deployment details]
Press 'Enter' to continue with the deployment or Ctrl+C to cancel:
ok: [validator.sui.testnet.encapsulate.xyz]
```

# Sui Bridge Node

Requires limiting 50 requests/second per unique IP, this will require change in the nginx
```bash
CONFIG="server {
listen 80;
server_name ${SUBDOMAIN};

    # Define a limit request zone that tracks the number of requests from each IP
    limit_req_zone \$binary_remote_addr zone=ip_limit:10m rate=50r/s;

    location / {
      limit_req zone=ip_limit burst=10 nodelay;  # Apply the rate limit here
      proxy_pass http://${IP}:${PORT};
      proxy_set_header Host \$host;
      proxy_set_header X-Real-IP \$remote_addr;
      proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto \$scheme;
    }
}"

echo "${CONFIG}" | sudo tee /etc/nginx/sites-available/${SUBDOMAIN}
```

If runs in two mode, client true, or false, if client is enabled, a Sui Account Key with enough SUI balance is needed as BridgeClient submits transaction on Sui Network.
