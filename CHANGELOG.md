# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Changed: `ans_firewall_mgmt_ips` and `ans_firewall_ssh_extra_ips` default to `[]`; the lab addresses are removed
  from the repository and belong on the AWX inventory or group. The role fails when an enabled rule set would have
  an empty allowlist, instead of creating rules open to any address.
- Changed: Renamed the role from `firewall` to `ans_firewall`, so its name matches its `ans_firewall_` variables.
  Variable names are unchanged; playbooks must use `role: ans_firewall`.
- Changed: Brought the repository in line with the shared `ans-template` standard: the `SAL9000-HomeLab/shared-actions`
  CI workflows (fixing CI, which pointed at the old `Bocephus/shared-github-actions`), shared lint configs, PR
  template, MIT license, and this changelog at the repository root.
- Added: `site.yml` and a placeholder `inventory/hosts.yml`.
- Changed: Collections are listed once, in `requirements.yml`, as `ansible.windows` and `community.windows` 3.x
  (the root file said `<3.0.0` while `collections/requirements.yml` pinned 3.3.0 and 3.1.0).
- Fixed: `ans_firewall_disable_builtin_rules` now controls whether the built-in rules are disabled (default `true`,
  as before); it was previously ignored.
- Removed: `ansible.cfg`, `collections/requirements.yml`, the galaxy `tests/` directory, empty `vars/` and
  `handlers/`, and the placeholder Molecule docs (they live in `ans-template`).
- Added: SQL Server Windows Firewall rules (`mssql-rules.yml`), opt-in via `ans_firewall_enable_mssql_rules`.
- Changed: Split RDP management rules out of `create-rules.yml` into their own `rdp-rules.yml`, scoped to
  `ans_firewall_mgmt_ips` under the `ANS_rdp_mgmt` group (still gated by `ans_firewall_enable_rdp_rules`).
- Changed: Split SSH rules out of `create-rules.yml` into their own `ssh-rules.yml`, scoped to
  `ans_firewall_ssh_ips` under the `ANS_ssh` group (still gated by `ans_firewall_enable_ssh_rules`).
- Changed: Replaced `create-rules.yml` with `remoteadmin-rules.yml`: real per-rule descriptions/display names,
  `RPC`/`RPCEPMap` local-port keywords instead of hardcoded `135`, fixed the LLMNR port typo (`5535` -> `5355`)
  and a swapped description/display name on the FPS ICMPv4 rule.
- Fixed: `create-rules.yml`'s rule-set merge referenced `ans_firewall_rules_remoteadmin`/`ans_firewall_rules_ssh`,
  which never matched the facts actually set (`ans_firewall_firewall_rules_remoteadmin`/`_ssh`), so the
  RemoteAdmin/SSH rules were silently never applied.
- Added: Initial release of the `firewall` role.
- Changed: Ansible CI runs on pull requests only, no longer on pushes to `main` (synced from ans-template).
