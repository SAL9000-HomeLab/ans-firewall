# ans_firewall

Manages host firewalls for SAL9000 homelab hosts.

- **Windows:** replaces the built-in inbound rules with versions scoped to a management IP allowlist,
  covering RDP, general remote administration (WinRM, Task Scheduler, Remote Event Log, Remote
  Service/Volume/Firewall management, File & Printer Sharing), SSH, and (opt-in) SQL Server.
- **Rocky/RHEL:** limits SSH to the same allowlist with firewalld. Opt-in, per group.

## Requirements

- Windows Server target hosts, reachable via PSRP or WinRM.
- RHEL/Rocky 9 or 10 target hosts for firewalld, reached with `become`.
- Collections: `community.windows` (rule management), `ansible.windows` (built-in rule disabling
  via `win_powershell`), `ansible.posix` (firewalld). See the repository root `requirements.yml`.

## Role variables

All variables live in `defaults/main.yml` and are prefixed `ans_firewall_`.

### Global toggles

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_enable_firewall_rules` | `true` | Master switch; gates the whole role. |
| `ans_firewall_enable_rdp_rules` | `true` | Windows: creates the rules in `rdp-rules.yml`, or removes them when `false`. |
| `ans_firewall_enable_remoteadmin_rules` | `true` | Windows: creates the rules in `remoteadmin-rules.yml`, or removes them when `false`. |
| `ans_firewall_enable_ssh_rules` | `true` | Windows: creates the rule in `ssh-rules.yml`, or removes it when `false`. |
| `ans_firewall_enable_mssql_rules` | `false` | Windows: creates the rules in `mssql-rules.yml`, or removes them when `false`. Opt-in — only set `true` on hosts actually running SQL Server. |
| `ans_firewall_disable_builtin_rules` | `true` | Windows: enables `disable-builtins.yml`. |
| `ans_firewall_firewalld_enabled` | `false` | Linux: enables `firewalld.yml`. Opt-in per group. |

### Shared IP allowlists

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_mgmt_ips` | `[]` (required) | IPv4 addresses or CIDR ranges allowed in. Set it on the AWX inventory or group; the role fails when it is empty and a rule set using it is enabled. |
| `ans_firewall_rdp_extra_ips` | `[]` | Added on top of `ans_firewall_mgmt_ips` for RDP on port 3389 only, e.g. the Kubernetes nodes a remote desktop gateway connects from. Not used for the Shadow rule, which allows any port on `svchost.exe`. |
| `ans_firewall_rdp_ips` | `(ans_firewall_mgmt_ips + ans_firewall_rdp_extra_ips) \| unique` | Remote-address scope for the RDP rules on port 3389. |
| `ans_firewall_ssh_extra_ips` | `[]` | Added on top of `ans_firewall_mgmt_ips` for SSH, e.g. a monitoring server. |
| `ans_firewall_ssh_ips` | `(ans_firewall_mgmt_ips + ans_firewall_ssh_extra_ips) \| unique` | Remote-address scope for the SSH rule. |
| `ans_firewall_profile` | `[Domain, Private, Public]` | Firewall profiles every managed rule applies to. |

### RemoteAdmin (`remoteadmin-rules.yml`)

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_group` | `ANS_remoteadmin_base` | Firewall rule group. |
| `ans_firewall_rule_prefix_rem` | `ANS-RemAdm` | Rule name prefix, e.g. `ANS-RemAdm-WINRM-HTTPS-In-TCP`. |

### RDP management (`rdp-rules.yml`)

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_rdp_mgmt_group` | `ANS_rdp_mgmt` | Firewall rule group. |

### SSH (`ssh-rules.yml`)

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_ssh_group` | `ANS_ssh` | Firewall rule group. |

### SQL Server (`mssql-rules.yml`)

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_mssql_group` | `ANS_mssql` | Firewall rule group. |
| `ans_firewall_rule_prefix_mssql` | `ANS-SQL` | Rule name prefix. |
| `ans_firewall_mssql_ips` | `[]` | IPv4 addresses or CIDR ranges allowed in. Empty means any address. |
| `ans_firewall_mssql_tcp_ports` | `[1433, 1434, 1435, 5022]` | TCP-In rule ports. |
| `ans_firewall_mssql_udp_ports` | `[1434, 3343]` | UDP-In rule ports. |

### WinRM rules from other tools (`disable-builtins.yml`)

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_winrm_extra_rules` | `[WinRM-HTTP, WinRM-HTTPS]` | Display names of WinRM rules created outside this role, open to any address. Disabled when RemoteAdmin rules are enabled. These are the ones `vm-templates` and `proxmox-deploy` create; running `proxmox-deploy` again re-enables its rule until this role next runs. |

### firewalld (`firewalld.yml`)

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_firewalld_zone` | `public` | Zone the rules go in: the zone of the host's interface. |
| `ans_firewall_firewalld_mgmt_services` | `[ssh]` | Services allowed only from `ans_firewall_ssh_ips`. |
| `ans_firewall_firewalld_remove_services` | `[ssh, cockpit]` | Services taken out of the zone, so they are no longer open to any address. |

## Tasks

On Windows, `tasks/windows.yml` runs, in order:

1. `validate.yml` — checks the allowlists are set and each entry is an IPv4 address or CIDR range.
2. `rdp-rules.yml` — RDP on TCP/UDP 3389, scoped to `ans_firewall_rdp_ips`, and Shadow, scoped to
   `ans_firewall_mgmt_ips`.
3. `remoteadmin-rules.yml` — Task Scheduler, WinRM, Remote Volume Management, Remote Event Log,
   Remote Service Management, Windows Firewall remote management, File & Printer Sharing, ICMP echo.
   Scoped to `ans_firewall_mgmt_ips`.
4. `ssh-rules.yml` — SSH (TCP 22), scoped to `ans_firewall_ssh_ips`.
5. `mssql-rules.yml` — SQL Server (TCP/UDP), opt-in via `ans_firewall_enable_mssql_rules`. Scoped to
   `ans_firewall_mssql_ips` when that is set, otherwise open to any address.
6. `disable-builtins.yml` (when `ans_firewall_disable_builtin_rules`) — disables the rules the sets above replace.

The scoped rules go first so a host always has a rule for the next management connection. A set that is turned off
has its managed rules removed and its built-in rules left as they are. `--tags builtin` on its own skips the creation
steps, so use it only on hosts that already have the scoped rules.

On Linux, when `ans_firewall_firewalld_enabled` is set, `tasks/firewalld.yml` installs and starts firewalld, adds rich
rules allowing `ans_firewall_firewalld_mgmt_services` from `ans_firewall_ssh_ips`, removes rich rules for those
services from addresses no longer listed, and takes `ans_firewall_firewalld_remove_services` out of the zone. Other
services, ports and rich rules in the zone are left alone. It uses rich rules rather than a zone bound to the source
addresses, so listed hosts can still reach ports other roles open in the zone. Turning the setting off later leaves
firewalld and its rules as they are. Leave it off on hosts that run with firewalld off, such as RKE2 nodes.

Each rule set can be run independently with tags: `builtin`, `rdp`, `remoteadmin`, `ssh`, `mssql`/`sql`, `firewalld`.

## Example playbook

```yaml
- name: Configure Windows Firewall
  hosts: windows
  roles:
    - role: ans_firewall
      vars:
        ans_firewall_enable_mssql_rules: true   # only for hosts running SQL Server
```

## License

MIT
