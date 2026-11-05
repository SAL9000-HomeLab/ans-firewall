# ans-firewall

Locks down inbound access on Windows Server and Rocky Linux hosts. On Windows, the
[`ans_firewall`](roles/ans_firewall/README.md) role replaces the built-in Windows Firewall rules with versions scoped
to a management-IP allowlist: RDP, general remote administration, SSH, and (opt-in) SQL Server. On Rocky, it does the
same for SSH with firewalld, opt-in per group. Run `site.yml` from AWX.

The inventory in [inventory/hosts.yml](inventory/hosts.yml) is a placeholder. Put your real hosts in the AWX
inventory, or in a gitignored `local/` copy for workstation runs.

## Requirements

| Requirement | Version |
| --- | --- |
| ansible-core | >= 2.16 |
| Python | >= 3.10 |

- Windows targets: Windows Server 2019, 2022 or 2025, reached over PSRP or WinRM. The controller needs the Python
  packages in [requirements.txt](requirements.txt).
- Linux targets: RHEL/Rocky 9 and 10, reached over SSH with `become`. Ubuntu is not supported yet.
- Collections: `ansible.posix`, `ansible.windows` and `community.windows`, from [requirements.yml](requirements.yml).

## Inventory

[site.yml](site.yml) runs on the `linux` group with `become` and on the `windows` group. Set
`ans_firewall_enable_mssql_rules: true` on hosts that run SQL Server, and `ans_firewall_firewalld_enabled: true` on the
Linux groups that should use firewalld.

**Lockout risk:** `ans_firewall_mgmt_ips` must include the address Ascender jobs connect from, or hosts stop answering
them. Try a change on one host with a job limit first.

## Usage

```yaml
- name: Configure Windows Firewall
  hosts: windows
  roles:
    - role: ans_firewall
      vars:
        ans_firewall_enable_mssql_rules: true   # only for hosts running SQL Server
```

Each rule set can be run on its own with tags: `builtin`, `rdp`, `remoteadmin`, `ssh`, `mssql`, `firewalld`.

See [roles/ans_firewall/README.md](roles/ans_firewall/README.md) for the variable reference and the order rules
are applied in.

## AWX setup

1. **Project**: this repository.
2. **Inventory**: a `windows` group with the connection variables (`ansible_connection: psrp`, etc.) and any
   overrides of the role's defaults, such as `ans_firewall_mgmt_ips`.
3. **Credentials**: a Machine credential for an account with local admin rights on the hosts.
4. **Job template**: playbook `site.yml`, with **Prompt on launch** for the limit and tags.

## Development and CI

The workflows and lint configs are the shared ones from `ans-template`:

- **Ansible CI** (`.github/workflows/ansible-ci.yml`), on every pull request: `yamllint`,
  `ansible-playbook --syntax-check site.yml`, `ansible-lint`.
- **Linting Validation** (`.github/workflows/ci.yml`), on pull requests: markdownlint, linkspector, yamllint.

There is no Molecule scenario yet; `ans-template` has one for Windows roles on Azure VMs.

Run the same checks locally:

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install "ansible-core>=2.16" "ansible-lint>=6" "yamllint>=1.30" -r requirements.txt
ansible-galaxy collection install -r requirements.yml
yamllint -f parsable .
ansible-playbook -i inventory/hosts.yml --syntax-check site.yml
ansible-lint .
npx markdownlint-cli2 "**/*.md" "#.venv"
```

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change. Pushing a `vX.Y.Z` tag
publishes that version's section as a GitHub release.

## License

MIT, see [LICENSE](LICENSE).
