# Multi-App Infrastructure & Deployment Automation (Ansible)

This directory provides a production-grade, modular, and reusable Ansible setup to configure an AWS EC2 instance (Amazon Linux 2023 / RHEL / Fedora) and run all three web applications simultaneously:
1. **DevOps Portfolio** (Next.js 16) - Port `80` default upstream via `127.0.0.1:3000`
2. **AMS** (Appointment Management System - CodeIgniter + MariaDB) - Subdomain on Port `80` & Direct Port `8080` via `127.0.0.1:8081`
3. **MusicBook** (Laravel + MariaDB + Redis + Socket.io) - Subdomain on Port `80` & Direct Port `8000` via `127.0.0.1:8001` + WebSocket `8890`

---

## 📁 Directory Structure

```text
ansible/
├── ansible.cfg                    # Ansible default options (YAML output, SSH config)
├── inventory/
│   └── hosts.ini                  # Target hosts and SSH connection settings
├── group_vars/
│   └── all.yml                    # Global definitions, swap size, applications list
├── site.yml                       # Primary orchestrator playbook
└── roles/
    ├── common/                    # Base system tools & 2GB Swap configuration
    │   └── tasks/main.yml
    ├── docker/                    # Docker Engine & Docker Compose plugin
    │   └── tasks/main.yml
    ├── nginx/                     # Reverse proxy configuration & default_server fix
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── portfolio.conf.j2
    │       ├── ams.conf.j2
    │       └── musicbook.conf.j2
    └── apps/                      # Git checkout and docker compose multi-container orchestration
        └── tasks/main.yml
```

---

## 🚀 Prerequisites

1. **Install Ansible** (on your local machine):
   ```bash
   # macOS via Homebrew
   brew install ansible

   # OR via pipx / pip
   pip install ansible
   ```

2. **Verify SSH Key**:
   Make sure your private key (e.g. `~/Desktop/shirish-webserver.pem`) has `400` permissions:
   ```bash
   chmod 400 ~/Desktop/shirish-webserver.pem
   ```

---

## ⚙️ Usage

### 1. Test Connection
```bash
ansible -m ping webservers
```

### 2. Dry-Run / Syntax Check
```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check
```

### 3. Run the Full Provisioning & Deployment
```bash
ansible-playbook site.yml
```

### 4. Run Selective Roles using Tags
You can execute specific roles without re-running everything:
- Only update Nginx proxy configurations:
  ```bash
  ansible-playbook site.yml --tags "nginx"
  ```
- Only rebuild/redeploy docker applications:
  ```bash
  ansible-playbook site.yml --tags "apps"
  ```
- Only configure base system and swap:
  ```bash
  ansible-playbook site.yml --tags "common"
  ```

---

## 🔒 Adding New Apps in the Future
To add another application, simply add a new entry to the `applications` list in `group_vars/all.yml` and provide its corresponding Nginx template in `roles/nginx/templates/`. Everything else is automated.
