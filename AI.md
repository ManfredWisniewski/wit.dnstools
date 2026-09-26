# wit.dnstools: AI implementation guide

This file is operational guidance for AI agents modifying this role. It is not a
replacement for reading the task files. The implementation is the source of
truth; do not infer behavior from variable names or README examples.

## Role boundary

The role currently implements one DNS-provider path:

- Discover authoritative nameservers with `dig @8.8.8.8 +short NS <domain>`.
- Treat the domain as Netcup-managed when the nameserver output contains
  `netcup.net`.
- Use the Netcup JSON API to read and update DNS records.

There is no second provider implementation in the role. Do not document or add
provider support without adding and tracing an implementation.

The role also has two separate capabilities:

- `tasks/checkhostip.yml`: resolve a host A record and compare it with
  `hostip`.
- `tasks/domain-export.yml`: install and run a Netcup reseller domain-export
  script when explicitly enabled.

## Source of truth map

Read these files before making related changes:

| Behavior | Source |
| --- | --- |
| Default variables | `defaults/main.yml` |
| Default entry point and provider detection | `tasks/main.yml` |
| Package installation | `tasks/requirements.yml` |
| Netcup login, record matching, update, and logout | `tasks/netcup.yml` |
| Host A-record check | `tasks/checkhostip.yml` |
| Domain-export files, cron, and initial run | `tasks/domain-export.yml` |
| Generated export script behavior | `templates/export-netcup-domains.py.j2` |
| Repository invocation and inventory shaping | Parent repository playbooks, `host_vars`, and `group_vars` |
| Local contribution rules | `.windsurfrules` |

There are currently no role-local `vars/main.yml`, handlers, meta files,
plugins, filter plugins, lookup files, molecule files, or role-local tests.
Verify this inventory again before assuming a new file does not exist.

## Entry points and task graph

### Default role entry point

`tasks/main.yml` runs in this order:

1. Include `requirements.yml`.
2. Perform an early login only when all three Netcup credential variables are
   non-empty.
3. Detect Debian and available DNS cache flush commands on `localhost`.
4. Attempt DNS cache flushing on `localhost`; these commands ignore errors.
5. Normalize `dns_domain` into `dns_domain_list`.
6. Query nameservers for every domain using `dig @8.8.8.8 +short NS`.
7. Include `netcup.yml` per domain when the nameserver output contains
   `netcup.net`.
8. Build optional change/deletion summaries.
9. Configure domain export when `netcup_domain_export_enabled` is true.

The default entry point does not have a role-level tag for DNS management.
Only the domain-export include has the `domain-export` tag.

### Direct task-file entry point

Other roles include `tasks/checkhostip.yml` directly with:

```yaml
ansible.builtin.include_role:
  name: wit.dnstools
  tasks_from: checkhostip
```

Do not assume that including `checkhostip` also runs `tasks/main.yml`.

## Inputs and defaults

Defaults are defined only in `defaults/main.yml`:

| Variable | Default | Notes |
| --- | --- | --- |
| `netcup_customer_number` | `""` | Required by `netcup.yml`. Usually supplied through an encrypted inventory variable. |
| `netcup_api_key` | `""` | Required by `netcup.yml`. Usually supplied through an encrypted inventory variable. |
| `netcup_api_password` | `""` | Required by `netcup.yml`. Usually supplied through an encrypted inventory variable. |
| `netcup_endpoint` | `https://ccp.netcup.net/run/webservice/servers/endpoint.php?JSON` | Used by all Netcup API calls and rendered into the export script. |
| `netcup_domain_export_enabled` | `false` | Gates `domain-export.yml`. |
| `file_target` | `/root/wit.ansible` | Domain-export script directory. |
| `file_ownership` | `www-data/www-data` | Split on `/` for export script and output ownership. |
| `netcup_domain_export_filename` | `domains.md` | Export Markdown filename. |
| `netcup_domain_export_output_dir` | `{{ file_target }}` | Export output directory. |
| `netcup_domain_export_file_mode` | `0644` | Export Markdown mode. |
| `netcup_domain_export_script_filename` | `export-netcup-domains.py` | Generated script filename. |
| `netcup_domain_export_credentials_file` | `{{ file_target }}/netcup.env` | Root-owned environment file path. |
| `netcup_domain_export_cron_minute` | `15` | Cron minute. |
| `netcup_domain_export_cron_hour` | `3` | Cron hour. |
| `netcup_domain_export_cron_day` | `*` | Cron day. |
| `netcup_domain_export_cron_month` | `*` | Cron month. |
| `netcup_domain_export_cron_weekday` | `*` | Cron weekday. |
| `dns_domain` | `""` | String or iterable of domains. Empty values result in no DNS checks. |

`dns_records_to_update`, `dnstools_use_pip`, and `wit_devmode` are not defined
in role defaults. Tasks use `default(...)` for the latter two, and the record
list is optional in some paths. Never describe them as default variables unless
that changes in `defaults/main.yml`.

Parent repository inventory may override these values. Inspect actual
`group_vars` and `host_vars` before changing precedence assumptions.

## DNS record input contract

`dns_records_to_update` is a list of dictionaries consumed by `tasks/netcup.yml`.
The implementation reads these fields:

- `hostname`, default `""`
- `type`, default `""`, uppercased before use
- `destination`, default `""`
- `priority`, default `0`
- `ttl`, default `300`
- `deleterecord`, default string `false`, compared after lowercasing to `true`
- `state`, default `yes`

The final payload rejects entries with empty `hostname`, `type`, or
`destination`. The code does not validate a fixed set of record types; the
Netcup API receives the uppercased type and decides whether it is valid.

Do not introduce a new record field in documentation without tracing it through
the `tmp_update` object and the API payload. Do not assume `state` is a boolean;
the current task uses the configured value with a `yes` default.

## Netcup API flow

`tasks/netcup.yml` performs these operations, all delegated to `localhost`:

1. Assert the three credentials are non-empty.
2. Remove whitespace from the credential values using `regex_replace` and
   `trim`.
3. Login with the Netcup `login` action.
4. Pause for one second after a successful login.
5. Save and normalize the API session ID.
6. Assert that the session ID is non-empty.
7. Re-login if the normalized session ID is empty.
8. Call `infoDnsRecords` for `dns_domain`.
9. Extract `responsedata.dnsrecords`.
10. Build `netcup_update_records` from the desired records.
11. Login again immediately before `updateDnsRecords`.
12. Submit `netcup_dnsrecords`.
13. Fetch records again for optional diagnostics.
14. Logout when a session ID exists.

API calls use JSON POST requests and check the response `status` for success.
Most API calls use `throttle: 1`. The update task reports changed when its JSON
response has a successful status and fails for unsuccessful API responses.

The source contains commented `no_log: true` lines rather than active `no_log`
settings. Treat API output as sensitive. Do not add credential logging. The
existing development diagnostics expose lengths and partial previews; do not
expand those previews.

## Matching and deletion behavior

When building an update object:

- Normal records find an existing ID by `hostname` and uppercased `type`.
- Deletion records find an existing ID by `hostname`, type, and exact
  `destination`.
- A matching ID is included; otherwise the object is sent without an ID.
- `deleterecord` is converted to a boolean in `tmp_update`.
- Empty hostname, type, and destination entries are removed from the final
  `netcup_dnsrecords` list.

The stale-duplicate task only logs a message for an existing `@` A record whose
value differs from the first desired `@` A record. It does not delete that
record. Preserve this safety behavior unless the user explicitly requests a
functional change and the matching logic is redesigned and tested.

The role does not independently skip the `updateDnsRecords` request when all
records are already identical. It constructs the payload and calls the API
when the required variables and successful initial login conditions are met.
Do not call this full idempotency without verifying the Netcup API semantics.

## Delegation and side effects

### Delegated to localhost

The normal DNS path delegates package detection/install, cache detection and
flushing, nameserver lookup, API calls, facts, assertions, and debug output to
`localhost`. Cache flush commands use privilege escalation and ignore errors.

`checkhostip.yml` delegates its package installation to `localhost` with
privilege escalation. Its `set_fact`, DNS lookup, and assertion are not
explicitly delegated in that file and therefore follow normal Ansible task
execution semantics.

### Not explicitly delegated

`domain-export.yml` does not set `delegate_to`. Its directories, generated
script, output file, cron job, and initial command run on the play's normal
execution host. Do not document these as localhost operations unless the
calling play itself runs locally.

The export feature has these side effects:

- Creates `file_target` as root/root `0755`.
- Creates the output directory without explicit owner or mode.
- Creates the credentials file only when absent, as root/root `0600`.
- Installs the generated script with mode `0750` and ownership parsed from
  `file_ownership`.
- Touches the output Markdown file with the configured ownership and mode.
- Installs a root cron job using the configured schedule.
- Executes the generated script immediately with `/usr/bin/python3`.

## Domain-export script contract

The Jinja template renders a standalone Python script using only Python
standard-library modules. It reads these environment variables from the
configured credentials file:

- `NETCUP_CUSTOMER_NUMBER`
- `NETCUP_API_KEY`
- `NETCUP_API_PASSWORD`

The script optionally accepts `NETCUP_ENDPOINT` and `NETCUP_ENV_FILE` from its
process environment. It validates the required credential keys, logs into the
API, calls reseller `listallDomains`, calls `infoDnsRecords` for each returned
domain, logs out in `finally`, renders Markdown, writes through a temporary
file, replaces the output, applies the configured mode, and changes ownership.

The placeholder credentials task uses `force: false`, so existing credentials
are preserved. However, the role runs the script immediately after creating the
placeholder file. Enabling export without valid credentials therefore causes
the initial export command to fail. This is an implementation behavior, not an
assumption.

The generated Markdown includes:

- A wildcard-only summary table of domain and destination.
- A detailed table containing domain, hostname, type, destination, priority,
  TTL, and state for every returned record.

## Safe modification guidance

Before changing a behavior:

1. Read the task file that owns it and every file it includes or renders.
2. Inspect parent inventory and playbooks for overrides and reshaping.
3. Preserve role-local comments, especially TODOs and security-related notes.
4. Keep credentials in Vault-backed variables or the protected export
   credentials file; never place real values in defaults, examples, or logs.
5. Preserve `delegate_to`, `become`, `throttle`, `when`, and `failed_when`
   semantics unless the requested change explicitly concerns them.
6. If adding a provider, add provider detection, credentials/API tasks,
   matching/update behavior, tests or safe validation, and documentation as a
   coherent feature. Do not merely add a provider name to the README.
7. If adding a record field, trace it from inventory input to the final API
   request and verify how existing records are matched.
8. If changing deletion behavior, require an explicit safety review; the
   current implementation deliberately avoids deleting conflicting A records.
9. Do not run an Ansible playbook during documentation or code review work.

## Safe verification

Use only checks that do not execute a playbook, for example:

```text
ansible-lint roles/wit.dnstools
yamllint roles/wit.dnstools
markdownlint roles/wit.dnstools/README.md
```

Also review the role-repository diff and check the parent repository's submodule
status. Report existing lint violations separately from changes introduced by
the current edit.

## Known discrepancies and uncertainty

- `wit_devmode` and `dnstools_use_pip` are used with `default(false)` but are
  not declared in `defaults/main.yml`.
- The normal requirements path installs Debian packages only when
  `/etc/debian_version` exists; non-Debian support is not implemented by the
  role.
- `checkhostip.yml` uses an Ansible `dig` lookup, but the role does not contain
  a lookup plugin or metadata declaring where that lookup is provided.
- The role's normal DNS path delegates most work to localhost, while the
  domain-export tasks are not explicitly delegated.
- The source has a TODO about checking behavior with a single `dns_domain`.
- No role-local automated test suite was found during inspection.

When any of these behaviors changes, update this file and the human README from
the implementation rather than relying on this document's previous wording.
