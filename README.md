# Ansible EC2 Automator

[![Ansible](https://img.shields.io/badge/Ansible-8.0%2B-red?style=flat-square&logo=ansible)](https://ansible.com)
[![AWS](https://img.shields.io/badge/AWS-EC2-orange?style=flat-square&logo=amazon-aws)](https://aws.amazon.com/ec2/)
[![Python](https://img.shields.io/badge/Python-3.8%2B-blue?style=flat-square&logo=python)](https://python.org)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

AWS EC2 automation with Ansible for infrastructure provisioning and management.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Detailed Setup](#detailed-setup)
- [Instance Connection](#instance-connection)
- [Acceptance Checks](#acceptance-checks)
- [Project Structure](#project-structure)
- [Usage](#usage)
- [Playbooks](#playbooks)
- [Security](#security)
- [Troubleshooting](#troubleshooting)

## Overview

This project provides Ansible automation for:

- **Provisioning** EC2 instances with different OS types
- **Managing** multiple instances simultaneously
- **Secure** credential handling with Ansible Vault
- **Gathering** system information and facts
- **Shutting down** instances by OS family

### Supported Features

- Multi-OS support (Ubuntu, Amazon Linux)
- Secure credential management
- Static inventory in `inventory/inventory.yaml`
- SSH key automation
- Security group configuration
- Fact gathering and system information

## Prerequisites

### AWS Requirements

- AWS account with permission to manage the EC2 resources used by these playbooks
- AWS access key and secret key stored locally in an encrypted Ansible Vault file
- SSH key pair and security group prepared for the target region

### Local System Requirements

- **Python**: 3.8 or higher
- **Operating System**: Linux
- **Git**: For cloning the repository

## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/horus0523/ansible-ec2-automator
cd ansible-ec2-automator

# 2. Set up virtual environment
python3 -m venv venv
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt
ansible-galaxy collection install amazon.aws

# 4. Configure inventory, vault credentials, and SSH key material
# 5. Run acceptance checks before touching AWS
ansible-playbook playbooks/create_ec2_instances.yaml --syntax-check

# 6. Create EC2 instances
ansible-playbook playbooks/create_ec2_instances.yaml

# 7. Configure SSH access (see instance connection)
# 8. Run management playbooks
```

## Detailed Setup

### 1. AWS IAM User Setup

#### 1.1 Create IAM User

1. **Log in to AWS Console** with permission to create or update IAM users
2. Navigate to **Services > IAM**
3. Select **Users** -> **Add user**
4. **Username**: `ansible-user` (or your preferred name)
5. **Permissions**: Select **Attach policies directly**
6. Attach a policy scoped to the EC2 actions and resources required for the target environment.
7. **Create user** and note the username

#### 1.2 Generate Access Keys

1. Click on the newly created user
2. Go to **Security credentials** tab
3. In **Access keys** section -> **Create access key**
4. Select **Application running outside AWS** -> **Next**
5. **Create access key**
6. Save the Access Key ID and Secret Access Key securely

### 2. Local Environment Setup

#### 2.1 Clone and Setup Project

```bash
# Clone the repository
git clone https://github.com/horus0523/ansible-ec2-automator
cd ansible-ec2-automator
```

```bash
# Install venv
sudo apt update
sudo apt install python3.12-venv
```

```bash
# Create and activate virtual environment
python3 -m venv venv
source venv/bin/activate
```

```bash
# Verify pip installation
pip --version
```

```bash
# If not installed, use
sudo apt install python3-pip
```

```bash
# Install Python dependencies
python3 -m pip install -r requirements.txt
```

```bash
# Install Ansible AWS collection
ansible-galaxy collection install amazon.aws
```

#### 2.2 Generate Vault Password

```bash
# Generate secure vault password
openssl rand -base64 2048 > vault.pass
chmod 600 vault.pass
```

#### 2.3 Configure AWS Credentials

Create `group_vars/all/pass.yml` as an encrypted local file before running any AWS playbook.
If your branch includes `group_vars/all/vault.example.yml`, copy that file first and then encrypt the local copy.

```bash
ansible-vault create group_vars/all/pass.yml --encrypt-vault-id default
```

**Add your AWS credentials in the editor:**

```yaml
ec2_access_key: "YOUR_AWS_ACCESS_KEY_ID"
ec2_secret_key: "YOUR_AWS_SECRET_ACCESS_KEY"
```

Save and close the editor.

### 3. AWS Infrastructure Setup

The repository uses the static inventory at `inventory/inventory.yaml`. Replace the placeholder hosts with the public IPs for your environment and keep connection data such as `ansible_user` and `ansible_ssh_private_key_file` there.

#### 3.1 Create SSH Key Pair

1. **AWS Console** -> **Services > EC2**
2. **Network & Security** -> **Key Pairs**
3. **Create key pair**
4. **Name**: `my-ansible-key`
5. **Key pair type**: `ED25519`
6. **File format**: `.pem`
7. **Create** and download the key

```bash
# Move downloaded key to project directory
mkdir -p ~/ansible-ec2-automator/files/ssh_keys
mv ~/my-ansible-key.pem ~/ansible-ec2-automator/files/ssh_keys/
chmod 400 ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem
```

#### 3.2 Create Security Group

1. **AWS Console** -> **Services > EC2**
2. **Network & Security** -> **Security Groups**
3. **Create security group**
4. **Name**: `ansible-sg-ssh`
5. **Description**: "SSH access for Ansible managed instances"
6. **Inbound rules**:
   - **Type**: SSH (22)
   - **Source**: Your public IP or another intentionally restricted source
7. **Create security group**

#### 4. Provision EC2 Instances

```bash
# Run from the project directory
cd ~/ansible-ec2-automator/

# Create the instances
ansible-playbook playbooks/create_ec2_instances.yaml
```

Before running the playbook, verify this checklist:

- `group_vars/all/pass.yml` exists and is encrypted.
- `vault.pass` exists locally.
- `files/ssh_keys/my-ansible-key.pem` exists and has `chmod 400`.
- `inventory/inventory.yaml` has the managed hosts and users for your instances.

## Instance Connection

After creating the instances, you need to configure SSH access and test connectivity.

### 4.1 Configure Passwordless Authentication

Replace `<MANAGE-NODE-IP>` with the actual IP addresses from your instance creation:

```bash
# Generate the public key
ssh-keygen -y -f ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem > ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pub
```

```bash
# Start the SSH agent in your current session
eval "$(ssh-agent -s)"

# Add the SSH private key to the agent for passwordless authentication
ssh-add ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem
```

```bash
# manage-node-1 (Amazon Linux)
ssh-copy-id -f "-o IdentityFile ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem" ec2-user@<MANAGE-NODE-1-IP>
```

```bash
# manage-node-2 (Ubuntu)
ssh-copy-id -f "-o IdentityFile ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem" ubuntu@<MANAGE-NODE-2-IP>
```

```bash
# manage-node-3 (Ubuntu)
ssh-copy-id -f "-o IdentityFile ~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem" ubuntu@<MANAGE-NODE-3-IP>
```

### 4.2 Test SSH Connectivity

#### Direct SSH Connection (with key)

```bash
# Test connection with private key (modify the path to the my-ansible-key.pem file)
ssh -i "~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem" ec2-user@<MANAGE-NODE-1-IP>
```

```bash
ssh -i "~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem" ubuntu@<MANAGE-NODE-2-IP>
```

```bash
ssh -i "~/ansible-ec2-automator/files/ssh_keys/my-ansible-key.pem" ubuntu@<MANAGE-NODE-3-IP>
```

#### Passwordless SSH Connection (after ssh-copy-id)

```bash
# Test passwordless connection
ssh ec2-user@<MANAGE-NODE-1-IP>
```

```bash
ssh ubuntu@<MANAGE-NODE-2-IP>
```

```bash
ssh ubuntu@<MANAGE-NODE-3-IP>
```

### 4.3 Update Inventory File

Create or update `inventory/inventory.yaml` with your instance IPs:

```yaml
all:
  children:
    debian_instances:
      hosts:
        <MANAGE-NODE-2-IP>:
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ./files/ssh_keys/my-ansible-key.pem
        <MANAGE-NODE-3-IP>:
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ./files/ssh_keys/my-ansible-key.pem
    redhat_instances:
      hosts:
        <MANAGE-NODE-1-IP>:
          ansible_user: ec2-user
          ansible_ssh_private_key_file: ./files/ssh_keys/my-ansible-key.pem
```

### 4.4 Test Ansible Connectivity

```bash
# Test Ansible can reach all instances
ansible all -m ping
```

```bash
# Test specific groups debian instances
ansible debian_instances -m ping
```

```bash
# Test specific groups redhat instances
ansible redhat_instances -m ping
```

## Acceptance Checks

Run acceptance checks before relying on AWS execution results or implementation detail sections.

| Check | Command | Notes |
|-------|---------|-------|
| Syntax | `ansible-playbook playbooks/create_ec2_instances.yaml --syntax-check` | Parses the playbook without executing tasks. |
| Check mode | `ansible-playbook playbooks/create_ec2_instances.yaml --check` | Requires valid AWS credentials. For playbooks that connect to managed hosts, it also requires a usable `inventory/inventory.yaml` and working host connectivity. Placeholder inventory entries are not enough for live checks. |
| Connectivity | `ansible all -m ping` | Verifies the inventory and SSH configuration against reachable hosts. |
| CI baseline | See `.github/workflows/ansible-ci.yml` when that workflow is part of the branch | The current automated checks cover `ansible-lint --offline` and Molecule scenario discovery; syntax, check mode, and connectivity still need to be run locally. |

## Project Structure

```text
ansible-ec2-automator/
├── README.md                    # README file
├── requirements.txt             # Python dependencies
├── ansible.cfg                  # Ansible configuration
├── vault.pass                   # Vault password (local only)
├── .gitignore                   # Git exclusions
├──
├── playbooks/                   # Ansible playbooks
│   ├── create_ec2_instances.yaml
│   ├── show_gathered_facts.yaml
│   ├── show_only_os_family_fact.yaml
│   ├── shutdown_debian_ec2_instances.yaml
│   └── shutdown_redhat_ec2_instances.yaml
├──
├── group_vars/                  # Group variables
│   └── all/
│       └── pass.yml             # Encrypted credentials (local only)
├──
├── inventory/                   # Inventory files
│   └── inventory.yaml           # Static inventory
├──
└── files/                       # Additional files
    └── ssh_keys/
        └── my-ansible-key.pem   # SSH private key (local only)
```

## Usage

### Basic Commands

```bash
# Activate virtual environment (always first)
source venv/bin/activate
```

```bash
# Create EC2 instances
ansible-playbook playbooks/create_ec2_instances.yaml
```

```bash
# Show system facts
ansible-playbook playbooks/show_gathered_facts.yaml
```

```bash
# Show only OS family information
ansible-playbook playbooks/show_only_os_family_fact.yaml
```

```bash
# Shutdown Debian-based instances (Ubuntu)
ansible-playbook playbooks/shutdown_debian_ec2_instances.yaml
```

```bash
# Shutdown RedHat-based instances (Amazon Linux)
ansible-playbook playbooks/shutdown_redhat_ec2_instances.yaml
```

### Advanced Commands

```bash
# Check playbook syntax
ansible-playbook playbooks/create_ec2_instances.yaml --syntax-check
```

```bash
# Dry run (check mode)
ansible-playbook playbooks/create_ec2_instances.yaml --check
```

```bash
# Run with extra verbosity
ansible-playbook playbooks/create_ec2_instances.yaml -vvv
```

```bash
# Run specific tasks with tags
ansible-playbook playbooks/create_ec2_instances.yaml --tags "environment"
```

## Playbooks

### create_ec2_instances.yaml

**Purpose**: Provisions 3 EC2 instances with different configurations

**Instances Created**:

- `manage-node-1`: Amazon Linux (ec2-user)
- `manage-node-2`: Ubuntu (ubuntu)
- `manage-node-3`: Ubuntu (ubuntu)

**Features**:

- Automated security group association
- Public IP assignment
- Instance tagging
- SSH key configuration

### show_gathered_facts.yaml

**Purpose**: Displays comprehensive system information from all managed instances

**Use Cases**:

- System inventory
- Compliance checking
- Configuration validation

### show_only_os_family_fact.yaml

**Purpose**: Shows only OS family information (Debian, RedHat, etc.)

**Use Cases**:

- OS-specific task planning
- Conditional playbook execution

### shutdown_debian_ec2_instances.yaml

**Purpose**: Safely shuts down Ubuntu/Debian instances

**Safety Features**:

- OS family verification
- Conditional execution

### shutdown_redhat_ec2_instances.yaml

**Purpose**: Safely shuts down Amazon Linux/RedHat instances

**Safety Features**:

- OS family verification
- Conditional execution

## Security

### Repository Security Practices

- **Encrypted Credentials**: AWS keys stored in Ansible Vault
- **Local Secrets**: Vault password not committed to repository
- **SSH Security**: Private keys excluded from Git
- **Secure Defaults**: Review `ansible.cfg` and your SSH options before automation against new hosts

### Security Prerequisites

- **Access Control Guidance**: Use an IAM user or role with only the permissions required for your environment

### Files NOT in Repository

```bash
# These files are automatically excluded via .gitignore
vault.pass                    # Vault password
files/ssh_keys/*.pem         # SSH private keys
group_vars/all/pass.yml      # Encrypted credentials (optional)
```

### Security Checklist

- AWS IAM user has only necessary permissions
- Vault password is strong and unique
- SSH keys have correct permissions (400)
- Security groups limit access to necessary ports
- Production environments use restricted IP ranges

## Troubleshooting

### Common Issues

#### 1. "ec2_access_key is undefined"

**Solution**: Verify vault configuration

```bash
# Check vault contents
ansible-vault view group_vars/all/pass.yml --vault-password-file vault.pass
```

```bash
# Test variable loading
ansible localhost -m debug -a "var=ec2_access_key"
```

#### 2. SSH Connection Failed

**Solution**: Check inventory format and test manual SSH

```bash
# Test manual SSH connection first
ssh -i ./files/ssh_keys/my-ansible-key.pem ubuntu@<INSTANCE-IP>
```

```bash
# Test Ansible connectivity
ansible all -m ping
```

#### 3. Playbook Syntax Errors

**Solution**: Validate syntax

```bash
ansible-playbook playbooks/create_ec2_instances.yaml --syntax-check
```

### Debug Commands

```bash
# Show Ansible configuration
ansible-config dump | grep vault
```

```bash
# Test inventory
ansible-inventory --list
```

```bash
# View hosts by group
ansible-inventory --graph
```

```bash
# Test connectivity
ansible all -m ping
```

```bash
# Test Specific Groups
ansible debian_instances -m ping
```

```bash
ansible redhat_instances -m ping
```

```bash
# Show gathered facts
ansible all -m setup
```

```bash
# Debug vault variables
ansible localhost -m debug -a "var=ec2_access_key"
ansible localhost -m debug -a "var=ec2_secret_key"
```

## Useful Links

- [Ansible Documentation](https://docs.ansible.com/)
- [AWS EC2 Documentation](https://docs.aws.amazon.com/ec2/)
- [Amazon AWS Ansible Collection](https://docs.ansible.com/ansible/latest/collections/amazon/aws/)
