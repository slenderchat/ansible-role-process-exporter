# process_exporter

Ansible role that installs [Process Exporter](https://github.com/ncabatoff/process-exporter) from the operating system package and runs it as a hardened systemd service, under a dedicated system user with exactly two Linux capabilities: `CAP_DAC_READ_SEARCH` and `CAP_SYS_PTRACE`.

Implemented by Nikita Grabar.

> **Copyright.** Copyright (c) 2026 Nikita Grabar. All rights reserved. This repository is not open-source software and is not distributed under an open-source license. No license is granted by this distribution: using, copying, modifying or redistributing it requires the prior permission of the copyright holder. See [COPYRIGHT](COPYRIGHT).

## Supported platforms

- RED OS 7.3 (`golang-github-ncabatoff-process-exporter-0.8.7-1.el7`);
- RED OS 8 (`golang-github-ncabatoff-process-exporter-0.8.7-1.red80`).

Both platforms use the same implementation. The separate `process-exporter` package available on RED OS 8 is not used.

The role has been deployed on RED OS 7.3 and RED OS 8 hosts, both with the package preinstalled and with a fresh package installation. Each deployment passed the initial run, a repeated run (`changed=0`, `failed=0`) and a final check mode run. Runtime checks confirmed an enabled and active service, the dedicated user, the expected capability masks, `NoNewPrivs=1`, a listener only on `127.0.0.1:9256`, HTTP 200 from `/metrics` and no warnings in the service journal.

## Requirements

- The package `golang-github-ncabatoff-process-exporter` is available from the host's configured repositories. The role installs it with `state: present`: the version is not pinned and the role never upgrades it.
- systemd with `AmbientCapabilities=` support (systemd 229 or later).
- `ss` (package `iproute`) for the post-deployment checks.
- Privilege escalation (`become: true`): the role installs a package, creates a system user and manages a systemd unit.
- Only `ansible.builtin` modules are used; no collections are required.
- `127.0.0.1:9256` must be free when the role runs. If another exporter, such as a Docker container, listens there, see [Migrating from a Docker-based exporter](#migrating-from-a-docker-based-exporter).

## Installation

Place the repository contents in a directory named `process_exporter` inside one of your Ansible roles paths, for example `roles/process_exporter/` next to your playbooks, or point `roles_path` (or `ANSIBLE_ROLES_PATH`) at its parent directory. The role has no dependencies.

## Role variables

None. The listen address `127.0.0.1:9256`, the file paths and the privileges are fixed on purpose.

## Example playbook

```yaml
- name: Deploy Process Exporter
  hosts: all
  become: true
  gather_facts: false
  roles:
    - process_exporter
```

Run check mode first, then the deployment, limited to the intended hosts:

```bash
ansible-playbook -i INVENTORY site.yml -l HOST --check --diff
ansible-playbook -i INVENTORY site.yml -l HOST --diff
```

## What the role does

1. Installs the package. It contains only `/usr/bin/process-exporter`, without a unit file or configuration.
2. Creates the system group and user `process-exporter` (`/sbin/nologin`, home `/`, no home directory is created).
3. Deploys the files:

   | File | Source | Owner and mode |
   |---|---|---|
   | `/opt/process-exporter/process-exporter.yml` | `files/process-exporter.yml` | `root:root 0644` |
   | `/etc/systemd/system/process-exporter.service` | `files/process-exporter.service` | `root:root 0644` |

4. Removes `/etc/sysconfig/process-exporter`, which a legacy unit variant used to pass `OPTIONS` to the exporter; all options now live in the unit file. The file is removed only after systemd has loaded the new unit, because the legacy unit would start the exporter on all interfaces without it. If systemd has not loaded the new unit, the run fails and the file is kept.
5. Enables and starts the `process-exporter` service.
6. Validates the running exporter (see [Post-deployment validation](#post-deployment-validation)).

A configuration change restarts the service; a unit file change triggers `systemctl daemon-reload` and a restart. Without changes the service is not restarted. Whether a reload is pending is read from systemd (`NeedDaemonReload`), not only from file-change notifications, so a run that installed the unit file but failed before the reload is recovered on the next run.

## Process grouping

```yaml
process_names:
  - name: "{{.Comm}}"
    cmdline:
      - ".+"
```

Each group corresponds to a command name (`comm`, at most 15 characters); processes with an empty command line (kernel threads) are not counted. Exporter options other than the listen address and the configuration path keep their defaults (`-threads`, `-gather-smaps` and `-children` enabled, `-procfs /proc`).

## Privileges

The service runs as the system user `process-exporter` with exactly two capabilities (`files/process-exporter.service`):

```ini
CapabilityBoundingSet=CAP_DAC_READ_SEARCH CAP_SYS_PTRACE
AmbientCapabilities=CAP_DAC_READ_SEARCH CAP_SYS_PTRACE
NoNewPrivileges=yes
```

`AmbientCapabilities=` grants the capabilities to the process of the unprivileged user, `CapabilityBoundingSet=` prevents it from obtaining any others, and `NoNewPrivileges=yes` blocks privilege gain through setuid files and file capabilities. For the running process, the `CapInh`, `CapPrm`, `CapEff`, `CapBnd` and `CapAmb` masks in `/proc/<pid>/status` equal `0000000000080004` (`capsh --decode=0000000000080004` prints `cap_dac_read_search,cap_sys_ptrace`). Additional systemd hardening applies (`ProtectSystem=strict`, `ProtectHome`, `PrivateTmp`, `PrivateDevices`, `ProtectKernel*`, `RestrictAddressFamilies` and others). Options that restrict `/proc` visibility (`ProtectProc`, `ProcSubset`, `PrivateUsers`) are deliberately not used. The capability set is not configurable.

### Why both capabilities are required

Without capabilities, the `process-exporter` user can read `/proc/<pid>/stat`, `status`, `cmdline` and `limits` of any user's processes: process and thread counts, CPU, page faults, context switches, `resident`/`virtual`/`swapped` memory, states and start times. The kernel exposes the remaining data about other users' processes, including root's, only with a capability:

| Source | `namedprocess_namegroup_*` metrics | Required capability |
|---|---|---|
| `/proc/<pid>/io` | `read_bytes_total`, `write_bytes_total`, `thread_io_bytes_total` | `CAP_SYS_PTRACE` (ptrace access check) |
| `/proc/<pid>/smaps_rollup` | `memory_bytes{memtype="proportionalResident\|proportionalSwapped"}` | `CAP_SYS_PTRACE` |
| `/proc/<pid>/fd` | `open_filedesc`, `worst_fd_ratio` | `CAP_DAC_READ_SEARCH` (the directory is accessible only to the process owner) |

The set was chosen by testing on RED OS 7.3 and RED OS 8: a temporary instance running as `process-exporter` was tested without capabilities, with `CAP_DAC_READ_SEARCH` only, with `CAP_SYS_PTRACE` only, and with both. Only both together restored the tested metrics of other users' processes and eliminated persistent partial scrape errors (`namedprocess_scrape_partial_errors`); the actual capability masks were verified on both platforms. With either capability alone, part of the metrics in the table stays unavailable.

`smaps_rollup` exists only in Linux 4.14 and later; on older kernels proportional memory is not collected regardless of privileges.

### Security implications

This is not an unprivileged mode: both capabilities apply to all processes and files on the host, so a compromise of the exporter must be treated as a compromise of the host.

- `CAP_SYS_PTRACE` passes ptrace access checks for any process, including root's. Besides reading `/proc/<pid>/io` and `smaps_rollup`, it allows reading `/proc/<pid>/environ` and process memory, attaching with `ptrace(2)` and modifying process memory (`/proc/<pid>/mem`, `process_vm_writev(2)`). `kernel.yama.ptrace_scope` values 1 and 2 do not restrict a holder of this capability. Code execution in the exporter is a practical path to code execution as root.
- `CAP_DAC_READ_SEARCH` bypasses read permission checks on files and search/read checks on directories: the exporter can read any file it can see (`/etc/shadow`, private keys, configuration files with passwords). `ProtectHome` and `PrivateTmp` hide paths but are not a reliable boundary: the same capability allows `open_by_handle_at(2)`, which opens files by file handle without path lookup.
- What limits the risk: the process cannot obtain any other capability (`CapabilityBoundingSet=`, `NoNewPrivileges=yes`); it has no permission to write files owned by others (`CAP_DAC_OVERRIDE` is not granted), and `ProtectSystem=strict` makes its file system read-only; the exporter listens only on `127.0.0.1`, and the user has no login shell.
- Compared with a root container in privileged mode (all capabilities), the privileges are narrower but remain sensitive. Package updates, including security fixes, arrive through regular OS updates: the role installs the package with `state: present` and never upgrades it.

Do not add other capabilities or run the service as root. Without either of the two capabilities, the metrics in the table above become unavailable for other users' processes.

## Post-deployment validation

Outside check mode, the role:

1. waits for `http://127.0.0.1:9256/metrics` to respond;
2. checks that the service is active and that the only listener on port 9256 is `127.0.0.1:9256` owned by the service's main PID (an HTTP 200 alone could come from a leftover container);
3. checks that the process groups `systemd` (PID 1, root) and `process-exporte` (the exporter itself) are exported, which confirms the `{{.Comm}}` grouping and visibility of another user's processes.

The role does not evaluate the `namedprocess_scrape_*` counters: they are cumulative (a non-zero value remains after the cause is gone), and occasional partial scrape errors were observed equally with a Docker-based and the native exporter. The role does not check the capability masks of the process either; check them manually after deployment (step 4 below).

A successful validation does not prove that metrics are complete for every process.

## Migrating from a Docker-based exporter

If a Docker container (here named `process-exporter`) already serves `127.0.0.1:9256`, the native service cannot start while the port is taken: the process exits with `address already in use`, systemd restarts it every 5 seconds (`Restart=on-failure`), and the role fails at the listener check. The role does not manage Docker and does not stop the container, so the operator performs the cutover manually, one host at a time, in an agreed maintenance window. Process metrics of the host are not collected between stopping the container and starting the service; after the cutover the counters start from zero (a normal counter reset for `rate()`/`increase()`).

1. **Preparation** (no changes on the host; the container keeps running):
   - save the container definition for rollback: `docker inspect process-exporter > /root/process-exporter.inspect.json`; note the restart policy: `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' process-exporter`;
   - make sure no other automation (an older deployment playbook, a compose file, cron) recreates the container;
   - run the role in check mode: `ansible-playbook -i INVENTORY site.yml -l HOST --check --diff` must finish without errors.
2. **Stop the container** without removing it, so that it can be restored quickly and does not start again when Docker or the host restarts:

   ```bash
   docker update --restart=no process-exporter
   docker stop process-exporter
   ss -ltnp 'sport = :9256'      # no LISTEN lines expected
   ```

   Do not continue while the port is still in use.
3. **Run the role** without `--check`, for this host only: `ansible-playbook -i INVENTORY site.yml -l HOST --diff`. It must finish without errors; its checks confirm that the only listener on `127.0.0.1:9256` is the `process-exporter` service.
4. **Check the host:**

   ```bash
   systemctl is-enabled process-exporter; systemctl is-active process-exporter    # enabled, active
   ss -ltnp 'sport = :9256'                                    # one LISTEN line: 127.0.0.1:9256, "process-exporte"
   PID=$(systemctl show -p MainPID process-exporter | cut -d= -f2)
   ps -o user=,comm= -p "$PID"                                 # process-exporter process-exporte
   grep -E '^(CapInh|CapPrm|CapEff|CapBnd|CapAmb|NoNewPrivs)' /proc/"$PID"/status
   curl -s --noproxy '*' http://127.0.0.1:9256/metrics \
       | grep -E '^namedprocess_namegroup_(open_filedesc|memory_bytes)[{]groupname="systemd"'
   ```

   Expected: all `Cap*` masks equal `0000000000080004`, `NoNewPrivs: 1`; for the `systemd` group (PID 1, root), `open_filedesc` and `memory_bytes{memtype="proportionalResident"}` are greater than zero, which they would not be without the capabilities. Confirm that your monitoring system receives the host's metrics again.
5. **Finish.** After acceptance, remove the container: `docker rm process-exporter`. Removing the image and any other Docker cleanup is a separate decision.

If the role is run by mistake while the container still holds the port, it fails at the task `Verify that the service owns the only listener on 127.0.0.1:9256`. The container keeps serving metrics, while the native service stays enabled and keeps restarting. Either roll back (below) or continue with step 2 and run the role again.

### Rollback

While the container still exists:

```bash
systemctl disable --now process-exporter      # free the port first
docker update --restart=POLICY process-exporter   # the policy noted in step 1
docker start process-exporter
ss -ltnp 'sport = :9256'                       # the container is the listener again
```

After a rollback, do not run the role against that host until the next cutover attempt: it would enable and start the native service again. The package, the user and the role's files do not interfere with the container. If the container has already been removed, it has to be recreated from the saved `docker inspect` output, so remove it only after acceptance.

## Check mode

The role supports `--check` (and `--diff`) on hosts with and without the package installed. When the package or unit file installation is only simulated in check mode, service management (enable, start, restart) is skipped; the `/metrics` validation never runs in check mode.

## Monitoring

The exporter listens only on `127.0.0.1:9256`, so metrics can be scraped only from the host itself, for example by a local agent. The role does not configure Prometheus, VictoriaMetrics or any other scraper.

## Tags

None.

## Copyright

Copyright (c) 2026 Nikita Grabar. All rights reserved. No license is granted; see [COPYRIGHT](COPYRIGHT).
