# Operations

## Initial provisioning

Install the required collection and validate connectivity first:

```shell
ansible-galaxy collection install -r requirements.yml
ansible all -m ping -i inventory
ansible-playbook playbooks/setup-k3s.yml -i inventory --check --diff
ansible-playbook playbooks/setup-k3s.yml -i inventory --diff
```

The setup order is deliberate:

1. Upgrade packages and reboot if required.
2. Open required firewall ports before enabling UFW.
3. Install fail2ban and connect its SSH jail to UFW.
4. Write persistent K3s configuration and install K3s.
5. Configure bundled Traefik and wait for its rollout.

An initial check-mode run cannot validate resources that do not exist yet, but
it is still useful for inventory and variable validation.

## Updating K3s

Set a tested `k3s_version` in `inventory/group_vars/all.yml`, then run:

```shell
ansible-playbook playbooks/update-k3s.yml -i inventory --check --diff
ansible-playbook playbooks/update-k3s.yml -i inventory --diff
```

The update role compares the requested version with `k3s --version` and invokes
the installer only when they differ. It passes the same kubelet arguments as the
initial installation, preventing the systemd service from losing them during an
upgrade.

Back up workloads and K3s state before significant upgrades. Review upstream
release notes and avoid skipping unsupported Kubernetes minor-version steps.

## Useful checks

```shell
sudo systemctl status k3s fail2ban ufw
sudo fail2ban-client status sshd
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl -n kube-system get pods
sudo k3s kubectl -n kube-system logs deployment/traefik
```

Traefik access logs are JSON records. `externalTrafficPolicy: Local` preserves
the source address presented to Traefik instead of replacing it with a node
address.
