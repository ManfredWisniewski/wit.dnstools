# wit.dnstools

An Ansible role for managing DNS records for automation with Ansible.
The role discovers the authoritative DNS provider, updates domains by Netcup,
and reports unsupported providers without attempting an update.
Supports the following providers:
- [Netcup](https://ccp.netcup.net/run/webservice/servers/endpoint.php)

## Requirements

- Ansible with the `ansible.builtin.uri`, `ansible.builtin.command`,
  `ansible.builtin.apt`, and `ansible.builtin.pip` modules.
- A Debian-based control host, or the Python dependencies installed by another
  method.
- Netcup API credentials:
  - `netcup_customer_number`
  - `netcup_api_key`
  - `netcup_api_password`
- DNS lookups from the control host using `dig`.

The role installs `python3-requests`, `python3-dnspython`, and `dnsutils` on a
Debian-based control host. Set `dnstools_use_pip: true` to additionally install
`requests` and `dnspython` with pip.

## Credentials

The credential variables should reference encrypted vault variables rather
than containing credentials directly. This repository provides the following
mapping in `group_vars/all/vars.yml`:

```yaml
netcup_customer_number: "{{ vault_netcup_customer_number }}"
netcup_api_key: "{{ vault_netcup_api_key }}"
netcup_api_password: "{{ vault_netcup_api_password }}"
```

The role validates the credentials before making DNS changes. API requests
are delegated to `localhost`; the role does not connect to the managed host.

## Basic usage

Include the role from a playbook and define the domain and desired records in
`host_vars` or `group_vars`:

```yaml
- name: Manage DNS
  hosts: dns_hosts
  gather_facts: false
  roles:
    - wit.dnstools
```

```yaml
dns_domain: "example.com"
dns_records_to_update:
  - hostname: "@"
    type: "A"
    destination: "203.0.113.10"
    ttl: "300"
  - hostname: "www"
    type: "A"
    destination: "203.0.113.10"
    ttl: "300"
  - hostname: "@"
    type: "TXT"
    destination: "google-site-verification=verification-token"
    ttl: "300"
```

The role performs the following steps for each configured domain:

1. Resolve the domain's authoritative nameservers using `dig` and Google DNS
   (`8.8.8.8`).
2. Continue with the Netcup API only when the nameservers indicate Netcup.
3. Log in to the Netcup API and retrieve the current DNS records.
4. Build the desired record set and update Netcup only when applicable.
5. Retrieve the records again for optional visibility and log out.

If the domain is not hosted by Netcup, the role prints a warning and does not
modify DNS records.

## Supported variables

### DNS variables

| Variable | Default | Description |
| --- | --- | --- |
| `dns_domain` | `""` | A domain string or a list of domains. Each domain is processed separately. |
| `dns_records_to_update` | undefined | List of desired DNS records. |
| `netcup_endpoint` | `https://ccp.netcup.net/run/webservice/servers/endpoint.php?JSON` | Netcup API endpoint. |
| `wit_devmode` | `false` | Print additional DNS, API, and credential-length diagnostics. Secrets are not intended to be printed. |
| `dnstools_use_pip` | `false` | Additionally install `requests` and `dnspython` with pip. |

Each item in `dns_records_to_update` supports:

| Field | Default | Description |
| --- | --- | --- |
| `hostname` | `""` | Netcup record name, such as `@`, `www`, or `*`. |
| `type` | `""` | DNS record type, such as `A`, `AAAA`, `CNAME`, `MX`, or `TXT`. |
| `destination` | `""` | Record value. |
| `priority` | `0` | Record priority, where supported by the record type. |
| `ttl` | `300` | Record TTL. |
| `state` | `yes` | Netcup record state. |
| `deleterecord` | `false` | When `true`, delete the matching record identified by hostname, type, and destination. |

Existing records are matched by hostname and type for updates. Records marked
for deletion are matched by hostname, type, and destination. Conflicting A
records are not deleted automatically.

### Multiple domains

When `dns_domain` is a list, the same desired record list is applied to each
domain:

```yaml
dns_domain:
  - example.com
  - example.org

dns_records_to_update:
  - hostname: "@"
    type: "A"
    destination: "203.0.113.10"
  - hostname: "www"
    type: "A"
    destination: "203.0.113.10"
```

## DNS host IP check

Other roles can include the role's `checkhostip` task file without running the
DNS record management workflow:

```yaml
- name: Check that DNS points to hostip
  ansible.builtin.include_role:
    name: wit.dnstools
    tasks_from: checkhostip
```

For a hostname, this resolves the `A` record and compares it with `hostip`.
When `inventory_hostname` is already an IP address, the check is skipped. The
assertion is skipped when `wp_copy` is not configured or is set to `none`.

## Optional domain export

Set `netcup_domain_export_enabled: true` to install a script and a daily cron
job that exports all domains and their DNS records to Markdown. This requires
Netcup reseller API access because it calls `listallDomains`; standard Netcup
accounts cannot use that method.

```yaml
netcup_domain_export_enabled: true
file_target: "/root/wit.ansible"
file_ownership: "www-data/www-data"
netcup_domain_export_filename: "domains.md"
netcup_domain_export_output_dir: "{{ file_target }}"
netcup_domain_export_credentials_file: "{{ file_target }}/netcup.env"
netcup_domain_export_cron_minute: "15"
netcup_domain_export_cron_hour: "3"
```

The export uses these defaults:

| Variable | Default |
| --- | --- |
| `file_target` | `/root/wit.ansible` |
| `file_ownership` | `www-data/www-data` |
| `netcup_domain_export_filename` | `domains.md` |
| `netcup_domain_export_output_dir` | `{{ file_target }}` |
| `netcup_domain_export_file_mode` | `0644` |
| `netcup_domain_export_script_filename` | `export-netcup-domains.py` |
| `netcup_domain_export_credentials_file` | `{{ file_target }}/netcup.env` |
| `netcup_domain_export_cron_minute` | `15` |
| `netcup_domain_export_cron_hour` | `3` |
| `netcup_domain_export_cron_day` | `*` |
| `netcup_domain_export_cron_month` | `*` |
| `netcup_domain_export_cron_weekday` | `*` |

The role creates the following files when export is enabled:

- The executable export script in `file_target`.
- A root-owned credentials file with mode `0600`, created only when it does
  not already exist.
- The Markdown output file with the configured owner, group, and mode.

Populate the credentials file before the first cron execution:

```text
NETCUP_CUSTOMER_NUMBER=123456
NETCUP_API_KEY=your-api-key
NETCUP_API_PASSWORD=your-api-password
```

The export runs once during deployment and then according to the configured
cron schedule. To apply only the export tasks:

```text
ansible-playbook <playbook>.yml --tags domain-export
```

## Execution behavior

- DNS API operations and DNS checks are delegated to `localhost`.
- Domain-export files, the cron job, and the initial export run on the host
  where the role is executed.
- DNS cache flushing is attempted with `resolvectl` or `systemd-resolve` when
  available.
- The role throttles Netcup API operations to one request flow at a time.
- `wit_devmode` enables additional diagnostics and post-update record output.
- The role does not modify domains whose authoritative nameservers are not
  detected as Netcup nameservers.
