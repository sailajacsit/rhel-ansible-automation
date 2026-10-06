Red Hat Enterprise Linux Server Configuration & Automation using Ansible

1. Project Overview
This project automates the configuration and administration of Red Hat Enterprise Linux (RHEL) servers using Ansible. The main goal is to reduce manual configuration work and make sure that all servers are configured in a consistent, repeatable way.

Instead of logging into each server and running commands by hand, the desired state of every server is described in code — inventories, playbooks and reusable roles — and Ansible brings each host to that state, verifies it, and reports any failures.

2. Technologies Used
Technology	Purpose
Red Hat Enterprise Linux	Target operating system (RHEL 8/9)
Ansible	Configuration management and automation
Bash	Shell tasks and ad-hoc administration
SSH	Secure connection to managed hosts (key-based auth)
systemd	Service management on target hosts
YAML	Playbooks, roles and variable files
Linux networking	Host connectivity, firewall rules
dnf	Package management (install/update)
3. Key Capabilities
3.1 Consistent configuration
An inventory groups all hosts (webservers, dbservers, combined as rhel) with connection variables per group.
Playbooks and reusable roles replace manual per-server setup, so every RHEL host converges to the same state.
Idempotent tasks: re-running a playbook only changes what is not yet correct.
3.2 Baseline administration tasks (roles)
Role	What it does
common	Installs baseline packages (vim-enhanced, tmux, firewalld, policycoreutils-python-utils), sets the timezone, enforces root-owned file permissions on /etc/sysconfig.

Applies only the selected packages (via patch_packages, e.g. httpd, openssl), so patches are deliberate, not blanket.
Reboots only when the kernel was updated.
Post-patch validation: collects service facts and prints a health report (e.g. free memory) so an operator can confirm the host is healthy.
3.4 Error handling and validation
Playbooks verify the expected state was reached, not just that commands ran: service_facts collection, explicit fail tasks when a required service is not running, and validate: clauses on config files.
sshd configuration changes are validated with /usr/sbin/sshd -t -f %s before being written — a typo cannot lock everyone out.
Failed tasks stop the run and are reported clearly in the PLAY RECAP (failed=0 means every host reached its expected state).
4. Project Structure
ansible-rhel-automation/
├── ansible.cfg              # Ansible defaults (inventory, roles path, SSH)
├── inventory/
│   └── production.ini       # Host groups and connection variables
├── group_vars/
│   └── rhel.yml             # Shared variables: admin_users, patch_packages
├── site.yml                 # Baseline configuration playbook
├── patch.yml                # Guarded, selective patching playbook
├── roles/
│   ├── common/              # packages, timezone, permissions
│   │   └── tasks/main.yml
│   ├── users/               # admin group, users, SSH keys
│   │   └── tasks/main.yml
│   ├── sshd/                # SSH hardening
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml   # Restart sshd
│   ├── services/            # systemd management + validation
│   │   └── tasks/main.yml
│   ├── httpd/               # Apache web tier
│   │   ├── tasks/main.yml
│   │   └── handlers/main.yml   # Restart httpd
│   └── firewall/            # firewalld rules
│       └── tasks/main.yml
└── README.md

5. Running the Automation
# Syntax check and dry run (no changes made)
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check --diff

# Run the full baseline configuration
ansible-playbook -i inventory/production.ini site.yml

# Patch only the web tier, one host at a time
ansible-playbook -i inventory/production.ini patch.yml \
  --limit webservers -e "patch_packages=[httpd,openssl]"

# Ad-hoc: verify all hosts are reachable
ansible rhel -i inventory/production.ini -m ping



Safe change management: dry runs, staged (serial) patching, pre/post health checks.
Validation-first automation: verify the result, fail loudly on drift.
