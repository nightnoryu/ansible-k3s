# Security and networking

## Exposed ports

The UFW role permits:

| Port               | Protocol | Use                             |
|--------------------|----------|---------------------------------|
| inventory SSH port | TCP      | Administration                  |
| 80                 | TCP      | HTTP and ACME HTTP-01 challenge |
| 443                | TCP      | HTTPS ingress                   |
| 6443               | TCP      | Kubernetes API                  |
| 10250              | TCP      | Kubelet API                     |
| 8472               | UDP      | Flannel VXLAN                   |

The provided rules preserve the repository's simple single-node behavior and
allow these ports from any source. On an Internet-facing host, restrict 6443 and
10250 to trusted administrative networks. Port 8472 must never be exposed to
untrusted networks; for a single-node cluster it normally does not need a public
inbound rule. Multi-node clusters should allow it only between nodes.

Review rules after provisioning:

```shell
sudo ufw status verbose
```

Kubernetes networking and UFW forwarding policies can interact. Test pod-to-pod,
pod-to-service, DNS, and ingress traffic after tightening routed traffic rules.

## SSH banning

The fail2ban SSH jail uses the systemd journal, detects the configured Ansible
SSH port, and inserts bans through UFW. A configuration change restarts fail2ban.
Keep a second SSH session open when first applying firewall or jail changes.

## Installer integrity

The K3s installer is downloaded over HTTPS. For reproducible high-assurance
deployments, mirror and audit it, then set `k3s_install_script_url` and
`k3s_install_script_checksum` to the controlled artifact. Pinning `k3s_version`
pins the K3s binary release but not the installer script itself.

## Unsafe sysctls

The configured kubelet allowlist lets workloads request sysctls Kubernetes calls
unsafe. Grant that ability only to trusted workloads and keep the list as narrow
as possible. Admission controls and pod security settings remain separate layers
and should also be reviewed.

## Secrets

Local inventory files are ignored by Git, but Ansible output and controller
backups may still contain addresses or credentials. Store sensitive additions in
Ansible Vault rather than plaintext group variables.
