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

### 4. Security Hardening

```bash
ansible-playbook playbooks/secure_mariadb.yml --ask-vault-pass
```

### 5. Create microservice databases

```bash
ansible-playbook playbooks/setup_databases.yml --ask-vault-pass
```


## Configuration

### AWS EC2 Dynamic Inventory

The `inventory/aws_ec2.yml` file configures dynamic inventory for AWS EC2 instances.

### Galera Configuration

The `templates/galera.cnf.j2` contains the MariaDB Galera cluster configuration.

### Variables

Edit `group_vars/all.yml` to configure:

- MariaDB version
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

### check_galera_cluster.yml

- Monitoring the cluster's health
- verifying node synchronisation
- Checking cluster connectivity
- Ensuring all nodes are operational

### secure_mariadb.yml

Some of the security / hardening implemented: 

#### a. Password Policies
- Minimum length: 12 characters 
- Basic character requirements configured
- Password validation plugin enabled

#### b. User Access Control
- Anonymous users removed 
- Test database dropped 
- Root password stored securely

#### c. Configuration Security
- Root password stored in Ansible vault
- Credentials managed through vault encryption
- Basic MariaDB configuration hardening
- Basic file permissions (mode: '0644') for config files

#### d. Database Isolation
- CustomerOrder and PackageTracking databases separated

#### e. Monitoring & Auditing
- Performance logging enabled
- Error logging configured
- Access logging enabled

## Security Notes

1. Root password is stored in `group_vars/vault.yml`
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

3. If galera error persist, reinstall mariaDB: 
```bash
ansible-playbook playbooks/uninstall_mariadb.yml
```

## Galera Cluster Auto-Restart Configuration

### Overview
Implements automatic restart of Galera nodes after system reboot with proper cluster synchronization:

- **First Node (Bootstrap Node)**
  - Uses `galera_new_cluster` for initialization
  - 5-second restart delay
  - Ensures cluster bootstrap before other nodes join

- **Secondary Nodes**
  - 30-second restart delay
  - Waits for bootstrap node
  - Maintains cluster synchronization

### Implementation Details

The configuration uses systemd override files:
- Located in `/etc/systemd/system/mariadb.service.d/galera.conf`
- Different settings for bootstrap vs secondary nodes
- Ensures proper startup order
- Handles network dependencies

### Verification

```bash
# Check configuration on all nodes
ansible role_database -m shell -a "sudo cat /etc/systemd/system/mariadb.service.d/galera.conf" --ask-vault-pass

# Check if service is enabled on all nodes
ansible role_database -m shell -a "sudo systemctl is-enabled mariadb" --ask-vault-pass

# Check service status on all nodes
ansible role_database -m shell -a "sudo systemctl status mariadb" --ask-vault-pass
```

### Notes
- First node must be fully operational before other nodes join
- Network availability is verified before startup
- Automatic restart on failure with configurable delays
