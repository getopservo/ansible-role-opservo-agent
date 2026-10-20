# Ansible role: opservo.opservo_agent

Installs the [Opservo](https://getopservo.com) monitoring agent on Linux hosts
with the **official installer** (`<API>/install.sh`), so the agent binary is
verified against Opservo's Ed25519-signed manifest exactly as in a manual
install. Hosts appear in your Opservo workspace within a minute.

- **Idempotent** — skipped on hosts that already have `/etc/infrapilot/config`;
  the service is only ensured enabled + running.
- **Key never logged** — the installer task is `no_log: true`.
- **Read-only agent** — remote command execution stays off. Turning it on is a
  deliberate per-server step (`OPSERVO_ENABLE_ACTIONS=true` in
  `/etc/infrapilot/config`), not something this role toggles.

## Requirements

- Ansible 2.14+, systemd hosts (Ubuntu, Debian, RHEL/Rocky/Alma, Amazon Linux).
- Outbound HTTPS from each host to the Opservo API (no inbound ports).
- A **fleet enrollment key** (`fleet_…`) from the dashboard: *Add server →
  Auto Scaling / golden AMI → Create fleet key*. Fleet keys are multi-use and
  revocable; a single-use `ipk_…` token only enrolls one host.

## Install

```bash
ansible-galaxy role install opservo.opservo_agent
```

## Role variables

| Variable | Default | Meaning |
|---|---|---|
| `opservo_agent_enroll_key` | `""` | Fleet key (`fleet_…`). Required for new installs. Store it in Ansible Vault. |
| `opservo_agent_api_url` | `https://api.getopservo.com` | API the agent reports to. Must be `https://`. |
| `opservo_agent_installer_url` | `{{ opservo_agent_api_url }}/install.sh` | Installer location. |
| `opservo_agent_force_reinstall` | `false` | Re-run the installer on installed hosts (upgrades the binary, keeps identity). |
| `opservo_agent_force_reregister` | `false` | Re-register as a NEW server (`OPSERVO_FORCE=1`). |
| `opservo_agent_install_env` | `{}` | Extra installer env, e.g. `{OPSERVO_VERSION: latest}`. |

## Example playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: opservo.opservo_agent
      vars:
        opservo_agent_enroll_key: "{{ vault_opservo_enroll_key }}"
```

```bash
ansible-vault encrypt_string 'fleet_…' --name vault_opservo_enroll_key >> group_vars/all/vault.yml
ansible-playbook -i inventory.ini site.yml --ask-vault-pass
```

The agent updates itself (signed manifests), so you do not need to re-run the
role to upgrade it.

## Uninstall

```bash
ansible all -b -m shell -a 'curl -sSL https://api.getopservo.com/install.sh | OPSERVO_UNINSTALL=1 bash'
```

## License

MIT
