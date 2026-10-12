# Multi-App Infrastructure & Deployment Automation (Ansible)

A production-grade, 100% DRY (Don't Repeat Yourself), and role-based Ansible automation suite designed to provision cloud servers (AWS EC2 - Amazon Linux 2023 / RHEL / Fedora) and orchestrate multiple dockerized web applications concurrently behind an Nginx reverse proxy.

---

## 🏗️ What this Automation Handles

1. **Host Hardening & Resource Optimization (`roles/common`)**:
   - Automatically allocates a persistent **2GB swapfile** (`/swapfile`) with `/etc/fstab` persistence to keep memory-constrained `t3.micro` instances completely stable during multi-container builds.
   - Installs essential base tools (`procps-ng`, `git`, `tar`).

2. **Docker Engine & Docker Compose (`roles/docker`)**:
   - Installs Docker daemon, enables the systemd service, and ensures the non-root user (`ec2-user`) belongs to the `docker` group.
   - Automatically installs the official Docker Compose CLI plugin.

3. **Dynamic Nginx Reverse Proxy (`roles/nginx`)**:
   - Fixes the root port `80` catch-all issue with `default_server`, routing direct IP traffic to the designated root application (DevOps Portfolio).
   - Generates virtual hosts via a single unified Jinja2 template supporting subdomain wildcard routing (`*.nip.io` or custom domains), dedicated direct ports (e.g., `:8080`, `:8000`), custom file upload limits, and WebSocket proxies (e.g. `Socket.io`).

4. **Multi-App Container Orchestration (`roles/apps`)**:
   - Clones or pulls the latest code from GitHub for every application defined in `group_vars/all.yml`.
   - Executes `docker compose up -d --build` dynamically in each directory.
   - Prunes dangling docker images to prevent disk exhaustion.

---

## 📁 Repository Structure

```text
infrastructure-ansible/
├── ansible.cfg                          # Global Ansible defaults (inventory, SSH options, YAML output)
├── site.yml                             # Master orchestrator playbook
├── README.md                            # Comprehensive runbook and documentation
├── inventory/
│   └── hosts.ini                        # Target hosts and SSH connection parameters
├── group_vars/
│   └── all.yml                          # Centralized configuration (Apps, Swap, Paths)
└── roles/
    ├── common/                          # System packages & 2GB Swap setup
    │   └── tasks/main.yml
    ├── docker/                          # Docker Engine, Compose plugin, user groups
    │   └── tasks/main.yml
    ├── nginx/                           # Dynamic Nginx virtual hosts & default_server fix
    │   ├── tasks/main.yml
    │   ├── handlers/main.yml
    │   └── templates/
    │       ├── default_upstream.conf.j2 # Serves the designated default app at root http://<IP>/
    │       └── app_vhost.conf.j2        # Universal single template for all virtual hosts
    └── apps/                            # Generic Git clone & Docker Compose execution loop
        └── tasks/main.yml
```

---

## ⚡ Prerequisites

### 1. Install Ansible (Locally)
```bash
# macOS (via Homebrew)
brew install ansible

# Linux / Python (via pipx or pip)
pip install ansible
```

### 2. Verify Your SSH Key Permissions
Make sure your AWS EC2 key has restricted `400` permissions:
```bash
chmod 400 ~/Desktop/shirish-webserver.pem
```

*(If your key is stored elsewhere, adjust `ansible_ssh_private_key_file` in [`inventory/hosts.ini`](inventory/hosts.ini)).*

---

## 🚀 Step-by-Step Instructions for Running

### Step 1: Clone the Repo & Check Connectivity
```bash
git clone git@github.com:zugger14/infrastructure-ansible.git
cd infrastructure-ansible

# Test SSH connection to the server
ansible -m ping webservers
```

### Step 2: Run a Dry-Run (Check Mode)
Validate syntax and preview actions before making any server changes:
```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --check
```

### Step 3: Run the Full Playbook
Execute all roles across the host:
```bash
ansible-playbook site.yml
```

### Step 4: Selective Execution Using Tags
You don't need to re-run the full playbook when making small updates:
- **Only update Nginx reverse proxy configurations:**
  ```bash
  ansible-playbook site.yml --tags "nginx"
  ```
- **Only pull latest code & redeploy Docker containers:**
  ```bash
  ansible-playbook site.yml --tags "apps"
  ```
- **Only reconfigure Docker service or base packages:**
  ```bash
  ansible-playbook site.yml --tags "docker"
  ansible-playbook site.yml --tags "common"
  ```

---

## 💡 How to Add a New Application in the Future

The entire infrastructure is **100% DRY and configuration-driven**. To add a 4th or 5th application, **you never touch any task or template files**.

Simply open [`group_vars/all.yml`](group_vars/all.yml) and add an item to the `applications` list:

```yaml
applications:
  # ... existing apps ...

  - name: "my-new-app"
    repo_url: "https://github.com/zugger14/my-new-app.git"
    version: "main"                 # Git branch, tag, or commit
    service_port: 5000              # Port exposed by the container to localhost
    subdomain: "myapp"              # Accessible at http://myapp.<EC2_IP>.nip.io/
    direct_port: 8090               # (Optional) Accessible directly at http://<EC2_IP>:8090/
    client_max_body_size: "50M"     # (Optional) Default is 50M
    websocket_port: 6001            # (Optional) Set only if app requires WebSockets
```

Then execute:
```bash
ansible-playbook site.yml
```
Ansible will automatically clone the repository, run `docker compose up -d --build`, generate `/etc/nginx/conf.d/my-new-app.conf` from the universal template, and reload Nginx seamlessly without downtime.
