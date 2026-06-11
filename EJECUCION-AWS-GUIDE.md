# Ansible EC2 Automator AWS Execution Guide

This guide describes how to run the existing Ansible playbooks against AWS and how to record evidence that can be reproduced from the checked-in playbooks, static inventory, and local configuration.

It does not include live AWS results. Record live metrics only after running the commands in your own AWS account.

## Scope and limits

| Area | Current operating rule |
|------|------------------------|
| Inventory | The repository uses `inventory/inventory.yaml` as a static inventory. Replace placeholder hosts before host-targeted playbooks or connectivity checks. |
| Credentials | AWS keys must be stored in a local vault-encrypted `group_vars/all/pass.yml` created from `group_vars/all/vault.example.yml`. Do not commit real credentials or `vault.pass`. |
| AWS access | Use credentials with only the EC2 permissions required for the target environment. Avoid broad administrative policies. |
| Check mode | `--check` avoids the provisioning task mutation path, but it still requires valid local prerequisites. Host-targeted checks also require reachable hosts. |
| CI evidence | Do not cite CI or cloud-backed evidence unless the workflow and logs are checked in or linked from the reviewed change. |
| Live evidence | Do not publish instance IDs, timings, or cost results unless they were captured from the current playbooks and environment. |

## Prerequisites

Run commands from the repository root unless a step says otherwise.

```bash
python3 -m venv venv
source venv/bin/activate
python3 -m pip install -r requirements.txt
ansible-galaxy collection install amazon.aws
aws --version
aws sts get-caller-identity
```

Before any AWS execution, confirm:

- AWS credentials are configured for the account and region you intend to use.
- AWS CLI is installed locally and configured for the account and region you intend to use.
- `group_vars/all/pass.yml` exists locally and is encrypted with Ansible Vault.
- `vault.pass` exists locally or another vault password method is available.
- `files/ssh_keys/my-ansible-key.pem` exists locally and has restrictive permissions.
- `inventory/inventory.yaml` contains real reachable host addresses when running host-targeted playbooks.
- AMI IDs and region defaults still match the target AWS region.

## Configure encrypted AWS variables

Create the local vault file from the checked-in example:

```bash
cp group_vars/all/vault.example.yml group_vars/all/pass.yml
ansible-vault encrypt group_vars/all/pass.yml --vault-password-file vault.pass
ansible-vault edit group_vars/all/pass.yml --vault-password-file vault.pass
```

The encrypted file must define:

```yaml
ec2_access_key: "YOUR_AWS_ACCESS_KEY_ID"
ec2_secret_key: "YOUR_AWS_SECRET_ACCESS_KEY"
```

Routine verification does not require `ansible-vault view`. That command decrypts the file and prints its contents to the terminal, so use it only in a trusted local session when you intentionally need to inspect the values. The syntax and check-mode steps later in this guide verify that Ansible can decrypt the file during execution.

## Prepare static inventory

`ansible.cfg` points to `inventory/inventory.yaml`. The checked-in inventory uses placeholders such as `<MANAGE-NODE-1-IP>`.

Replace those placeholders with the public IPs or DNS names for the instances you manage:

```yaml
all:
  children:
    debian_instances:
      hosts:
        203.0.113.10:
          ansible_user: ubuntu
          ansible_ssh_private_key_file: ./files/ssh_keys/my-ansible-key.pem
    redhat_instances:
      hosts:
        203.0.113.20:
          ansible_user: ec2-user
          ansible_ssh_private_key_file: ./files/ssh_keys/my-ansible-key.pem
```

Validate the inventory before running host-targeted playbooks:

```bash
ansible-inventory --list
ansible all -m ping
```

The ping check requires SSH reachability, security group ingress, correct users, and a usable private key.

## Local acceptance checks

Run these checks before any live AWS operation.

| Check | Command | Expected result | Limitation |
|-------|---------|-----------------|------------|
| Inventory parse | `ansible-inventory --list` | Static inventory renders successfully. | Does not prove hosts are reachable. |
| Syntax | `ansible-playbook playbooks/create_ec2_instances.yaml --syntax-check` | Playbook syntax is valid. | Does not call AWS. |
| Check mode | `ansible-playbook playbooks/create_ec2_instances.yaml --check --vault-password-file vault.pass` | Local prerequisites and non-mutating paths validate. | Requires encrypted variables and local files; cloud behavior is skipped where guarded. |
| Connectivity | `ansible all -m ping` | Managed hosts respond through SSH. | Requires real inventory hosts and network access. |
| Lint, if installed | `ansible-lint playbooks/` | Lint rules pass for the checked-in playbooks used in this guide. | Tool availability depends on local environment. |

Repeat syntax checks for every playbook you intend to run:

```bash
for playbook in playbooks/*.yaml; do
  ansible-playbook "$playbook" --syntax-check
done
```

## Execution sequence

Use this sequence for a live AWS run:

1. `playbooks/create_ec2_instances.yaml`
2. Update `inventory/inventory.yaml` with the created instance addresses if needed.
3. `playbooks/show_gathered_facts.yaml`
4. `playbooks/show_only_os_family_fact.yaml`
5. `playbooks/shutdown_debian_ec2_instances.yaml`
6. `playbooks/shutdown_redhat_ec2_instances.yaml`

### 1. Create EC2 instances

```bash
START_TIME=$(date +%s)

ansible-playbook playbooks/create_ec2_instances.yaml \
  --vault-password-file vault.pass

END_TIME=$(date +%s)
echo "Duration seconds: $((END_TIME - START_TIME))"
```

Capture the resulting instance state from AWS:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=Ansible" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

### 2. Gather facts

```bash
ansible-playbook playbooks/show_gathered_facts.yaml \
  --vault-password-file vault.pass \
  2>&1 | tee execution-output-show-facts.log
```

### 3. Show OS family facts

```bash
ansible-playbook playbooks/show_only_os_family_fact.yaml \
  --vault-password-file vault.pass \
  2>&1 | tee execution-output-os-family.log
```

### 4. Stop Debian instances

```bash
ansible-playbook playbooks/shutdown_debian_ec2_instances.yaml \
  --vault-password-file vault.pass
```

### 5. Stop Red Hat instances

```bash
ansible-playbook playbooks/shutdown_redhat_ec2_instances.yaml \
  --vault-password-file vault.pass
```

Verify final state:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=Ansible" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

## Evidence template

Use this template for local evidence. Keep it out of public docs unless the values were captured from the current code and environment.

```markdown
## Environment
- Date:
- AWS region:
- Ansible version:
- Python version:
- Inventory source: inventory/inventory.yaml

## Pre-flight checks
- ansible-inventory --list:
- syntax checks:
- check mode:
- connectivity:

## EC2 creation
- Duration seconds:
- Instances created or reused:
- Instance IDs:
- Instance types:
- Public IPs recorded in inventory:

## Facts and OS family checks
- Fact-gathering result:
- OS family result:
- Warnings or errors:

## Shutdown checks
- Debian shutdown result:
- Red Hat shutdown result:
- Final AWS state:
```

## Cleanup

The shutdown playbooks stop instances. If the run created temporary infrastructure that should not remain allocated, terminate it explicitly after recording evidence:

```bash
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=Ansible" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text)

aws ec2 terminate-instances --instance-ids $INSTANCE_IDS --output table
aws ec2 wait instance-terminated --instance-ids $INSTANCE_IDS
```

If `INSTANCE_IDS` is empty, stop and inspect the AWS filters before running termination commands.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Vault variables are missing | Confirm `group_vars/all/pass.yml` exists, is encrypted, and contains `ec2_access_key` plus `ec2_secret_key`. |
| Inventory placeholders appear in output | Replace `<MANAGE-NODE-*>` entries in `inventory/inventory.yaml` with real hosts. |
| SSH fails | Check security group ingress, instance state, user names, and private key permissions. |
| AWS access is denied | Verify the credential identity with `aws sts get-caller-identity` and review the EC2 permissions for the requested action. |
| Check mode fails | Check local prerequisites first; `--check` does not remove the need for vault variables, inventory parsing, and file paths. |

## Publication rule

Public documentation should only reference evidence that a reviewer can connect to checked-in files, commands, workflow logs, or attached run output. If a command was not run in the current environment, describe it as a procedure, not as completed evidence.
