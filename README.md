# Ansible K3s

Ansible automation for provisioning and maintaining a secure, single-server K3s
control plane for self-hosted workloads.

The initial setup installs operating-system updates, UFW, fail2ban, and K3s.
The separate Traefik playbook configures ACME and manages DNS credentials when
DNS-01 is selected.

## ✨ Features

- Pinned K3s installation and idempotent upgrades
- Persistent, inventory-defined Kubernetes node names
- Kubelet unsafe sysctls required by networking workloads such as WARP
- UFW firewall rules and fail2ban protection for SSH
- Traefik Let's Encrypt DNS challenge with multiple wildcard domains
- JSON access logs with requester IP preservation
- Automatic host reboots when package upgrades require one

## 🛠️ Quick start

Requirements: a Debian-based server with Python 3, SSH access through root or a
passwordless sudo user, and [mise](https://mise.jdx.dev/) on the controller.
The project pins uv, ansible-core, yamllint, and ansible-lint in `mise.toml`;
mise uses uv to install the Python tools.

```shell
git clone https://github.com/nightnoryu/ansible-k3s
cd ansible-k3s

mise install
mise run collections

cp inventory/hosts.example.yml inventory/hosts.yml
cp inventory/group_vars/all.example.yml inventory/group_vars/all.yml
$EDITOR inventory/hosts.yml inventory/group_vars/all.yml

mise run ping
mise run setup
# Reconcile Traefik ACME settings after setting its variables.
mise run setup-traefik-acme
```

Every host must have a stable, unique `k3s_node_name`. Review the firewall ports
and set `traefik_acme_email`. For DNS-01, also set the DNS provider and credential
variables before running the Traefik playbook. Generated inventory files are
ignored by Git so local addresses and account details are not committed.

## 🔄 Operations

```shell
# Upgrade system packages and reboot only when required
mise run update-system

# Reconcile the full initial configuration
mise run setup

# Reconcile only Traefik and ACME settings
mise run setup-traefik-acme

# Upgrade K3s to k3s_version from group_vars
mise run update-k3s
```

Run the same YAML and Ansible checks as CI with `mise check`.

Playbook tasks use `inventory` and `--diff` by default. Pass additional Ansible
options after the task name, for example `mise run setup --check` or
`mise run setup --tags fail2ban`.

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
spec:
  ingressClassName: traefik
  rules:
    - host: your-service.example.com
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
        - your-service.example.com
```

## 🎯 Scope

The supplied inventory and playbooks target a single K3s server. They are not a
multi-server/agent bootstrap solution; adding HA requires server tokens,
cluster-init/join settings, and tighter node-to-node firewall rules.

## 📜 License

Distributed under the MIT License. See [License](/LICENSE) for more information.
