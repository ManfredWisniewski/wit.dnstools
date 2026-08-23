---
# wit.dnstools

Manage DNS records through the Netcup DNS API.

## FIX
- `vault_wit_dnstools.netcup_api_key`
- `vault_wit_dnstools.netcup_api_password`

### Optional Variables
- `netcup_endpoint`: API endpoint URL (default: https://ccp.netcup.net/run/webservice/servers/endpoint.php)
- `netcup_domain_export_enabled`: Install the Netcup domain export script and cron job (default: `false`). Requires reseller API access for `listallDomains`.
- `file_target`: Directory for the generated script (default: `/root/wit.ansible`).
- `file_ownership`: Owner/group for the generated files in `owner/group` format (default: `www-data/www-data`).
- `netcup_domain_export_filename`: Generated Markdown filename (default: `domains.md`).
- `netcup_domain_export_output_dir`: Directory for the generated Markdown file (default: `{{ file_target }}`).
- `netcup_domain_export_credentials_file`: External environment file containing `NETCUP_CUSTOMER_NUMBER`, `NETCUP_API_KEY`, and `NETCUP_API_PASSWORD` (default: `/root/netcup.env`).
- `netcup_domain_export_cron_minute`, `netcup_domain_export_cron_hour`, `netcup_domain_export_cron_day`, `netcup_domain_export_cron_month`, `netcup_domain_export_cron_weekday`: Cron schedule fields (default: daily at 03:15).

When domain export is enabled, the role creates a root-owned `0600` credentials file with placeholder values if it does not already exist. Existing values are preserved. Replace the placeholders before the cron job runs. Example format:

```text
NETCUP_CUSTOMER_NUMBER=123456
NETCUP_API_KEY=your-api-key
NETCUP_API_PASSWORD=your-api-password
```

The export uses the Netcup reseller `listallDomains` method, then queries `infoDnsRecords` for every returned domain. Standard Netcup accounts cannot use this domain-discovery method.
- `dns_domain`: Domain to manage. Accepts a single domain string or a list of domains (aliases). When a list is provided, the role iterates over each domain and applies `dns_records_to_update` to all of them.
- `dns_record_name`: Record name to update
- `dns_record_type`: Record type (e.g., 'A')
- `dns_record_value`: Record value

Notes:
- This role always executes on localhost (delegated), so it never attempts to SSH to the target host. This avoids issues when the domain still points to an old server.

## Examples

### Domain export
Enable the scheduled export in `host_vars/<host>/vars.yml`:

```yaml
netcup_domain_export_enabled: true
file_target: "/root/wit.ansible"
file_ownership: "www-data/www-data"
netcup_domain_export_filename: "domains.md"
netcup_domain_export_credentials_file: "/root/wit.ansible/netcup.env"
netcup_domain_export_cron_minute: "15"
netcup_domain_export_cron_hour: "3"
```

The role creates the script at `/root/wit.ansible/export-netcup-domains.py`,
creates the root-owned `0600` credentials file with placeholders, and installs
a daily cron job. Replace the placeholders in `/root/wit.ansible/netcup.env`
before the first run. The generated document is `/root/wit.ansible/domains.md`.

For a reseller account, the export discovers all domains with
`listallDomains` and includes their Netcup DNS records as Markdown table rows.
To apply only the export tasks, use the `domain-export` tag:

```text
ansible-playbook <playbook>.yml --tags domain-export
```

### Single domain
```yaml
dns_domain: "example.com"
dns_records_to_update:
  - hostname: "@"
    type: "A"
    destination: "203.0.113.10"
  - hostname: "www"
    type: "A"
    destination: "203.0.113.10"
  - hostname: "*"
    type: "A"
    destination: "203.0.113.10"
```

### Multiple domains (aliases)
All listed records are applied to each domain in `dns_domain`.
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
  - hostname: "*"
    type: "A"
    destination: "203.0.113.10"
```

## Sources used for research:
https://github.com/couchtyp/certbot-dns-schlundtech
