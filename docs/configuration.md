# Configuration

Copy both example inventory files before running a playbook:

```shell
cp inventory/hosts.example.yml inventory/hosts.yml
cp inventory/group_vars/all.example.yml inventory/group_vars/all.yml
```

## Host variables

Set these under each host in `inventory/hosts.yml`:

| Variable        | Required | Purpose                                                         |
|-----------------|----------|-----------------------------------------------------------------|
| `ansible_host`  | yes      | IP address or DNS name used for SSH and kubeconfig              |
| `ansible_port`  | no       | SSH port; defaults to `22` and is also used by UFW and fail2ban |
| `ansible_user`  | no       | Remote SSH user                                                 |
| `k3s_node_name` | yes      | Stable, unique Kubernetes node name                             |

Do not change `k3s_node_name` after workloads have been scheduled. K3s stores it
in `/etc/rancher/k3s/config.yaml`; changing it makes Kubernetes see a different
node and leaves the old node object behind.

Example:

```yaml
all:
  hosts:
    control-plane:
      ansible_host: 203.0.113.10
      ansible_user: deploy
      ansible_port: 2222
      k3s_node_name: edge-01
```

## Group variables

| Variable                      | Purpose                                                   |
|-------------------------------|-----------------------------------------------------------|
| `k3s_version`                 | Exact K3s release installed by setup and update playbooks |
| `k3s_install_script_url`      | Installer URL; defaults to the official K3s endpoint      |
| `k3s_install_script_checksum` | Optional Ansible checksum such as `sha256:...`            |
| `k3s_bin_path`                | Installed K3s binary path                                 |
| `k3s_kubeconfig_remote_path`  | Kubeconfig path on the server                             |
| `k3s_kubeconfig_local_path`   | Destination on the Ansible controller                     |
| `k3s_api_endpoint`            | Optional kubeconfig address; defaults to `ansible_host`   |
| `k3s_allowed_unsafe_sysctls`  | Sysctls admitted by kubelet for requesting pods           |
| `traefik_acme_email`          | Let's Encrypt account email                               |
| `fail2ban_sshd_bantime`       | Duration of an SSH ban                                    |
| `fail2ban_sshd_findtime`      | Time window in which SSH failures are counted             |
| `fail2ban_sshd_maxretry`      | Failures allowed during the find-time window              |

The example unsafe-sysctl list permits the forwarding and source-mark settings
needed by WARP-style network containers. Allowing a sysctl in kubelet does not
apply it automatically; a workload must still request it in its pod security
context.

The local kubeconfig is mode `0600`. Treat it as an administrative credential.
