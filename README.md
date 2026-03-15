# Ansible Lab — Setting Up an Initial Ansible Project

A hands-on lab demonstrating how to install Ansible on WSL2 (Windows), connect to AWS EC2 instances, and automate server configuration using playbooks.

---

## Lab Architecture

```
Windows PC (WSL2 - Ubuntu)
└── Ansible Control Node
        │
        ├── SSH (port 22)
        │
        ├── my-server-1 (AWS EC2 - Ubuntu 24.04)
        └── my-server-2 (AWS EC2 - Ubuntu 24.04)
```

---

## Prerequisites

- Windows 10/11
- AWS Account with 2 running Ubuntu EC2 instances
- `.pem` key file for EC2 SSH access
- EC2 Security Group with port 22 (SSH) and port 80 (HTTP) open

---

## Project Structure

```
ansible-project/
├── ansible.cfg          # Ansible configuration file
├── inventory/
│   └── hosts            # EC2 instance inventory
├── playbooks/
│   └── setup.yml        # Main server setup playbook
├── vars/                # Variable files (for future use)
├── roles/               # Roles directory (for future use)
└── README.md
```

---

## Step-by-Step Setup

### Step 1 — Install WSL2 on Windows

Open **PowerShell as Administrator** and run:

```powershell
wsl --install -d Ubuntu
```

- Restart your PC when prompted
- After restart, Ubuntu will launch automatically
- Set a **username and password** when asked

Verify WSL2 is installed:

```powershell
wsl --list --verbose
# Should show: Ubuntu    Running    2
```

---

### Step 2 — Install Ansible in WSL2

Open your **Ubuntu WSL2 terminal** and run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y software-properties-common python3 python3-pip
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
ansible --version
```

Expected output:
```
ansible [core 2.16.3]
```

---

### Step 3 — Copy .pem Key into WSL2

```bash
# Create SSH directory
mkdir ~/.ssh

# Copy key from Windows Downloads folder into WSL2
cp /mnt/c/Users/<YourWindowsUsername>/Downloads/server-key.pem ~/.ssh/

# Fix permissions (required by SSH)
chmod 400 ~/.ssh/server-key.pem
```

> **Note:** In WSL2, your Windows `C:` drive is accessible at `/mnt/c/`

---

### Step 4 — Test SSH to EC2 Instances

```bash
# Test node 1
ssh -i ~/.ssh/server-key.pem ubuntu@<NODE1_PUBLIC_IP>
# Type 'yes' for fingerprint, then 'exit' to return

# Test node 2
ssh -i ~/.ssh/server-key.pem ubuntu@<NODE2_PUBLIC_IP>
# Type 'yes' for fingerprint, then 'exit' to return
```

---

### Step 5 — Create Ansible Project Structure

```bash
mkdir ~/ansible-project && cd ~/ansible-project
mkdir -p inventory playbooks vars roles
```

---

### Step 6 — Create ansible.cfg

```bash
cat > ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory/hosts
remote_user = ubuntu
private_key_file = ~/.ssh/server-key.pem
host_key_checking = False
EOF
```

| Setting | Purpose |
|---|---|
| `inventory` | Path to inventory file |
| `remote_user` | Default SSH user for EC2 (ubuntu) |
| `private_key_file` | Path to your .pem key |
| `host_key_checking` | Disable SSH fingerprint prompt |

---

### Step 7 — Create Inventory File

```bash
cat > inventory/hosts << 'EOF'
[webservers]
my-server-1 ansible_host=<NODE1_PUBLIC_IP>
my-server-2 ansible_host=<NODE2_PUBLIC_IP>

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/server-key.pem
EOF
```

> Replace `<NODE1_PUBLIC_IP>` and `<NODE2_PUBLIC_IP>` with your actual EC2 Public IPs from AWS Console.

---

### Step 8 — Test Ansible Connectivity

```bash
ansible all -m ping
```

Expected output:
```
my-server-1 | SUCCESS => {
    "ping": "pong"
}
my-server-2 | SUCCESS => {
    "ping": "pong"
}
```

---

### Step 9 — Create the Playbook

```bash
cat > playbooks/setup.yml << 'EOF'
---
- name: Initial Server Setup
  hosts: webservers
  become: yes

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install nginx, git, curl
      apt:
        name:
          - nginx
          - git
          - curl
        state: present

    - name: Start and enable nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Create index page
      copy:
        content: "Server managed by Ansible!\n"
        dest: /var/www/html/index.html
        owner: www-data
        mode: '0644'
EOF
```

**Playbook Tasks Explained:**

| Task | What it does |
|---|---|
| Update apt cache | Refreshes package list |
| Install packages | Installs nginx, git, curl |
| Start nginx | Starts nginx and enables it on boot |
| Create index page | Deploys a custom HTML file |

---

### Step 10 — Run the Playbook

**Dry run first (no actual changes):**
```bash
ansible-playbook playbooks/setup.yml --check
```

**Run for real:**
```bash
ansible-playbook playbooks/setup.yml
```

Expected output:
```
PLAY RECAP
my-server-1 : ok=5  changed=3  unreachable=0  failed=0
my-server-2 : ok=5  changed=3  unreachable=0  failed=0
```

---

### Step 11 — Verify Results

**From terminal:**
```bash
ansible webservers -m command -a "curl localhost"
```

Expected output:
```
my-server-1 | CHANGED | rc=0 >>
Server managed by Ansible!

my-server-2 | CHANGED | rc=0 >>
Server managed by Ansible!
```

**From browser:**
- `http://<NODE1_PUBLIC_IP>` → Should show `Server managed by Ansible!`
- `http://<NODE2_PUBLIC_IP>` → Should show `Server managed by Ansible!`

---

## Key Ansible Commands

| Command | Purpose |
|---|---|
| `ansible all -m ping` | Test connectivity to all hosts |
| `ansible-playbook setup.yml --check` | Dry run (no changes) |
| `ansible-playbook setup.yml` | Run playbook |
| `ansible-playbook setup.yml -v` | Run with verbose output |
| `ansible webservers -m command -a "uptime"` | Run ad-hoc command |
| `ansible all -m command -a "df -h"` | Check disk space on all hosts |

---

## Important Notes

- **Ansible only runs on Linux/macOS** as a control node — that's why WSL2 is needed on Windows
- Always run `--check` (dry run) before applying changes to production servers
- Use **Private IPs** if all EC2 instances are in the same VPC; use **Public IPs** if control node is outside AWS
- **Stop EC2 instances** after the lab to avoid AWS charges

---

## Tech Stack

- **Control Node:** WSL2 (Ubuntu 22.04) on Windows
- **Managed Nodes:** AWS EC2 (Ubuntu 24.04) — t3.micro
- **Ansible Version:** 2.16.3
- **Region:** us-east-1
