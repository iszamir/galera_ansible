# MariaDB Galera Cluster Setup with Ansible

This project contains Ansible playbooks for setting up and managing a MariaDB Galera cluster on AWS EC2 instances.

## Project Structure

```
ansible/
├── inventory/
│   └── aws_ec2.yml          # Dynamic AWS inventory configuration
├── group_vars/
│   └── all.yml             # Global variables including passwords and configs
├── playbooks/
│   ├── aws_setup.yml            # AWS infrastructure setup
│   ├── galera_setup.yml         # Main Galera cluster setup
│   ├── uninstall_mariadb.yml    # Clean uninstall of MariaDB
│   ├── setup_root_password.yml  # One-time root password setup
|   └── check_galera_cluster.yml # Check Galera cluster
└── templates/
    └── galera.cnf.j2       # Galera cluster configuration template
```

## Prerequisites

- Ansible 2.9+
- AWS credentials configured
- Python 3.x with boto3

## Infrastructure
- 3 AWS EC2 instances running Ubuntu 
- MariaDB Galera Cluster
- Ansible for automation

## Setup Steps

### 1. AWS Infrastructure Setup

```bash
ansible-playbook playbooks/aws_setup.yml
```

### 2. Setting up the Galera Cluster

```bash
# Main cluster setup
ansible-playbook playbooks/galera_setup.yml

# After cluster is running, set up root password (one-time setup)
ansible-playbook playbooks/setup_root_password.yml
```

### 3. Cleanup (if needed)

```bash
ansible-playbook playbooks/uninstall_mariadb.yml
```

### 4. Create microservice databases

```bash
ansible-playbook playbooks/setup_databases.yml
```

## Configuration

### AWS EC2 Dynamic Inventory

The `inventory/aws_ec2.yml` file configures dynamic inventory for AWS EC2 instances.

### Galera Configuration

The `templates/galera.cnf.j2` contains the MariaDB Galera cluster configuration.

### Variables

Edit `group_vars/all.yml` to configure:

- MariaDB version
- Root password
- Cluster name
- Other MySQL/Galera settings

## Playbook Details

### galera_setup.yml

- Installs MariaDB and Galera packages
- Configures cluster settings
- Bootstraps first node
- Joins additional nodes to cluster

### setup_root_password.yml

- One-time setup of root password and saves it in the vault
- Configures root access permissions
- Creates .my.cnf for root access
- Saved password can be seen at ~/.mariadb/credentials"

### uninstall_mariadb.yml

- Completely removes MariaDB
- Cleans up data directories
- Removes configuration files

## Security Notes

1. Root password is stored in `group_vars/all.yml`
2. Firewall rules are configured for ports:
   - 3306 (MySQL)
   - 4567 (Galera)
   - 4568 (IST)
   - 4444 (SST)

## Troubleshooting

1. If cluster fails to start:

   ```bash
   sudo journalctl -u mariadb
   sudo tail -f /var/log/mysql/error.log
   ```
2. To manually check the cluster's status:

   ```bash
   mysql -u root -p -e "SHOW STATUS LIKE 'wsrep%';"
   ```
