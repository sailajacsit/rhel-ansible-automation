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
users	Creates the sysadmins group, creates admin users with wheel membership, and deploys their authorized SSH public keys.
sshd	Hardens the SSH daemon: disables root login and password authentication, limits auth tries. Uses sshd -t validation so a broken config can never be written.
services	Manages system services through systemd (e.g. keeps firewalld enabled and running) and validates the service state after deployment.
httpd	Web tier: installs Apache, deploys the index page, ensures the service is enabled and running, restarts it on config change.
firewall	Opens only the required web service ports (http, https) in firewalld, permanently and immediately.
3.3 Patching with guardrails (patch.yml)
Patches one server at a time (serial: 1) so the service stays available.
Pre-patch health check: fails the run if the root filesystem usage is above 85% — the patch is aborted before anything changes.
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
5. Inventory Explained (inventory/production.ini)
[webservers]
web01.example.com ansible_host=10.0.1.11
web02.example.com ansible_host=10.0.1.12

[dbservers]
db01.example.com  ansible_host=10.0.2.21

[rhel:children]      # group of groups — every RHEL host
webservers
dbservers

[rhel:vars]          # connection settings for all RHEL hosts
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
site.yml targets hosts: rhel (baseline for everyone).
The web tier roles target hosts: webservers only.
6. Key Playbooks
6.1 site.yml — baseline configuration
- name: Baseline RHEL configuration
  hosts: rhel
  become: true
  roles:
    - common
    - users
    - sshd
    - services

- name: Web tier configuration
  hosts: webservers
  become: true
  roles:
    - httpd
    - firewall
become: true means tasks run with elevated (sudo) privileges.

6.2 patch.yml — selective patching with health checks
serial: 1 — one server at a time.
pre_tasks: reads disk usage (df -h /) and fails the run if the root filesystem is above 85%.
tasks: updates only the packages in patch_packages; reboots if the kernel package was updated.
post_tasks: verifies critical services and prints a health report.
7. Variables (group_vars/rhel.yml)
admin_users:            # created by the users role
  - name: sailaja
    ssh_key: "ssh-ed25519 AAAA... user@laptop"

patch_packages:         # patched by patch.yml
  - kernel
  - httpd
  - openssl
You can override patch_packages per run: -e "patch_packages=[httpd,openssl]".

8. Running the Automation
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
Expected output
A successful run ends with a recap like:

PLAY RECAP
web01.example.com : ok=42  changed=12  failed=0
web02.example.com : ok=42  changed=12  failed=0
db01.example.com  : ok=42  changed=12  failed=0
failed=0 — every host reached its expected state.

9. Requirements
Control node: Ansible 2.12 or newer; SSH access to all target hosts.
Target hosts: RHEL 8/9 with the ansible user provisioned, key-based SSH login, and passwordless sudo (for become: true).
Collection: ansible.posix — install with ansible-galaxy collection install ansible.posix.
10. Before Using With Real Servers
Replace the example hostnames and IPs in inventory/production.ini with your real servers.
Replace the placeholder SSH public keys in group_vars/rhel.yml with real keys for your admin users.
Test with --check --diff (dry run) before the first real run.
Keep serial: 1 for patching production systems so one failing host cannot affect the rest of the fleet.
11. What This Project Demonstrates
Infrastructure-as-code thinking: server state described in versionable YAML.
Idempotent, reusable roles instead of one-off scripts.
Safe change management: dry runs, staged (serial) patching, pre/post health checks.
Validation-first automation: verify the result, fail loudly on drift.
