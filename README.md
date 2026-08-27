# Ansible k3s

Platform for automated k3s cluster management.

## How to use

### Prerequisites

1. Debian-based VM
2. SSH access with either root or sudo and NOPASSWD

### Running playbooks

```shell
git clone https://github.com/nightnoryu/ansible-k3s
cd ansible-k3s

# Set your VM IPs
cp inventory/hosts.example.yml inventory/hosts.yml
$EDITOR inventory/hosts.yml

# Check connectivity
ansible all -m ping -i inventory

# Use ansible for operations
ansible-playbook playbooks/update-system.yml -i inventory --diff       # update system packages
ansible-playbook playbooks/setup-k3s.yml -i inventory --diff           # setup k3s node
ansible-playbook playbooks/setup-traefik-acme.yml -i inventory --diff  # setup ACME certificate resolver
ansible-playbook playbooks/update-k3s.yml -i inventory --diff          # update k3s version
```
