server-hardening
A public server without hardening is not a question of if it gets hit. It is a question of when.
This repo is a collection of Ansible playbooks that harden any fresh Ubuntu server before anything else runs on it. Firewall, SSH, intrusion prevention, automatic security updates, credential protection, SSL verification, and a full security audit. One command runs all of it.
Built and battle-tested on Hetzner Cloud running k3s. Works equally on plain servers with no Kubernetes.

What it does
ufw.yml configures the OS-level firewall. Default deny on all inbound traffic. SSH rate-limited to trusted IP ranges. HTTP and HTTPS open publicly. Kubernetes API port restricted to known ranges if applicable. All outbound allowed.
ssh-hardening.yml locks down the SSH daemon. Password authentication disabled. Root login restricted to key only. Max authentication attempts and login grace time configurable via variables. kubeconfig permissions locked to root if present.
users.yml creates a dedicated non-root application user with no login shell. Creates a restricted application directory. Audits sudo group membership and checks for empty passwords.
fail2ban.yml installs and configures Fail2Ban. Ban thresholds, duration, and SSH port are all configurable via variables. Verifies the SSH jail is active after installation.
auto-updates.yml enables unattended security updates. Configures Ubuntu to apply security patches daily without manual intervention. Checks for pending kernel updates that require a reboot.
docker-credentials.yml installs the GPG-based credential helper for Docker. Stops registry credentials being stored in plain text. Skips gracefully if Docker is not installed.
docker-security.yml installs Trivy and scans every Docker image on the server for HIGH and CRITICAL CVEs. Skips gracefully if Docker is not installed.
ssl-renewal.yml verifies certbot is installed and the renewal timer is active. Runs a dry-run renewal to confirm configuration is valid. Lists all certificates and their expiry status. If domain_name and certbot_email are set in your variables file, automatically obtains an SSL certificate on first run.
audit.yml runs a full security audit and prints the results. Active sessions, login history, failed login attempts, running processes, network connections, recently modified system files, cron jobs, sudo users, and a scan for unexpected sensitive files. Excluded paths and sensitive file patterns are fully configurable.
site.yml runs all playbooks in the correct order with one command.

Requirements

Ansible installed on your control node (WSL Ubuntu recommended on Windows)
Target server running Ubuntu 22.04
SSH key access to the server
community.general Ansible collection (installed automatically by the playbook)


Setup
Three files need to be created locally from their examples before running the playbook. None of them get committed — they are all in .gitignore. Your server IPs, key paths, and domain names stay on your machine.
1. Clone the repo:
bashgit clone https://github.com/echezonaifejianyi-off/server-hardening.git
cd server-hardening
2. Create your inventory file:
bashcp inventory.ini.example inventory.ini
Open inventory.ini and fill in your server IP and SSH key path:
ini[servers]
my-server ansible_host=YOUR_SERVER_IP

[servers:vars]
ansible_user=root
ansible_ssh_private_key_file=/path/to/your/key.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
3. Create your Ansible config:
bashcp ansible.cfg.example ansible.cfg
No edits needed unless you want to change SSH connection behaviour.
4. Create your variables file:
bashcp group_vars/all.yml.example group_vars/all.yml
Open group_vars/all.yml and fill in your values. Every variable is documented with a comment in the example file. The key ones to update:

allowed_ssh_ranges — IP ranges allowed to SSH into the server e.g. 102.89.0.0/16
allowed_k8s_ranges — IP ranges allowed to reach port 6443. Leave empty if not running Kubernetes
app_user and app_dir — the non-root user and directory for your application
domain_name and certbot_email — required only if you want certbot to obtain an SSL certificate automatically
fail2ban_maxretry, fail2ban_bantime, fail2ban_findtime — tunable Fail2Ban thresholds
max_auth_tries and login_grace_time — SSH hardening values
sensitive_file_patterns — file patterns the audit will flag
excluded_audit_paths — paths excluded from the sensitive file scan

This file stays local. It is in .gitignore and will never be committed.

Usage
Test connectivity first:
bashansible all -i inventory.ini -m ping
Dry run — see what will change without touching anything:
bashansible-playbook -i inventory.ini site.yml --check
Full hardening run:
bashansible-playbook -i inventory.ini site.yml
Run a single playbook:
bashansible-playbook -i inventory.ini fail2ban.yml
ansible-playbook -i inventory.ini audit.yml

Repo structure
server-hardening/
site.yml                    master playbook, runs everything in order
ufw.yml                     OS-level firewall configuration
ssh-hardening.yml           SSH daemon hardening
users.yml                   non-root app user, sudo audit, password audit
fail2ban.yml                intrusion prevention for SSH
auto-updates.yml            automatic security patch application
docker-credentials.yml      GPG credential helper for Docker
docker-security.yml         Trivy image scanning for CVEs
ssl-renewal.yml             certbot installation and renewal verification
audit.yml                   full security audit and reporting
group_vars/
  all.yml.example           variable template — copy to all.yml and edit
inventory.ini.example       inventory template — copy to inventory.ini and edit
ansible.cfg.example         Ansible config template — copy to ansible.cfg

What the audit checks
Every run of site.yml ends with a full audit report printed to your terminal:

Who is currently logged in
Last 20 login attempts with IP addresses
Last 20 failed SSH attempts
Top 20 processes by CPU usage
All active network connections
Files modified in the last 7 days in sensitive system directories
Root crontab and cron.d contents
All users with sudo access
Scan for sensitive files matching patterns in sensitive_file_patterns, excluding paths in excluded_audit_paths

If unexpected sensitive files are found, the playbook fails loudly and stops. You investigate before continuing.

What this does not cover
This playbook hardens the OS and SSH layer. It does not:

Configure application-level security (RBAC, network policies, secrets encryption)
Set up a cloud provider firewall (Hetzner, AWS security groups, etc.)
Rotate existing credentials
Scan running containers for runtime threats

Set up your cloud provider firewall before running this playbook. Defence in depth means both layers working together.

Tested on

Ubuntu 22.04 LTS
Hetzner Cloud CX22 (k3s node)
Hetzner Cloud plain server