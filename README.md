# wit.dnstools

An Ansible role for managing DNS records for automation with Ansible.
The role discovers the authoritative DNS provider, updates domains by Netcup,
and reports unsupported providers without attempting an update.
Supports the following providers:
- [Netcup](https://ccp.netcup.net/run/webservice/servers/endpoint.php)

## Requirements

- An Ansible execution environment with the modules used by the role,
  including `uri`, `command`, `apt`, `pip`, `stat`, `set_fact`, `assert`, and
  `debug`.
- A control host reachable as `localhost` from the play.
- `dig` available on the control host for nameserver discovery.
- Netcup API credentials when a configured domain is detected as Netcup-hosted.
- Permission to install packages on the control host when the role installs its
  dependencies or when `checkhostip` is included.

The normal role entry point installs these Debian packages on a Debian-based
control host:

- `python3-requests`
- `python3-dnspython`
- `dnsutils`

Set `dnstools_use_pip: true` to additionally install `requests` and `dnspython`
with pip using `--break-system-packages`. The package installation task is
skipped on non-Debian control hosts, but the role still expects its runtime
requirements to be available there.

The separate `checkhostip` task file installs `python3-dnspython` with `apt`
regardless of the Debian detection used by the normal entry point.

## Credentials

The DNS-management tasks require these variables when a domain is detected as
Netcup-hosted:

- `netcup_customer_number`
- `netcup_api_key`
- `netcup_api_password`

Store the values in Ansible Vault and reference them from inventory variables.
For example, this repository maps them in `group_vars/all/vars.yml`:

```yaml
netcup_customer_number: "{{ vault_netcup_customer_number }}"
netcup_api_key: "{{ vault_netcup_api_key }}"
netcup_api_password: "{{ vault_netcup_api_password }}"
```

The role validates that the credentials are non-empty before making Netcup DNS
changes. The API tasks are delegated to `localhost`.

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

The role performs these operations for each value in `dns_domain`:

1. Normalize `dns_domain` to a list.
2. Query `dig @8.8.8.8 +short NS <domain>` on `localhost`.
3. Include the Netcup task file only when the nameserver output contains
   `netcup.net`.
4. Log in to the Netcup API and retrieve the existing DNS records.
5. Build the `updateDnsRecords` payload from `dns_records_to_update`.
6. Submit the DNS record set, fetch the records again, and log out.

When no supported provider is detected, the role prints a warning and does not
perform a DNS update.

## Variables

### DNS-management variables

| Variable | Default in role | Description |
| --- | --- | --- |
| `dns_domain` | `""` | A domain string or an iterable such as a list of domains. An empty value produces no domain checks. |
| `dns_records_to_update` | Not defined | Desired DNS records. The update path is only built when this variable is defined. |
| `netcup_customer_number` | `""` | Netcup customer number. |
| `netcup_api_key` | `""` | Netcup API key. |
| `netcup_api_password` | `""` | Netcup API password. |
| `netcup_endpoint` | `https://ccp.netcup.net/run/webservice/servers/endpoint.php?JSON` | Netcup API endpoint. |
| `dnstools_use_pip` | Not defined | When true, install `requests` and `dnspython` with pip. The tasks use `default(false)`. |
| `wit_devmode` | Not defined | When true, print additional diagnostics and post-update records. The tasks use `default(false)`. |

### DNS record structure

Each item in `dns_records_to_update` is converted into a Netcup record object:

| Field | Effective default | Description |
| --- | --- | --- |
| `hostname` | `""` | Record name, for example `@`, `www`, or `*`. |
| `type` | `""` | Record type. The role uppercases this value before sending it. |
| `destination` | `""` | Record value. Entries with an empty value are removed from the final payload. |
| `priority` | `0` | Record priority. |
| `ttl` | `300` | Record TTL. |
| `deleterecord` | `false` | When the string value, lowercased, equals `true`, mark the record for deletion. |
| `state` | `yes` | Record state sent to Netcup. |

Example for several record types:

```yaml
dns_records_to_update:
  - hostname: "@"
    type: "A"
    destination: "203.0.113.10"
  - hostname: "mail"
    type: "CNAME"
    destination: "mail.example.net"
  - hostname: "@"
    type: "TXT"
    destination: "google-site-verification=verification-token"
  - hostname: "@"
    type: "MX"
    destination: "mail.example.net"
    priority: "10"
```

The role does not define a closed list of DNS record types. It passes the
configured `type` to the Netcup API after uppercasing it; the API determines
whether the record type and its fields are accepted.

### Multiple domains

The role accepts a list of domains and processes each domain separately with
the same desired record list:

```yaml
dns_domain:
  - example.com
  - example.org

dns_records_to_update:
  - hostname: "@"
    type: "A"
    destination: "203.0.113.10"
```

## Matching, idempotency, and deletion

Before constructing the update payload, the role retrieves the existing
Netcup records.

- A normal record is matched by `hostname` and uppercased `type`; the matching
  record ID is attached to the update object.
- A record with `deleterecord` set to `true` is matched by `hostname`, type, and
  `destination` before its ID is attached.
- Records without a matching ID are sent without an ID so Netcup can create
  them.
- The final payload excludes records with an empty hostname, type, or
  destination.
- Existing conflicting A records are only reported by a debug task and are not
  automatically deleted.
- The API update task is marked changed when Netcup returns a successful
  response; the role does not independently prove that every desired record is
  unchanged before calling `updateDnsRecords`.

The role performs a fresh login immediately before the update request. API
requests use `throttle: 1`. A one-second pause follows the initial login, and a
second login is attempted if the initial session ID is empty.

## Debug behavior and security notes

Set `wit_devmode: true` to show nameserver results, normalized record data,
credential lengths, partial API-key/session previews, API results, and records
read after the update.

The source contains commented-out `no_log: true` lines around API calls rather
than active `no_log` protection. Do not enable development mode for sensitive
runs without reviewing the resulting output, and do not add credentials to
inventory files outside encrypted variables.

## DNS host IP check

Other roles in this repository include the role's `checkhostip` task file
without running the DNS-management workflow:

```yaml
- name: Check that DNS points to hostip
  ansible.builtin.include_role:
    name: wit.dnstools
    tasks_from: checkhostip
```

The task file:

1. Installs `python3-dnspython` on `localhost` with `apt` and privilege
   escalation.
2. Skips the check when `inventory_hostname` matches the task's IPv4 regular
   expression.
3. Otherwise resolves `inventory_hostname + "/A"` with the Ansible `dig`
   lookup.
4. Asserts that the result equals `hostip` when `hostip` is defined and
   `wp_copy` is unset or equals `none`.

## Optional domain export

Set `netcup_domain_export_enabled: true` to include `tasks/domain-export.yml`.
This feature requires Netcup reseller API access because the generated script
calls `listallDomains`; standard Netcup accounts cannot use that method.

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

| Variable | Default |
| --- | --- |
| `netcup_domain_export_enabled` | `false` |
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

When enabled, the tasks run on the play's normal execution host and:

- Create `file_target` as root-owned `0755`.
- Create the output directory if necessary without explicitly setting its
  owner or mode.
- Create the credentials file only if it does not already exist, with root
  ownership and mode `0600`.
- Install the generated script at
  `file_target/netcup_domain_export_script_filename` with mode `0750` and the
  owner/group parsed from `file_ownership`.
- Create or touch the Markdown output file with the configured owner, group,
  and mode.
- Install a root cron job using the configured schedule.
- Run the generated script once immediately during deployment.

The credentials file must contain these variables before the initial export
run succeeds:

```text
NETCUP_CUSTOMER_NUMBER=123456
NETCUP_API_KEY=your-api-key
NETCUP_API_PASSWORD=your-api-password
```

The role creates the placeholder file with `CHANGE_ME` values when the file is
absent, but it does not replace an existing file. Because the script is run
immediately, enabling export without first providing valid credentials causes
the initial export command to fail.

The generated Markdown contains a wildcard-domain summary and a detailed table
of the discovered domains and records. The script logs in, calls
`listallDomains`, calls `infoDnsRecords` for every returned domain, writes the
output through a temporary file, replaces the destination atomically, applies
the configured mode, and logs out in a `finally` block.

The export tasks are tagged `domain-export`:

```text
ansible-playbook <playbook>.yml --tags domain-export
```

## Execution and task structure

- `tasks/main.yml` is the default entry point.
- `tasks/main.yml` includes `requirements.yml` and conditionally includes
  `netcup.yml` for domains whose nameserver output contains `netcup.net`.
- `tasks/main.yml` conditionally includes `domain-export.yml` with the
  `domain-export` tag.
- DNS discovery and Netcup API operations are delegated to `localhost`.
- Domain-export file, cron, and command tasks are not explicitly delegated.
- DNS cache flushing with `resolvectl` or `systemd-resolve` is attempted on
  `localhost` and ignores errors.
- The role has no handlers, plugins, filters, lookup files, meta files, or
  role-local test files in this repository.

## Troubleshooting and limitations

- Confirm that `dig @8.8.8.8 +short NS <domain>` returns nameservers containing
  `netcup.net`; otherwise the role deliberately skips the update.
- Confirm that the three Netcup credentials are non-empty and available to the
  delegated localhost context.
- Confirm that the endpoint is reachable and that the API account is allowed
  to manage the domain.
- Domain export requires reseller access to `listallDomains`.
- The role has a TODO in `tasks/netcup.yml` concerning use with a single
  `dns_domain`; verify behavior before changing that path.
- No provider implementation other than the Netcup nameserver/API path is
  present in the role.

## Safe validation

The following checks do not execute a playbook:

```text
ansible-lint roles/wit.dnstools
yamllint roles/wit.dnstools
markdownlint roles/wit.dnstools/README.md
```

Run syntax or lint checks in the repository's configured environment and review
any pre-existing violations separately from documentation changes.
