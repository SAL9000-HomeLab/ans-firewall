# ans_firewall

Manages Windows Firewall rules for SAL9000 homelab hosts: disables the noisy built-in inbound
rules and replaces them with versions scoped to a management IP allowlist, covering RDP, general
remote administration (WinRM, Task Scheduler, Remote Event Log, Remote Service/Volume/Firewall
management, File & Printer Sharing), SSH, and (opt-in) SQL Server.

## Requirements

- Windows Server target hosts, reachable via WinRM.
- Collections: `community.windows` (rule management), `ansible.windows` (built-in rule disabling
  via `win_powershell`). See the repository root `requirements.yml`.

## Role variables

All variables live in `defaults/main.yml` and are prefixed `ans_firewall_`.

### Global toggles

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_enable_firewall_rules` | `true` | Master switch; gates the whole role. |
| `ans_firewall_enable_rdp_rules` | `true` | Enables `rdp-rules.yml`. |
| `ans_firewall_enable_remoteadmin_rules` | `true` | Enables the rule set in `remoteadmin-rules.yml`. |
| `ans_firewall_enable_ssh_rules` | `true` | Enables `ssh-rules.yml`. |
| `ans_firewall_enable_mssql_rules` | `false` | Enables `mssql-rules.yml`. Opt-in — only set `true` on hosts actually running SQL Server. |
| `ans_firewall_disable_builtin_rules` | `true` | Enables `disable-builtins.yml`. |

### Shared IP allowlists

| Variable | Default | Purpose |
| --- | --- | --- |
| `ans_firewall_mgmt_ips` | `[]` (required) | Remote-address scope for RDP and RemoteAdmin rules. Set it on the AWX inventory or group; the role fails when it is empty and those rules are enabled. |
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
| `ans_firewall_mssql_display_prefix` | `ANS - Microsoft SQL Server` | Display-name prefix. |
| `ans_firewall_mssql_tcp_ports` | `[1433, 1434, 1435, 5022]` | TCP-In rule ports. |
| `ans_firewall_mssql_udp_ports` | `[1434, 3343]` | UDP-In rule ports. |

## Tasks

`tasks/main.yml` runs, in order:

1. `disable-builtins.yml` (when `ans_firewall_disable_builtin_rules`) — disables the stock Windows Firewall rules
   that the rules below replace (RDP, RemoteAdmin, SSH, File & Printer Sharing built-ins).
2. `rdp-rules.yml` — RDP (TCP/UDP 3389 + Shadow), scoped to `ans_firewall_mgmt_ips`.
3. `remoteadmin-rules.yml` — Task Scheduler, WinRM, Remote Volume Management, Remote Event Log,
   Remote Service Management, Windows Firewall remote management, File & Printer Sharing, ICMP.
   Scoped to `ans_firewall_mgmt_ips`.
4. `ssh-rules.yml` — SSH (TCP 22), scoped to `ans_firewall_ssh_ips`.
5. `mssql-rules.yml` — SQL Server (TCP/UDP), not scoped to an IP allowlist (opt-in via
   `ans_firewall_enable_mssql_rules`).

Each rule set can be run independently with tags: `builtin`, `rdp`, `remoteadmin`, `ssh`, `mssql`/`sql`.

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
