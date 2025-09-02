# Ansible Playbook for Percona XtraDB Cluster 8.0

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Overview

This repository contains a flexible set of Ansible playbooks to deploy a secure, multi-node Percona XtraDB Cluster (PXC) on Ubuntu/Debian systems. It provides a turn-key solution for creating a high-availability MySQL cluster with synchronous multi-master replication, automated TLS encryption, and an optional Galera Arbitrator for maintaining quorum.

The playbooks are designed to be generic, supporting any number of data nodes (a minimum of 3 is recommended for a high-availability setup).

## Why Percona XtraDB Cluster?

Percona XtraDB Cluster (PXC) is a fully open-source database clustering solution that is widely adopted when high availability and data consistency are critical. Its synchronous replication model ensures that transactions are committed on all nodes or not at all, eliminating data drift between nodes. Each node in the cluster contains the same set of data, providing robust data redundancy and automatic failover.

## Repository Structure

The project is organized as a standard Ansible playbook:

| Path                             | Purpose                                                                                |
| -------------------------------- | -------------------------------------------------------------------------------------- |
| `inventory/hosts.ini`            | Defines the inventory of PXC data nodes and the optional arbiter.                      |
| `plays/00_tls.yml`               | Playbook for generating the CA and node TLS certificates.                              |
| `plays/01_pxc.yml`               | Main playbook for deploying and configuring the PXC cluster.                           |
| `plays/ansible.cfg`              | Ansible configuration file specifying inventory, roles path, etc.                      |
| `roles/pxc_node/tasks/main.yml`  | Main task file for the `pxc_node` role, configuring data nodes.                        |
| `roles/pxc_node/defaults/main.yml`| Default variables for the `pxc_node` role, e.g., for dynamic InnoDB tuning.            |
| `roles/pxc_node/templates/mysqld.cnf.j2` | Jinja2 template for the MySQL configuration file (`mysqld.cnf`).                         |
| `roles/pxc_garb/tasks/main.yml`  | Main task file for the `pxc_garb` role, configuring the arbitrator.                    |
| `roles/pxc_garb/templates/garb.j2` | Jinja2 template for the Galera Arbitrator configuration file.                            |
| `LICENSE`                        | The MIT License file for the project.                                                  |
| `README.md`                      | This documentation file.                                                               |

## Requirements

- **Control Node**:
    - Ansible `2.9` or later.
    - `community.crypto` Ansible collection.
- **Target Nodes**:
    - A flexible number of Ubuntu/Debian servers.
    - For a high-availability (HA) production cluster, a minimum of **3 data nodes** is recommended.
    - An **optional arbiter node** can be added to the cluster, which is particularly useful for setups with an even number of data nodes.
    - SSH access with key-based authentication.
    - The Ansible user must have `sudo` privileges.
- **Networking**: Ensure nodes can communicate over the following ports:
    - `3306` – MySQL client connections & State Snapshot Transfer (SST).
    - `4444` – SST via XtraBackup.
    - `4567` – Galera replication traffic (TCP & UDP).
    - `4568` – Incremental State Transfer (IST).

## Quick Start Guide

**1. Prepare the Control Node**

On your Ansible control machine (e.g., Ubuntu/Debian), install Ansible and the required collection:
```bash
sudo apt update && sudo apt install ansible python3-pip -y
ansible-galaxy collection install community.crypto
```

**2. Clone the Repository**
```bash
git clone https://github.com/cracya01/ansible-pxc-cluster-ubuntu.git
cd ansible-pxc-cluster-ubuntu/
```

**3. Configure the Inventory**

Edit `inventory/hosts.ini` to define your cluster. Ensure one data node is designated as the bootstrap node by setting `bootstrap_first_node=true`.

**Example for a 3-node cluster:**
```ini
[pxc_nodes]
pxc-node-1 ansible_host=192.168.1.2 wsrep_node_name=pxc-node-1 wsrep_node_address=192.168.1.2 server_id=101 bootstrap_first_node=true
pxc-node-2 ansible_host=192.168.1.3 wsrep_node_name=pxc-node-2 wsrep_node_address=192.168.1.3 server_id=102
pxc-node-3 ansible_host=192.168.1.4 wsrep_node_name=pxc-node-3 wsrep_node_address=192.168.1.4 server_id=103

# To add an arbiter, uncomment and configure the following section:
# [pxc_arbiter]
# pxc-arbiter-1 ansible_host=192.168.1.5 wsrep_node_name=pxc-arbiter-1 wsrep_node_address=192.168.1.5

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

**4. Run the Playbooks**

All commands must be executed from the `plays/` directory.
```bash
cd plays/
```

First, generate the TLS certificates for all nodes:
```bash
ansible-playbook 00_tls.yml
```

Then, deploy the PXC cluster:
```bash
ansible-playbook 01_pxc.yml
```

**5. Verify Cluster Status**

After deployment, connect to any data node to check the cluster size. The initial root password is `changeme`.
```bash
mysql -u root -p'changeme' -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```
The output should match the total number of nodes in your cluster.

## Security Considerations

- **Change Default Password**: The most critical step is to change the `root` password immediately.
  ```sql
  ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewStrongPassword';
  ```
- **Firewall**: Limit network access to the required ports.
- **Client Connections**: Configure applications to connect using TLS.

## Customization

- **Scaling**: Add or remove hosts from `inventory/hosts.ini` and re-run the playbooks.
- **Remote User**: If your SSH user is not `root`, modify the `remote_user` parameter in `plays/ansible.cfg` or pass `-u <username>` to the `ansible-playbook` command.
- **TLS Regeneration**: Force regeneration of all certificates with: `ansible-playbook 00_tls.yml -e "regenerate_tls=true"`.
- **Resource Tuning**: Override default InnoDB parameters in your inventory.

## License

This project is released under the terms of the MIT License. See the `LICENSE` file for the full text.

## References

- [Percona XtraDB Cluster Documentation](https://www.percona.com/doc/percona-xtradb-cluster/8.0/index.html)
- [Ansible Documentation](https://docs.ansible.com/)
