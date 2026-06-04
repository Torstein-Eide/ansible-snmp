# Ansible Role: SNMP

Installs and configures SNMPv3 on Debian/Ubuntu and RHEL/CentOS, and SNMPv2 on Windows. Automatically detects installed services and wires up the corresponding [LibreNMS agent](https://github.com/librenms/librenms-agent) SNMP application extension scripts.

Forked from [sbaerlocher/ansible.snmp](https://galaxy.ansible.com/sbaerlocher/snmp).

## Requirements

- Ansible 2.7+
- Target hosts: Debian, Ubuntu, RHEL/CentOS 7, or Windows

## Role variables

### SNMPv3 credentials

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_user` | `snmp` | SNMPv3 username |
| `snmp_password` | `snmp_password` | SNMPv3 auth password (SHA) |
| `snmp_encryption` | `snmp_encryption` | SNMPv3 privacy password (AES) |

### Agent address

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_agentadress_protocol.ipv4` | `udp` | Protocol for IPv4 |
| `snmp_agentadress_protocol.ipv6` | `udp6` | Protocol for IPv6 |
| `snmp_agentadress_adress.ipv4` | `ansible_default_ipv4.address` | Bind address for IPv4 |
| `snmp_agentadress_adress.ipv6` | `ansible_default_ipv6.address` | Bind address for IPv6 |
| `snmp_agentadress_port.ipv4` | `161` | Port for IPv4 |
| `snmp_agentadress_port.ipv6` | `161` | Port for IPv6 |

### Community strings (SNMPv1/v2c)

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_source` | `[127.0.0.1, 10.0.0.0/8, 192.168.0.0/16]` | Allowed IPv4 sources |
| `snmp_source6` | `['::1']` | Allowed IPv6 sources |

### System

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_contact` | — | `sysContact` value |
| `snmp_location` | — | `sysLocation` value |
| `snmp_additional_packages` | `[]` | Extra packages to install alongside snmpd |
| `snmp_custom_line` | — | List of raw lines appended to `snmpd.conf` |

### lldpd

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_lldpd_enabled` | `true` | Install lldpd and register it as an AgentX subagent |

When enabled, the role installs lldpd, adds its daemon user to the snmpd group so it can connect to the AgentX socket, and deploys `/etc/lldpd.conf`. lldpd then exposes LLDP-MIB data through snmpd automatically. Requires `snmp_agentx_enabled: true` (the default).

### Application extensions

All application extensions are auto-detected by probing for the relevant binary (or process). Each can be individually disabled.

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_application_extensions_enabled` | `true` | Master switch for all extensions |
| `snmp_application_validate_snmp_output` | `true` | Query each extension via SNMPv3 after deploy to verify it responds |
| `snmp_application_apache_enabled` | `true` | Apache |
| `snmp_application_nginx_enabled` | `true` | Nginx |
| `snmp_application_mysql_enabled` | `true` | MySQL / MariaDB |
| `snmp_application_redis_enabled` | `true` | Redis |
| `snmp_application_memcached_enabled` | `true` | Memcached |
| `snmp_application_fail2ban_enabled` | `true` | Fail2ban (cached via systemd timer) |
| `snmp_application_chronyd_enabled` | `true` | Chrony |
| `snmp_application_docker_enabled` | `true` | Docker |
| `snmp_application_systemd_enabled` | `true` | systemd |
| `snmp_application_borgbackup_enabled` | `true` | Borg Backup |
| `snmp_application_mdadm_enabled` | `true` | mdadm software RAID |
| `snmp_application_nfs_enabled` | `true` | NFS |
| `snmp_application_osupdate_enabled` | `true` | OS package updates |
| `snmp_application_pihole_enabled` | `true` | Pi-hole |
| `snmp_application_proxmox_enabled` | `true` | Proxmox VE (cached via systemd timer) |
| `snmp_application_smart_enabled` | `true` | S.M.A.R.T. / smartmontools (cached via systemd timer) |
| `snmp_application_nvidia_gpu_enabled` | `true` | NVIDIA GPU (nvidia-smi) |
| `snmp_application_ups_nut_enabled` | `true` | NUT UPS |

Extensions marked *cached via systemd timer* run on a 5-minute systemd timer and write their output to `/run/snmp/extension/<app>`. The snmpd `extend` directive reads from the cache file, avoiding blocking SNMP polls on slow scripts.

#### S.M.A.R.T. specific variables

| Variable | Default | Description |
| :--- | :--- | :--- |
| `snmp_application_smart_devices` | `[]` | Explicit device list; auto-scanned with `smartctl --scan-open` if empty |
| `snmp_application_smart_use_sn` | `1` | Use serial number as device identifier |

## Example playbook

```yaml
- hosts: all
  roles:
    - role: ansible-snmp
      snmp_user: monitoring
      snmp_password: "{{ vault_snmp_password }}"
      snmp_encryption: "{{ vault_snmp_encryption }}"
      snmp_location: "Server Room A"
      snmp_contact: "ops@example.com"
```

## Dependencies

None.

## Author

Originally by [Simon Bärlocher](https://sbaerlocher.ch).

## License

MIT
