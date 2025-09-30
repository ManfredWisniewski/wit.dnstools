---
# wit.dnstools

Manage DNS records through the Netcup DNS API.

## FIX
- `vault_wit_dnstools.netcup_api_key`
- `vault_wit_dnstools.netcup_api_password`

### Optional Variables
- `netcup_endpoint`: API endpoint URL (default: https://ccp.netcup.net/run/webservice/servers/endpoint.php)
- `dns_domain`: Domain to manage. Accepts a single domain string or a list of domains (aliases). When a list is provided, the role iterates over each domain and applies `dns_records_to_update` to all of them.
- `dns_record_name`: Record name to update
- `dns_record_type`: Record type (e.g., 'A')
- `dns_record_value`: Record value

Notes:
- This role always executes on localhost (delegated), so it never attempts to SSH to the target host. This avoids issues when the domain still points to an old server.

With `wit.wordpress`, the DNS check/update is included automatically via `roles/wit.wordpress/tasks/01_dns.yml`.


## Examples

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