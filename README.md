# Ansible WordPress Server on LXD/LXC

Automated WordPress installation on LXD containers using Ansible. This playbook installs a complete LAMP stack (Apache, MariaDB, PHP) and WordPress.

## Quick Start

### 1. Prerequisites

- Ansible installed
- LXD/LXC installed
- Ubuntu container with SSH access

```bash
# Install Ansible
sudo apt install ansible

# Install LXD
sudo apt install lxd
sudo lxd init
```

### 2. Create LXD Container

```bash
# Create container
lxc launch ubuntu:22.04 wordpress-server

# Configure SSH access
lxc exec wordpress-server -- bash
adduser ubuntu
usermod -aG sudo ubuntu
apt update && apt install openssh-server
systemctl enable ssh && systemctl start ssh
exit
```

### 3. Configure Inventory

Update `inventory.ini` with your container IP:

```ini
[demo]
your-container-ip

[all:vars]
ansible_user=ubuntu
```

### 4. Install Dependencies

```bash
ansible-galaxy collection install community.mysql
```

### 5. Run Playbook

```bash
ansible-playbook -i inventory.ini playbooks/wordpress.yml
```

### 6. Access WordPress

Open browser: `http://your-container-ip/wordpress`

Database settings:
- **Database Name**: `wordpress_db`
- **Username**: `wordpress`
- **Password**: `wordpress_pwd`
- **Host**: `localhost`

## What Gets Installed

- **Apache2** web server
- **MariaDB** database (root password: `password`)
- **PHP** with extensions
- **WordPress** (latest version)

## Troubleshooting

```bash
# Test connectivity
ansible -i inventory.ini demo -m ping

# Check services
ansible -i inventory.ini demo -m shell -a "sudo systemctl status apache2 mariadb"

# View logs
ansible -i inventory.ini demo -m shell -a "sudo journalctl -u apache2 -f"
```

---
**Note**: This is for development/testing only. Use proper security for production.