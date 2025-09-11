# wit.dnstools

Manage DNS records through the Netcup DNS API.

## FIX
#FIX: Nameserver resolution still not correct. (rootserver)

## TODO
#TODO: Handle MX Servers
#TODO: Add Antispam support
#TODO: Handle SPF Entries
#TODO: Handle DMARC Entries
#TODO: Handle google site verification (still valid?)
#TODO: Handle subdomains

## Requirements

- Python packages:
  - requests
  - dnspython

## Role Variables

### Required Variables (stored in vault)
- `vault_wit_dnstools.netcup_customer_number`
- `vault_wit_dnstools.netcup_api_key`
- `vault_wit_dnstools.netcup_api_password`

### Optional Variables
- `netcup_endpoint`: API endpoint URL (default: https://ccp.netcup.net/run/webservice/servers/endpoint.php)
- `dns_domain`: Domain to manage
- `dns_record_name`: Record name to update
- `dns_record_type`: Record type (e.g., 'A')
- `dns_record_value`: Record value

Notes:
- This role always executes on localhost (delegated), so it never attempts to SSH to the target host. This avoids issues when the domain still points to an old server.

With `wit.wordpress`, the DNS check/update is included automatically via `roles/wit.wordpress/tasks/01_dns.yml`.


## Sources used for research:
https://github.com/couchtyp/certbot-dns-schlundtech