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
| `traefik_acme_dns_provider`   | Traefik/lego DNS provider code                             |
| `traefik_acme_dns_resolvers`  | Optional DNS servers for ACME propagation checks           |
| `traefik_acme_dns_credentials` | Provider environment variables stored in a Kubernetes Secret |
| `traefik_acme_wildcard_domains` | Base domains covered by the wildcard certificate          |
| `fail2ban_sshd_bantime`       | Duration of an SSH ban                                    |
| `fail2ban_sshd_findtime`      | Time window in which SSH failures are counted             |
| `fail2ban_sshd_maxretry`      | Failures allowed during the find-time window              |

The example unsafe-sysctl list permits the forwarding and source-mark settings
needed by WARP-style network containers. Allowing a sysctl in kubelet does not
apply it automatically; a workload must still request it in its pod security
context.

The local kubeconfig is mode `0600`. Treat it as an administrative credential.

## Wildcard HTTPS

HTTP-01 is the default. For wildcard HTTPS, set
`traefik_acme_challenge: dns`, set `traefik_acme_dns_provider` to the
Traefik/lego provider for the authoritative DNS zone, and set
`traefik_acme_wildcard_domains` to a list of base domains, such as
`['nightnoryu.com', 'lumiopho.com']`.
`traefik_acme_dns_credentials` maps the provider's environment variable names
to their values. The role creates or updates the `kube-system` Secret named by
`traefik_acme_dns_secret` (default `traefik-dns`) before configuring Traefik.
For Beget, supply `BEGET_USERNAME` and `BEGET_PASSWORD`. Other providers require
different keys; see
[Traefik DNS-01 providers](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/).
Keep credentials outside Git, preferably in Ansible Vault. Secret tasks suppress
their output, including diffs. If credentials are in the ignored local inventory,
restrict that file to mode `0600`. Changing a credential makes the role update
the Secret and restart Traefik so its environment uses the new value.

Start with the example staging CA and `acme-staging.json`. After verifying
issuance, set `traefik_acme_ca_server` to
`https://acme-v02.api.letsencrypt.org/directory` and
`traefik_acme_storage_file: acme.json`. These separate files avoid reusing an
untrusted staging certificate. The chart creates the default TLS store with a
certificate with the first wildcard as its main name and the remaining
wildcards as subject alternative names. Each wildcard covers one level of
subdomains. Set wildcard DNS A/AAAA records for each base domain to the ingress
IP if new names should resolve without individual DNS changes. Each new service
needs an Ingress with TLS enabled. Omit the `router.tls.certresolver` annotation
to use the shared wildcard certificate. The base domain itself is not covered.

With Beget, create the `_acme-challenge` subdomain for every base domain in the
Beget account before the first run. The subdomains must remain in the account
for renewals. The role does not manage DNS records.
If Kubernetes DNS caches an earlier TXT answer, set
`traefik_acme_dns_resolvers: ['ns1.beget.com:53', 'ns2.beget.com:53']` so
Traefik checks the published challenge against Beget's authoritative servers.
This affects ACME checks only.

The HelmChartConfig pins one Deployment replica, uses Recreate updates and a
persistent `/data` volume. Keep this PVC on upgrades and back it up manually as
a private key. Do not run multiple replicas sharing it.
HTTP redirects to HTTPS; HTTPS request reads allow five minutes for direct photo
uploads. Access logs omit paths, queries and headers to avoid logging signatures.

Set the DNS credential variables before running `playbooks/setup-traefik-acme.yml`.
Check issuance with staging, then switch to production and verify the wildcard
certificate and an existing service. Keep the production ACME file on the PVC
so Traefik can renew the certificate after restarts.
