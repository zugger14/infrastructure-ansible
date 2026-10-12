# Multi-App Infrastructure & Deployment Automation (Ansible)

A production-grade, 100% DRY (Don't Repeat Yourself), and role-based Ansible automation repository for provisioning AWS EC2 instances and orchestrating multiple dockerized web applications behind an Nginx reverse proxy.

---

## 🌟 How the DRY Architecture Works

Previously, adding a new project required creating new tasks, new paths, and a dedicated `.conf` Nginx template for every app.

Now, the entire system is **driven purely by configuration**. A **single unified Jinja2 template** handles:
- Subdomain routing on Port 80 (e.g., `myapp.*.nip.io` or `myapp.domain.com`)
- Optional direct public port routing (e.g., `http://<IP>:8090/`)
- Optional WebSocket reverse proxying (e.g., Socket.io, ws)
- Custom upload size limits (`client_max_body_size`)
- Automatic Git checkout & Docker Compose container lifecycle

---

## 💡 How to Add a New Application

To deploy another app in the future, **you do not touch any code or template files**. 
Simply append an entry to `applications` in [`group_vars/all.yml`](file:///Users/shirish/vibe-coding-apps/infrastructure-ansible/group_vars/all.yml):

```yaml
applications:
  # ... existing apps ...

  - name: "my-new-app"
    repo_url: "https://github.com/username/my-new-app.git"
    version: "main"                 # Git branch, tag, or commit hash
    service_port: 5000              # Internal container port
    subdomain: "myapp"              # Subdomain on Port 80 (e.g. myapp.34.229.18.255.nip.io)
    direct_port: 8090               # (Optional) Direct port (e.g. http://34.229.18.255:8090/)
    client_max_body_size: "50M"     # (Optional) Default is 50M
    websocket_port: 6001            # (Optional) If app uses WebSocket
    websocket_path: "/socket.io/"   # (Optional) Default is /socket.io/
```

Then run:
```bash
ansible-playbook site.yml
```
Ansible will automatically:
1. Clone the repository into `/home/ec2-user/my-new-app`
2. Run `docker compose up -d --build`
3. Generate `/etc/nginx/conf.d/my-new-app.conf` from the generic template
4. Test and reload Nginx without any downtime

---

## 📁 Repository Structure

```text
infrastructure-ansible/
├── ansible.cfg                          # Connection and execution settings
├── site.yml                             # Master orchestrator playbook
├── README.md                            # Documentation
├── inventory/
│   └── hosts.ini                        # Target hosts and SSH settings
├── group_vars/
│   └── all.yml                          # Centralized configuration (Apps, Swap, Paths)
└── roles/
    ├── common/                          # Base system packages & persistent 2GB swapfile
    │   └── tasks/main.yml
    ├── docker/                          # Docker Engine, Compose plugin, group membership
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

## 🚀 Execution Commands

From this folder:
```bash
# 1. Ping test
ansible -m ping webservers

# 2. Deploy everything
ansible-playbook site.yml

# 3. Only update Nginx configs without touching Docker
ansible-playbook site.yml --tags "nginx"

# 4. Only pull & restart Docker containers
ansible-playbook site.yml --tags "apps"
```
