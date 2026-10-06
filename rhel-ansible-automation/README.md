# ansible-rhel-automation

Ansible automation for Red Hat Enterprise Linux servers: baseline configuration,
user and SSH hardening, service management, and guarded patching.

## Layout

```
ansible-rhel-automation/
├── ansible.cfg
├── inventory/
│   └── production.ini
├── group_vars/
│   └── rhel.yml
├── site.yml
├── patch.yml
└── roles/
    ├── common/     # baseline packages, timezone, permissions
    ├── users/      # admin group, users, authorized SSH keys
    ├── sshd/       # SSH hardening with config validation
    ├── services/   # systemd service management + validation
    ├── httpd/      # Apache web tier
    └── firewall/   # firewalld rules for web tier
```

## Getting started

1. Edit `inventory/production.ini` and replace the example hostnames/IPs with
   your real servers.
2. Edit `group_vars/rhel.yml` and replace the placeholder SSH public keys with
   real ones for your admin users.
3. Make sure the `ansible` user on each target host can be reached with your
   SSH key and has passwordless sudo (for `become: true`).

## Usage

```bash
# Syntax check and dry run
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff

# Run the full baseline configuration
ansible-playbook -i inventory/production.ini site.yml

# Patch only the web tier, one host at a time
ansible-playbook -i inventory/production.ini patch.yml \
  --limit webservers -e "patch_packages=[httpd,openssl]"

# Ad-hoc: verify all hosts are reachable
ansible rhel -i inventory/production.ini -m ping
```

## Requirements

- Ansible 2.12+ on the control node
- Target hosts: RHEL 8/9 with the `ansible` user provisioned
- Collections: `ansible.posix` (`ansible-galaxy collection install ansible.posix`)
