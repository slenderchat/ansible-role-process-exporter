# Project Context

Durable handoff state for agents. See `AGENTS.md` for what belongs here.

## Current state

- The role is complete: it installs Process Exporter 0.8.7 from the RED OS package and runs it as a hardened systemd service on `127.0.0.1:9256`.
- Validated on RED OS 7.3 and RED OS 8 hosts, with the package preinstalled and with a fresh installation: initial deployment, repeated run (`changed=0`, `failed=0`) and final check mode. Runtime checks confirmed an enabled and active service, the dedicated user, capability masks `0x80004`, `NoNewPrivs=1`, a listener only on `127.0.0.1:9256`, HTTP 200 and no service journal warnings. Metric ingestion by a scraper is outside the role and was not part of this validation.

## Decisions and constraints

- Package `golang-github-ncabatoff-process-exporter` 0.8.7 ships only `/usr/bin/process-exporter`; the role owns the systemd unit (`files/process-exporter.service`) and the grouping configuration (`files/process-exporter.yml`, grouping by `{{.Comm}}`). Both are static files: no templates, no variables. The alternative `process-exporter` RPM on RED OS 8 is not used.
- Fixed bind `127.0.0.1:9256`, no role variables. Scrape configuration is out of scope.
- Privileges: user `process-exporter` with exactly `CAP_DAC_READ_SEARCH` + `CAP_SYS_PTRACE` (both `CapabilityBoundingSet=` and `AmbientCapabilities=`), `NoNewPrivileges=yes`, systemd hardening without `/proc` restrictions. Chosen from tests on both platforms (none / each alone / both): only both restore cross-user metrics (`/proc/<pid>/io` and `smaps_rollup` need `CAP_SYS_PTRACE`, `/proc/<pid>/fd` needs `CAP_DAC_READ_SEARCH`). No other capabilities, no root, not configurable; the security implications are documented in `README.md`.
- Validation in the role does not trust HTTP 200 alone: the only listener on `127.0.0.1:9256` must belong to the service's `MainPID` (a leftover container could answer otherwise). The role deliberately does not evaluate `namedprocess_scrape_*` counters (cumulative; occasional partial errors also occur with a Docker-based exporter) and does not assert capability masks.
- Docker is never managed by the role: stopping or removing an old container and rolling back are manual operator steps (README, "Migrating from a Docker-based exporter").
- Reload need is taken from systemd (`NeedDaemonReload`), not only from unit-file change notifications, so a run that wrote the unit but failed before daemon-reload is recovered on the next run. `/etc/sysconfig/process-exporter` (legacy unit options) is deleted only after asserting that systemd has loaded the new unit; otherwise the run fails and keeps the file.
- Check mode: when the package or unit installation is only simulated, service operations are skipped (`process_exporter_service_manageable` in `vars/main.yml`); the `/metrics` validation never runs in check mode.
- All rights reserved (`COPYRIGHT`): do not add an open-source license file or license metadata (e.g. Galaxy `meta/main.yml` with a `license` field) without the copyright holder's decision.

## Validation commands

Run from the repository root. The syntax check needs a playbook and the role under the name `process_exporter`:

```bash
tmp=$(mktemp -d); mkdir "$tmp/roles"; ln -s "$PWD" "$tmp/roles/process_exporter"
printf -- '- hosts: all\n  gather_facts: false\n  roles: [process_exporter]\n' > "$tmp/site.yml"
ANSIBLE_ROLES_PATH="$tmp/roles" ansible-playbook -i localhost, "$tmp/site.yml" --syntax-check
ansible-lint --profile production
yamllint -d '{extends: default, rules: {line-length: {max: 120}}}' .
```

`files/process-exporter.yml` intentionally has no `---`, so yamllint reports a `document-start` warning for it. Behaviour that depends on systemd, capabilities and `/proc` access can only be verified on a RED OS host (README, step 4 of the migration procedure).
