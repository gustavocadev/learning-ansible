# Ansible Lab: WordPress Server Configuration with LXD/LXC

This repository contains Ansible playbooks to automatically configure a WordPress server running on LXD/LXC containers. The setup includes a complete LAMP stack (Linux, Apache, MySQL/MariaDB, PHP) with WordPress installation and configuration.

## Table of Contents

- [Prerequisites](#prerequisites)
- [LXD/LXC Container Setup](#lxdlxc-container-setup)
- [Ansible Configuration](#ansible-configuration)
- [Playbook Overview](#playbook-overview)
- [Usage](#usage)
- [What Gets Installed](#what-gets-installed)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before running this Ansible playbook, ensure you have:

- **Ansible installed** on your control machine (version 2.9+)
- **LXD/LXC** installed and configured
- **SSH access** to the target container
- **sudo privileges** on the target container

### Installing Ansible

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible

# CentOS/RHEL
sudo yum install ansible

# macOS
brew install ansible
```

### Installing LXD

```bash
# Ubuntu/Debian
sudo apt install lxd
sudo lxd init
```

## LXD/LXC Container Setup

### 1. Create and Launch Container

```bash
# Create a new Ubuntu 22.04 container
lxc launch ubuntu:22.04 wordpress-server

# Check container status
lxc list

# Get container IP address
lxc info wordpress-server
```

### 2. Configure Container for Ansible

```bash
# Access the container
lxc exec wordpress-server -- bash

# Create ubuntu user (if not exists)
adduser ubuntu
usermod -aG sudo ubuntu

# Enable SSH access
apt update
apt install openssh-server
systemctl enable ssh
systemctl start ssh

# Set up SSH key authentication (recommended)
# Copy your public key to the container
mkdir -p /home/ubuntu/.ssh
echo "your-public-key-here" >> /home/ubuntu/.ssh/authorized_keys
chown -R ubuntu:ubuntu /home/ubuntu/.ssh
chmod 700 /home/ubuntu/.ssh
chmod 600 /home/ubuntu/.ssh/authorized_keys

# Allow password authentication (alternative to key-based)
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
systemctl restart ssh

# Exit container
exit
```

### 3. Test Container Connectivity

```bash
# Test SSH connection
ssh ubuntu@<container-ip>

# Test with Ansible
ansible -i inventory.ini demo -m ping
```

## Ansible Configuration

### Inventory File (`inventory.ini`)

The inventory file defines the target hosts and connection parameters:

```ini
[demo]
10.148.137.142

[all:vars]
ansible_user=ubuntu
```

**Configuration Details:**
- `[demo]`: Host group name containing the container IP
- `10.148.137.142`: IP address of the LXD container
- `ansible_user=ubuntu`: Default user for SSH connections

### Ansible Collections Required

Install the required Ansible collections:

```bash
ansible-galaxy collection install community.mysql
```

## Playbook Overview

The `playbooks/wordpress.yml` playbook performs the following tasks:

### System Updates and Apache Installation
- Updates all system packages
- Installs and configures Apache2 web server
- Enables Apache to start automatically on boot

### PHP Installation and Configuration
- Installs PHP and essential PHP extensions:
  - `php-mysql` (MySQL database connectivity)
  - `php-xml`, `php-xmlrpc` (XML processing)
  - `php-curl` (HTTP client functionality)
  - `php-gd`, `php-imagick` (Image processing)
  - `php-mbstring` (Multi-byte string handling)
  - `php-opcache` (Performance optimization)
  - `php-soap`, `php-zip`, `php-intl` (Additional functionality)

### MariaDB Database Setup
- Installs MariaDB server and client
- Configures MariaDB to start automatically
- Sets up root password security
- Removes default test database
- Creates dedicated WordPress database (`wordpress_db`)
- Creates WordPress database user with appropriate privileges

### Python Dependencies
- Installs Python3 and pip
- Removes PEP 668 restrictions for system-wide package installation
- Installs `mysqlclient` Python library for database operations

### WordPress Installation
- Downloads the latest WordPress release
- Extracts WordPress files to `/var/www/html/`

## Usage

### 1. Update Inventory

Edit `inventory.ini` to match your container's IP address:

```ini
[demo]
your-container-ip-here

[all:vars]
ansible_user=ubuntu
```

### 2. Run the Playbook

Execute the WordPress installation playbook:

```bash
# Run the complete playbook
ansible-playbook -i inventory.ini playbooks/wordpress.yml

# Run with verbose output for debugging
ansible-playbook -i inventory.ini playbooks/wordpress.yml -v

# Run specific tasks with tags (if you add them)
ansible-playbook -i inventory.ini playbooks/wordpress.yml --tags "database"
```

### 3. Complete WordPress Setup

After the playbook completes successfully:

1. **Access WordPress**: Open your browser and navigate to `http://<container-ip>/wordpress`

2. **Complete Installation**: Follow the WordPress setup wizard:
   - **Database Name**: `wordpress_db`
   - **Username**: `wordpress`
   - **Password**: `wordpress_pwd`
   - **Database Host**: `localhost`
   - **Table Prefix**: `wp_` (default)

3. **Create Admin User**: Set up your WordPress administrator account

## What Gets Installed

### Software Stack
- **Operating System**: Ubuntu 22.04 LTS (in LXD container)
- **Web Server**: Apache2
- **Database**: MariaDB 10.6+
- **PHP**: PHP 8.1+ with essential extensions
- **CMS**: WordPress (latest version)

### Database Configuration
- **Root Password**: `password`
- **WordPress Database**: `wordpress_db`
- **WordPress User**: `wordpress` (password: `wordpress_pwd`)
- **Privileges**: Full access to WordPress database only

### File Locations
- **WordPress Files**: `/var/www/html/wordpress/`
- **Apache Configuration**: `/etc/apache2/`
- **PHP Configuration**: `/etc/php/8.1/`
- **MariaDB Configuration**: `/etc/mysql/`

## Security Considerations

### Current Security Measures
- MariaDB root remote access disabled
- Test database removed
- Dedicated WordPress database user with limited privileges
- System packages updated to latest versions

### Additional Security Recommendations

```bash
# Change default passwords
mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY 'strong-root-password';
ALTER USER 'wordpress'@'localhost' IDENTIFIED BY 'strong-wordpress-password';

# Configure Apache security headers
# Add to /etc/apache2/sites-available/000-default.conf
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff

# Enable firewall
ufw enable
ufw allow ssh
ufw allow http
ufw allow https

# Set up SSL/TLS (recommended for production)
certbot --apache
```

## Troubleshooting

### Common Issues

#### SSH Connection Problems
```bash
# Test SSH connectivity
ssh -v ubuntu@<container-ip>

# Check SSH service in container
lxc exec wordpress-server -- systemctl status ssh
```

#### Ansible Permission Errors
```bash
# Ensure ubuntu user has sudo privileges
lxc exec wordpress-server -- usermod -aG sudo ubuntu

# Test sudo access
ansible -i inventory.ini demo -m shell -a "sudo whoami" --ask-become-pass
```

#### Database Connection Issues
```bash
# Check MariaDB status
ansible -i inventory.ini demo -m shell -a "sudo systemctl status mariadb"

# Test database connection
ansible -i inventory.ini demo -m shell -a "mysql -u wordpress -pwordpress_pwd -e 'SHOW DATABASES;'"
```

#### WordPress Access Problems
```bash
# Check Apache status
ansible -i inventory.ini demo -m shell -a "sudo systemctl status apache2"

# Verify file permissions
ansible -i inventory.ini demo -m shell -a "ls -la /var/www/html/"

# Check Apache error logs
ansible -i inventory.ini demo -m shell -a "sudo tail -f /var/log/apache2/error.log"
```

### Verification Commands

```bash
# Check all services are running
ansible -i inventory.ini demo -m shell -a "sudo systemctl is-active apache2 mariadb"

# Verify PHP installation
ansible -i inventory.ini demo -m shell -a "php --version"

# Test database connectivity
ansible -i inventory.ini demo -m shell -a "mysql -u wordpress -pwordpress_pwd -e 'SELECT VERSION();'"

# Check WordPress files
ansible -i inventory.ini demo -m shell -a "ls -la /var/www/html/wordpress/"
```

## Contributing

Feel free to submit issues and enhancement requests! When contributing:

1. Fork the repository
2. Create a feature branch
3. Test your changes thoroughly
4. Submit a pull request with detailed description

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

**Note**: This setup is designed for development and testing purposes. For production environments, implement additional security measures, SSL certificates, regular backups, and monitoring solutions.