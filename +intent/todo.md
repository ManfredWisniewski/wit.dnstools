## Discrepancies recorded

wit_devmode and dnstools_use_pip are used but not defined in role defaults.
Non-Debian dependency installation is not implemented.
checkhostip.yml relies on the Ansible dig lookup without a role-local lookup plugin or metadata declaration.
DNS tasks are delegated to localhost, while domain-export tasks are not explicitly delegated.
Domain export runs immediately after deployment, so placeholder credentials cause the initial export to fail.
Existing no_log: true protections are commented out.
The role contains a TODO concerning single-domain behavior.
No role-local automated tests were found.

## Validation

Fenced YAML examples parsed successfully.
Markdown code fences are balanced.
git diff --check passed.
No Ansible playbook was run.
ansible-lint reported 82 pre-existing violations across the role.
yamllint reported pre-existing formatting violations.
No Markdown linter is installed.
Documentation examples contain placeholders only; no real secrets were added.