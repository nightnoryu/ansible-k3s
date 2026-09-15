# Ansible K3s

Ansible automation for provisioning and maintaining a secure, single-server K3s
control plane for self-hosted workloads.

The initial setup installs operating-system updates, UFW, fail2ban, K3s, and a
Traefik ACME configuration. It also downloads a ready-to-use kubeconfig to the
Ansible controller.

## ✨ Features

- Pinned K3s installation and idempotent upgrades
- Persistent, inventory-defined Kubernetes node names
- Kubelet unsafe sysctls required by networking workloads such as WARP
- UFW firewall rules and fail2ban protection for SSH
- Traefik Let's Encrypt HTTP challenge support
- JSON access logs with requester IP preservation
- Automatic host reboots when package upgrades require one

## 🛠️ Quick start

Requirements: a Debian-based server with Python 3, SSH access through root or a
passwordless sudo user, and Ansible on the controller.

```shell
git clone https://github.com/nightnoryu/ansible-k3s
cd ansible-k3s

ansible-galaxy collection install -r requirements.yml

cp inventory/hosts.example.yml inventory/hosts.yml
cp inventory/group_vars/all.example.yml inventory/group_vars/all.yml
$EDITOR inventory/hosts.yml inventory/group_vars/all.yml

ansible all -m ping -i inventory
ansible-playbook playbooks/setup-k3s.yml -i inventory --diff
```

Every host must have a stable, unique `k3s_node_name`. Review the firewall ports
and set `traefik_acme_email` before the first run. Generated inventory files are
ignored by Git so local addresses and account details are not committed.

## 🔄 Operations

```shell
# Upgrade system packages and reboot only when required
ansible-playbook playbooks/update-system.yml -i inventory --diff

# Reconcile the full initial configuration
ansible-playbook playbooks/setup-k3s.yml -i inventory --diff

# Reconcile only Traefik and ACME settings
ansible-playbook playbooks/setup-traefik-acme.yml -i inventory --diff

# Upgrade K3s to k3s_version from group_vars
ansible-playbook playbooks/update-k3s.yml -i inventory --diff
```

Detailed documentation:

- [Configuration](docs/configuration.md)
- [Operations and upgrades](docs/operations.md)
- [Security and networking](docs/security.md)

## 🔐 ACME-enabled Ingress example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: your-service
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls.certresolver: letsencrypt
spec:
  ingressClassName: traefik
  rules:
    - host: your-domain.example
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: your-service
                port:
                  number: 3000
  tls:
    - hosts:
        - your-domain.example
```

## 🎯 Scope

The supplied inventory and playbooks target a single K3s server. They are not a
multi-server/agent bootstrap solution; adding HA requires server tokens,
cluster-init/join settings, and tighter node-to-node firewall rules.
