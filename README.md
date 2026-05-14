server-hardening
A public server without hardening is not a question of if it gets hit. It is a question of when.
This repo is a collection of Ansible playbooks that harden any fresh Ubuntu server before anything else runs on it. Firewall, SSH, intrusion prevention, automatic security updates, credential protection, SSL verification, and a full security audit. One command runs all of it.
Built and battle-tested on Hetzner Cloud running k3s. Works equally on plain servers with no Kubernetes.

What it does
ufw.yml configures the OS-level firewall. Default deny on all inbound traffic. SSH rate-limited to trusted IP ranges. HTTP and HTTPS open publicly. Kubernetes API port restricted to known ranges if applicable. All outbound allowed.
ssh-hardening.yml locks down the SSH daemon. Password authentication disabled. Root login restricted to key only. Max authentication attempts reduced to 3. Login grace time cut to 30 seconds. kubeconfig permissions locked to root if present.
users.yml creates a dedicated non-root application user with no login shell. Creates a restricted application directory. Audits sudo group membership and checks for empty passwords.
fail2ban.yml installs and configures Fail2Ban. Bans IPs after 3 failed SSH attempts within 10 minutes for 1 hour. Verifies the SSH jail is active after installation.
auto-updates.yml enables unattended security updates. Configures Ubuntu to apply security patches daily without manual intervention. Checks for pending kernel updates that require a reboot.
docker-credentials.yml installs the GPG-based credential helper for Docker. Stops registry credentials being stored in plain text. Skips gracefully if Docker is not installed.
docker-security.yml installs Trivy and scans every Docker image on the server for HIGH and CRITICAL CVEs. Skips gracefully if Docker is not installed.
ssl-renewal.yml verifies certbot is installed and the renewal timer is active. Runs a dry-run renewal to confirm configuration is valid. Lists all certificates and their expiry status. Installs certbot if not present.
audit.yml runs a full security audit and prints the results. Active sessions, login history, failed login attempts, running processes, network connections, recently modified system files, cron jobs, sudo users, and a scan for unexpected sensitive files.
site.yml runs all playbooks in the correct order with one command.

Requirements

Ansible installed on your control node (WSL Ubuntu recommended on Windows)
Target server running Ubuntu 22.04
SSH key access to the server
community.general Ansible collection (installed automatically by the playbook)


Setup
1. Clone the repo:
bashgit clone https://github.com/echezonaifejianyi-off/server-hardening.git
cd server-hardening
2. Create your inventory file:
ini[servers]
my-server ansible_host=YOUR_SERVER_IP

[servers:vars]
ansible_user=root
ansible_ssh_private_key_file=/path/to/your/key.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
Do not commit this file. It is in .gitignore for a reason.
3. Create ansible.cfg in the repo root:
ini[defaults]
host_key_checking = False

[ssh_connection]
ssh_args = -o ControlMaster=no -o ConnectTimeout=60 -o StrictHostKeyChecking=no
4. Update the IP ranges in ufw.yml:
Open ufw.yml and update the vars block to reflect your trusted IP ranges:
yamlvars:
  allowed_ssh_ranges:
    - "YOUR_IP_RANGE/CIDR"
  allowed_k8s_ranges:
    - "YOUR_IP_RANGE/CIDR"

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

Playbook structure
server-hardening/
├── site.yml                  master playbook, runs everything in order
├── ufw.yml                   OS-level firewall configuration
├── ssh-hardening.yml         SSH daemon hardening
├── users.yml                 non-root app user, sudo audit, password audit
├── fail2ban.yml              intrusion prevention for SSH
├── auto-updates.yml          automatic security patch application
├── docker-credentials.yml    GPG credential helper for Docker
├── docker-security.yml       Trivy image scanning for CVEs
├── ssl-renewal.yml           certbot installation and renewal verification
└── audit.yml                 full security audit and reporting

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
Scan for unexpected .env, .pem, and .key files outside known system paths

If unexpected sensitive files are found, the playbook fails loudly and stops. You investigate before continuing.

What this does not cover
This playbook hardens the OS and SSH layer. It does not:

Configure application-level security (RBAC, network policies, secrets encryption)
Set up a cloud provider firewall (Hetzner, AWS security groups, etc.)
Rotate existing credentials
Scan running containers for runtime threats

For cloud provider firewall configuration, set that up before running this playbook. Defence in depth means both layers — cloud firewall and OS firewall — should be in place.

Tested on

Ubuntu 22.04 LTS
Hetzner Cloud CX22 (k3s node)
Hetzner Cloud plain server