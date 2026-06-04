# AGENTS

This file provides guidance to AI agents when working with code in this repository.

## Role overview

This is an Ansible role that installs and configures SNMPv3 (Linux) / SNMPv2 (Windows) and wires up LibreNMS SNMP application extension scripts for monitored services. It is a fork of `sbaerlocher/ansible.snmp` with significant local extensions.

## Running the role

```bash
# Run against all hosts (from the playbook root, not this role directory)
ansible-playbook site.yml --tags snmp

# Limit to a single host
ansible-playbook site.yml -l hostname --tags snmp

# Configuration-only tasks (skip package install)
ansible-playbook site.yml -l hostname --tags configuration

# Syntax-check the role
ansible-playbook site.yml --syntax-check

# Dry-run
ansible-playbook site.yml -l hostname --check
```

## Architecture

### Task dispatch

`tasks/main.yml` runs `setup` (gather facts), loads the most-specific OS vars file from `vars/` (tries distro-release, distro, os_family, then `defaults.yml`), and then includes the most-specific distribution task file from `tasks/distribution/`.

For Linux hosts this is `tasks/distribution/Linux.yml`, which:
1. Installs packages and creates directories.
2. Renders `templates/default/snmpd.Debian.j2` or `snmpd.RedHat.j2` as the daemon sysconfig file.
3. Renders `templates/snmp/snmpd.conf.j2` as the main SNMP config — this is only written when the content actually changes (stop/start guard block).
4. Calls `Linux-applications.yml` to configure application extension scripts.

### Application extensions pipeline

`tasks/distribution/Linux-applications.yml` orchestrates LibreNMS per-application SNMP extends. The pipeline is:

1. **Discovery** (`applications-discovery.yml`): probes for each application binary using `command -v` (or equivalent). Records results in `snmp_application_discovery` dict and scans `snmp_config_include_dir` for already-deployed snippets.

2. **Dispatch**: each application's task file (`tasks/distribution/applications/<app>.yml`) is included only when the app is enabled **and** (the app was discovered **or** a stale snippet is present). The second condition allows cleanup when an app is removed.

3. **Extension patterns**: extensions that are slow to query (smart, fail2ban, proxmox) use a systemd caching pattern:
   - A generic template service `librenms-snmp-extension@.service` + timer `librenms-snmp-extension@.timer` run the script on a schedule and write output to `snmp_application_cache_dir` (default `/run/snmp/extension`).
   - A per-extension systemd override in `librenms-snmp-extension@<app>.service.d/override.conf` sets the correct script path and cache file.
   - The snmpd snippet (`snmp/proxmox-cached.conf.j2` etc.) reads from the cache file via `extend ... /bin/cat <cache_file>`.
   - On hosts without systemd the `-direct.conf.j2` snippet calls the script inline on every SNMP poll.

4. **Validation** (`applications-validate.yml`): after handlers flush, queries each active extension via `snmpget -v3` and fails if the OID is absent or empty. Controlled by `snmp_application_validate_snmp_output`.

### Key variables (defaults/main.yml)

- `snmp_user` / `snmp_password` / `snmp_encryption` — SNMPv3 credentials (SHA auth, AES priv).
- `snmp_source` / `snmp_source6` — allowed communities for SNMPv1/v2c.
- `snmp_config_include_dir` — drop-in directory included by snmpd; each app deploys one `<app>.conf` snippet here.
- `snmp_application_extensions_enabled` — master switch for all application extensions.
- `snmp_application_<app>_enabled` — per-app override; defaults all to `true`.
- `snmp_application_cache_dir` — `/run/snmp/extension` (tmpfs, recreated by tmpfiles.d on boot).
- `snmp_application_validate_snmp_output` — run post-deploy validation (default `true`).

### Adding a new application extension

Follow the pattern of `smart` or `fail2ban`:
1. Add detection shell in `applications-discovery.yml` → register `snmp_application_<app>_detect`.
2. Add the app key to the `snmp_application_discovery` fact at the bottom of that file.
3. Add a default `snmp_application_<app>_enabled: true` in `defaults/main.yml`.
4. Create `tasks/distribution/applications/<app>.yml` with install, cached+direct snippet tasks, and cleanup tasks.
5. Add templates under `templates/snmp/<app>-cached.conf.j2`, `<app>-direct.conf.j2`, and `templates/systemd/librenms-snmp-extension-<app>-override.conf.j2` if using the caching pattern.
6. Add a dispatch block in `Linux-applications.yml`.
7. Add the validation OID entry in `applications-validate.yml`.
8. Add handlers `refresh <app> cache` in `handlers/main.yml` if using the caching pattern.
