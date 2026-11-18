# blog-bind-rpz

Companion code for the blog post
[Implementing DNS-Based Security with BIND Response Policy Zones](https://blog.bsdserver.nl/implementing-dns-based-security-with-bind-response-policy-zones/).

It builds a BIND recursive resolver that filters DNS with Response Policy Zones, as the post describes:

- `response-policy` with `qname-wait-recurse no` and three zones, evaluated in order:
  - `rpz-whitelist` (`policy passthru`): the post's whitelist entries
  - `rpz.local`: the post's rules (NXDOMAIN, wildcard, NODATA, local data to the block page address,
    passthru for a subdomain, CNAME to an info page)
  - `rpz-blocklist`: an external hosts-format blocklist (Steven Black's hosts by default)
- `convert-blocklist.sh` and `update-rpz.sh` in `/usr/local/bin`, scheduled daily at 03:00 with cron,
  and run once during the first deployment
- an `rpz` log channel in `/var/log/named/rpz.log`
- optionally the nginx block page from the post (`bind_rpz_blockpage_enabled: true`)

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04 (tested) or RHEL/Rocky 9. RHEL uses the post's paths
  (`/etc/named.conf`, `/var/named/rpz`); Debian/Ubuntu use `/etc/bind/named.conf.options` and `/var/lib/bind/rpz`.
- Outbound DNS from the resolver, and HTTPS to the blocklist URL

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configure BIND
cd ../ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Run the post's checks (NXDOMAIN, NODATA, local data, CNAME, passthru, blocklist, rpz.log, cron)
ansible-playbook ../tests/verify.yml
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

BIND applies RPZ changes at most once per `min-update-interval` (60 seconds by default), so a new rule or a
refreshed blocklist can take up to a minute to take effect.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `bind_rpz_trusted_networks` | localhost, localnets, 192.168.0.0/16 | Members of `acl "trusted"` |
| `bind_rpz_zones` | whitelist, rpz.local and blocklist from the post | RPZ zones in order, with policy and records |
| `bind_rpz_qname_wait_recurse` | `false` | `qname-wait-recurse` |
| `bind_rpz_blocklist_enabled` | `true` | Install the blocklist scripts and the cron job |
| `bind_rpz_blocklist_url` | Steven Black's unified hosts | Hosts-format blocklist |
| `bind_rpz_blocklist_cron_hour` / `_minute` | `3` / `0` | Daily update time |
| `bind_rpz_blocklist_initial_update` | `true` | Load the blocklist during the first deployment |
| `bind_rpz_blockpage_enabled` | `false` | Install nginx with the block page |
| `bind_rpz_blockpage_server_name` | `blocked.example.com` | Block page `server_name` |
| `bind_install_source` | `distro` | `distro` (BIND 9.18 on Ubuntu 24.04) or `isc` (ppa:isc/bind, BIND 9.20) |

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND
- `ansible/roles/bind_rpz` writes the RPZ configuration, zones, scripts, cron job and optional block page
- `tests/verify.yml` runs the functional checks from the post

## License

MIT
