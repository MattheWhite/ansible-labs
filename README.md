# Ansible Labs

Ansible is an open-source automation engine for configuration management, application deployment, and orchestration. You describe the desired state of your systems in simple YAML files, and Ansible makes it so.

---

## 1. What Is Ansible & Why Use It

Ansible is an open-source **automation engine** for configuration management, application deployment, and orchestration. You describe the *desired state* of your systems in simple YAML files, and Ansible makes it so.

**Why teams pick Ansible:**

- **Agentless** — no software to install on managed servers. It uses SSH (Linux) or WinRM (Windows). Compare: Puppet/Chef require agents.
- **Idempotent** — running the same playbook twice doesn't break anything. Tasks only change what needs changing.
- **Human-readable YAML** — playbooks double as documentation.
- **Batteries included** — thousands of modules for Linux, Windows, cloud (AWS/Azure/GCP), networking gear, Kubernetes, databases, and more.
- **Push-based** — you run it from a control node when *you* decide; nothing polls.

**Real-world usage:** provisioning cloud fleets, deploying apps with zero downtime, enforcing CIS security baselines, patching hundreds of servers overnight, configuring Cisco/Juniper network devices, onboarding user accounts, disaster-recovery runbooks.

---

## 2. Architecture & Core Concepts

```
┌──────────────────┐        SSH / WinRM       ┌───────────────┐
│  Control Node    │ ───────────────────────▶ │ Managed Node │ web1
│  (your laptop /  │ ───────────────────────▶ │ Managed Node │ web2
│   CI runner /    │ ───────────────────────▶ │ Managed Node │ db1
│   AWX server)    │                          └───────────────┘
│                  │
│  - ansible core  │   Ansible copies small Python "module"
│  - inventory     │   scripts to each node, runs them,
│  - playbooks     │   collects JSON results, deletes them.
└──────────────────┘
```

| Term | Meaning |
|---|---|
| **Control node** | Machine where Ansible is installed and run from (Linux/macOS/WSL — not native Windows) |
| **Managed node** | Server being configured. Needs only SSH + Python |
| **Inventory** | File/plugin listing your hosts and groups |
| **Module** | Unit of work (`apt`, `copy`, `service`, `ec2_instance`…) |
| **Task** | One module call with arguments |
| **Play** | Set of tasks mapped to a group of hosts |
| **Playbook** | YAML file of one or more plays |
| **Role** | Reusable, structured bundle of tasks/vars/templates |
| **Collection** | Distribution format for roles + modules (e.g. `amazon.aws`) |
| **Facts** | System info Ansible auto-gathers from each host |
| **Handler** | Task triggered only when something changed (e.g. restart nginx) |
| **Idempotency** | Re-running produces the same end state, reporting `ok` instead of `changed` |

---

## 3. Installation & Lab Setup

### Install Ansible (control node)

```bash
# Recommended: pipx or pip (latest version)
pipx install --include-deps ansible
# or
python3 -m pip install --user ansible

# Ubuntu/Debian
sudo apt update && sudo apt install -y ansible

# RHEL/Fedora
sudo dnf install -y ansible-core

# macOS
brew install ansible

# Verify
ansible --version
```


> **Windows users:** run the control node inside WSL2. Windows machines can be *managed* by Ansible (via WinRM/SSH) but can't natively run it.

### Build a free practice lab

**Option A — Docker containers as "servers" (fastest):**

```bash
docker run -d --name web1 -p 2221:22 rastasheep/ubuntu-sshd:18.04
docker run -d --name web2 -p 2222:22 rastasheep/ubuntu-sshd:18.04
```

**Option B — Vagrant + VirtualBox:**

```ruby
# Vagrantfile
Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  (1..2).each do |i|
    config.vm.define "web#{i}" do |node|
      node.vm.hostname = "web#{i}"
      node.vm.network "private_network", ip: "192.168.56.1#{i}"
    end
  end
end
```

**Option C — two cheap cloud VMs** (AWS free tier, Hetzner, DigitalOcean).

### SSH key setup (do this once)

```bash
ssh-keygen -t ed25519 -C "ansible"
ssh-copy-id user@192.168.56.11
ssh-copy-id user@192.168.56.12
```

### Project layout & ansible.cfg

```bash
mkdir ansible-lab && cd ansible-lab
```

```ini
# ansible.cfg
[defaults]
inventory = ./inventory.ini
host_key_checking = False        # lab only — keep True in production
interpreter_python = auto_silent
forks = 20

[privilege_escalation]
become = True
become_method = sudo
```

---

## 4. Inventory — Defining Your Servers

### INI format (simple)

```ini
# inventory.ini
[webservers]
web1 ansible_host=192.168.56.11
web2 ansible_host=192.168.56.12

[dbservers]
db1 ansible_host=192.168.56.21

[production:children]
webservers
dbservers

[webservers:vars]
ansible_user=ubuntu
http_port=80
```

### YAML format (preferred for larger setups)

```yaml
# inventory.yml
all:
  children:
    webservers:
      hosts:
        web1:
          ansible_host: 192.168.56.11
        web2:
          ansible_host: 192.168.56.12
      vars:
        ansible_user: ubuntu
    dbservers:
      hosts:
        db1:
          ansible_host: 192.168.56.21
```

### group_vars / host_vars (the industry way)

Keep variables out of the inventory file:

```
inventory.ini
group_vars/
  all.yml           # applies to every host
  webservers.yml    # applies to the webservers group
host_vars/
  web1.yml          # applies only to web1
```

### Verify connectivity

```bash
ansible all -m ping                 # not ICMP — a full SSH+Python round trip
ansible webservers --list-hosts
ansible-inventory --graph
```

Expected output:

```
web1 | SUCCESS => { "changed": false, "ping": "pong" }
```

---

## 5. Ad-Hoc Commands

One-off commands without writing a playbook — great for investigation and quick fixes across a fleet.

```bash
# Run a shell command everywhere
ansible all -m ansible.builtin.command -a "uptime"

# Check disk space on web servers
ansible webservers -a "df -h /"

# Install a package (needs sudo → -b for "become")
ansible webservers -m ansible.builtin.apt -a "name=htop state=present" -b

# Restart a service
ansible webservers -m ansible.builtin.service -a "name=nginx state=restarted" -b

# Copy a file to all hosts
ansible all -m ansible.builtin.copy -a "src=motd dest=/etc/motd" -b

# Gather all facts about a host
ansible web1 -m ansible.builtin.setup

# Just one fact
ansible web1 -m ansible.builtin.setup -a "filter=ansible_memtotal_mb"

# Reboot every host in a group, 5 at a time
ansible webservers -m ansible.builtin.reboot -b -f 5
```

**Industry scenario:** security asks "which of our 300 servers still run OpenSSL 1.x?" — answer in one line:

```bash
ansible all -a "openssl version" | sort
```

---

## 6. Playbooks — The Heart of Ansible

### Your first playbook

```yaml
# webserver.yml
---
- name: Configure web servers
  hosts: webservers
  become: true                      # run tasks with sudo

  tasks:
    - name: Install nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure nginx is running and enabled at boot
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true

    - name: Deploy index page
      ansible.builtin.copy:
        content: "<h1>Deployed by Ansible on {{ inventory_hostname }}</h1>\n"
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: "0644"
```

Run it:

```bash
ansible-playbook webserver.yml

# Useful flags
ansible-playbook webserver.yml --check        # dry run — show what WOULD change
ansible-playbook webserver.yml --diff         # show file diffs
ansible-playbook webserver.yml --limit web1   # only one host
ansible-playbook webserver.yml -v             # verbose (-vvv for debugging)
ansible-playbook webserver.yml --syntax-check
```

### Reading the output

```
TASK [Install nginx] ***********************************
changed: [web1]        ← Ansible did something
ok: [web2]             ← already in desired state (idempotency!)

PLAY RECAP *********************************************
web1  : ok=3  changed=1  unreachable=0  failed=0
```

Run it a second time — everything reports `ok`, nothing `changed`. That's idempotency, and it's why Ansible is safe to run repeatedly (including from cron/CI).

### Multiple plays in one playbook

```yaml
---
- name: Configure databases first
  hosts: dbservers
  become: true
  tasks:
    - name: Install PostgreSQL
      ansible.builtin.apt:
        name: postgresql
        state: present

- name: Then configure web tier
  hosts: webservers
  become: true
  tasks:
    - name: Install app dependencies
      ansible.builtin.apt:
        name: [python3-pip, git]
        state: present
```

### Essential modules to know

| Module (FQCN) | Purpose |
|---|---|
| `ansible.builtin.apt` / `dnf` / `package` | Package management |
| `ansible.builtin.service` / `systemd_service` | Manage services |
| `ansible.builtin.copy` | Copy files/content to hosts |
| `ansible.builtin.template` | Render Jinja2 templates |
| `ansible.builtin.file` | Create dirs, set permissions, symlinks |
| `ansible.builtin.lineinfile` / `blockinfile` | Edit config lines |
| `ansible.builtin.user` / `group` | Manage accounts |
| `ansible.builtin.git` | Clone/checkout repos |
| `ansible.builtin.command` / `shell` | Run raw commands (last resort — not idempotent by default) |
| `ansible.builtin.uri` | HTTP requests / health checks |
| `ansible.builtin.cron` | Cron jobs |
| `ansible.builtin.unarchive` | Extract tarballs (can download too) |
| `ansible.posix.mount` | Filesystems |
| `community.general.ufw` | Firewall |

> **FQCN note:** modern Ansible uses Fully Qualified Collection Names (`ansible.builtin.apt` instead of `apt`). Short names still work for builtins, but FQCN is the professional standard — use it everywhere.

---

## 7. Variables, Facts & Precedence

### Defining variables

```yaml
- name: Variables demo
  hosts: webservers
  vars:
    http_port: 8080
    app_user: deploy
  vars_files:
    - vars/common.yml
  tasks:
    - name: Show a variable
      ansible.builtin.debug:
        msg: "Port is {{ http_port }}, user is {{ app_user }}"
```

Or in `group_vars/webservers.yml`:

```yaml
http_port: 8080
app_env: production
packages:
  - nginx
  - git
  - unzip
db:
  host: 10.0.0.5
  port: 5432
```

Access nested values: `{{ db.host }}` or `{{ db['host'] }}`.

### Facts — free data about every host

Ansible gathers facts at the start of each play:

```yaml
- name: Use facts
  ansible.builtin.debug:
    msg: >
      {{ inventory_hostname }} runs {{ ansible_facts['distribution'] }}
      {{ ansible_facts['distribution_version'] }} with
      {{ ansible_facts['memtotal_mb'] }}MB RAM on
      {{ ansible_facts['default_ipv4']['address'] }}
```

### Registered variables — capture task output

```yaml
- name: Check app version
  ansible.builtin.command: /opt/app/bin/app --version
  register: app_version
  changed_when: false          # reading state ≠ a change

- name: Show it
  ansible.builtin.debug:
    var: app_version.stdout
```

### Runtime variables

```bash
ansible-playbook deploy.yml -e "app_version=2.4.1 env=staging"
```

### Precedence (simplified, lowest → highest)

1. role defaults (`roles/x/defaults/main.yml`)
2. inventory group_vars → host_vars
3. play `vars:` and `vars_files:`
4. task vars
5. `-e` extra vars on the CLI — **always wins**

**Rule of thumb:** put safe defaults in role defaults, environment-specific values in group_vars, and use `-e` only for one-off overrides.

---

## 8. Conditionals, Loops & Error Handling

### Conditionals (`when`)

```yaml
- name: Install Apache on RedHat family
  ansible.builtin.dnf:
    name: httpd
    state: present
  when: ansible_facts['os_family'] == "RedHat"

- name: Install Apache on Debian family
  ansible.builtin.apt:
    name: apache2
    state: present
  when: ansible_facts['os_family'] == "Debian"

- name: Only in production
  ansible.builtin.include_tasks: harden.yml
  when:
    - app_env == "production"          # list items are AND-ed
    - ansible_facts['memtotal_mb'] > 2048
```

### Loops

```yaml
- name: Install packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop: [nginx, git, curl, unzip]
# NOTE: package modules accept lists directly — name: "{{ packages }}" is faster.

- name: Create users with details
  ansible.builtin.user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: /bin/bash
  loop:
    - { name: alice, groups: "sudo" }
    - { name: bob,   groups: "developers" }

- name: Loop over a dict
  ansible.builtin.debug:
    msg: "{{ item.key }} = {{ item.value }}"
  loop: "{{ app_settings | dict2items }}"
```

### Error handling

```yaml
- name: Try risky thing, don't abort the play
  ansible.builtin.command: /opt/legacy/flaky-script.sh
  register: result
  ignore_errors: true

- name: Custom failure condition
  ansible.builtin.command: /usr/bin/health-check
  register: health
  failed_when: "'UNHEALTHY' in health.stdout"
  changed_when: false

- name: Retry until service answers
  ansible.builtin.uri:
    url: http://localhost:8080/health
    status_code: 200
  register: ping_result
  until: ping_result.status == 200
  retries: 10
  delay: 5

# Transaction-style: try / catch / finally
- name: Deploy with rollback
  block:
    - name: Deploy new release
      ansible.builtin.unarchive:
        src: "app-{{ app_version }}.tar.gz"
        dest: /opt/app
  rescue:
    - name: Roll back symlink
      ansible.builtin.file:
        src: "/opt/app/releases/{{ previous_version }}"
        dest: /opt/app/current
        state: link
  always:
    - name: Notify Slack
      ansible.builtin.uri:
        url: "{{ slack_webhook }}"
        method: POST
        body_format: json
        body: { text: "Deploy finished on {{ inventory_hostname }}" }
```

---

## 9. Templates (Jinja2)

Templates generate per-host config files — one of the most-used features in real environments.

```jinja
{# templates/nginx.conf.j2 #}
# Managed by Ansible — manual edits will be overwritten
user www-data;
worker_processes {{ ansible_facts['processor_vcpus'] }};

events {
    worker_connections 1024;
}

http {
    upstream app_backend {
{% for host in groups['appservers'] %}
        server {{ hostvars[host]['ansible_host'] }}:{{ app_port }};
{% endfor %}
    }

    server {
        listen {{ http_port | default(80) }};
        server_name {{ server_name }};

{% if enable_tls %}
        listen 443 ssl;
        ssl_certificate     /etc/ssl/certs/{{ server_name }}.pem;
        ssl_certificate_key /etc/ssl/private/{{ server_name }}.key;
{% endif %}

        location / {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
        }
    }
}
```

```yaml
- name: Render nginx config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    mode: "0644"
    validate: "nginx -t -c %s"      # refuse to install a broken config!
  notify: Reload nginx
```

**Useful filters:**

```jinja
{{ mylist | join(",") }}
{{ value | default("fallback") }}
{{ password | b64encode }}
{{ users | map(attribute='name') | list }}
{{ 1073741824 | human_readable }}          {# "1.00 GB" #}
{{ hostvars | to_nice_json }}
{{ "%s-%s" | format(app_name, app_env) }}
```

Notice the `validate:` parameter and the loop over `groups['appservers']` — templates that auto-build load-balancer pools from inventory are everywhere in industry.

---

## 10. Handlers

Handlers run **once, at the end of the play, only if notified by a changed task**. Classic use: restart a service only when its config actually changed.

```yaml
- name: Web server with handlers
  hosts: webservers
  become: true

  tasks:
    - name: Deploy nginx config
      ansible.builtin.template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Reload nginx          # fires only on "changed"

    - name: Deploy vhost
      ansible.builtin.template:
        src: vhost.conf.j2
        dest: /etc/nginx/sites-enabled/app.conf
      notify: Reload nginx          # notified twice → still runs ONCE

  handlers:
    - name: Reload nginx
      ansible.builtin.service:
        name: nginx
        state: reloaded
```

Force handlers to run mid-play when ordering matters:

```yaml
- name: Apply pending handlers now
  ansible.builtin.meta: flush_handlers
```

---

## 11. Roles — Structuring Real Projects

Roles are how professionals organize Ansible. A role bundles everything needed for one capability (nginx, postgres, app, hardening) into a standard directory tree.

```bash
ansible-galaxy init roles/nginx
```

```
roles/nginx/
├── defaults/main.yml      # lowest-precedence vars (safe to override)
├── files/                 # static files for copy
├── handlers/main.yml
├── meta/main.yml          # dependencies, author info
├── tasks/main.yml         # entry point
├── templates/             # .j2 templates
└── vars/main.yml          # high-precedence vars (rarely used)
```

### Example role

```yaml
# roles/nginx/defaults/main.yml
nginx_port: 80
nginx_worker_connections: 1024
nginx_remove_default_vhost: true
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Remove default vhost
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  when: nginx_remove_default_vhost
  notify: Reload nginx

- name: Deploy config
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: "nginx -t -c %s"
  notify: Reload nginx

- name: Start and enable
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```

```yaml
# roles/nginx/handlers/main.yml
---
- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

### Using roles

```yaml
# site.yml
---
- name: Web tier
  hosts: webservers
  become: true
  roles:
    - common          # baseline: ntp, users, monitoring agent
    - nginx
    - { role: app, app_version: "2.4.1" }
```

### Standard production repo layout

```
ansible/
├── ansible.cfg
├── site.yml                  # master playbook
├── webservers.yml            # tier playbooks
├── dbservers.yml
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── webservers.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── app/
│   └── postgresql/
├── collections/requirements.yml
└── README.md
```

Run per environment:

```bash
ansible-playbook -i inventories/staging site.yml
ansible-playbook -i inventories/production site.yml --limit webservers
```

This *separate inventories, shared roles* pattern is the single most common Ansible layout in industry.

---

## 12. Ansible Vault — Secrets Management

Never commit plaintext passwords. Vault encrypts secrets so they can live safely in git.

```bash
# Create an encrypted file
ansible-vault create group_vars/production/vault.yml

# Edit / view / change password
ansible-vault edit group_vars/production/vault.yml
ansible-vault view group_vars/production/vault.yml
ansible-vault rekey group_vars/production/vault.yml

# Encrypt an existing file
ansible-vault encrypt secrets.yml

# Encrypt a single value (paste into any vars file)
ansible-vault encrypt_string 'S3cr3tP@ss' --name 'db_password'
```

Inside `vault.yml`:

```yaml
vault_db_password: "S3cr3tP@ss"
vault_api_key: "sk-abc123"
```

**Industry convention** — indirection via a plain vars file so `grep` still finds variable usage:

```yaml
# group_vars/production/vars.yml   (plaintext, in git)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"
```

Running with vault:

```bash
ansible-playbook site.yml --ask-vault-pass
# or (CI-friendly):
echo "$VAULT_PASS" > .vault_pass && chmod 600 .vault_pass
ansible-playbook site.yml --vault-password-file .vault_pass
```

In `ansible.cfg`: `vault_password_file = ~/.vault_pass` (never commit the password file — add it to `.gitignore`).

> Larger orgs often replace/augment Vault with HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault via lookup plugins:
> `{{ lookup('amazon.aws.aws_secret', 'prod/db/password') }}`

---

## 13. Ansible Galaxy & Collections

Don't reinvent wheels — the community has production-grade roles for almost everything.

```bash
# Install a famous community role
ansible-galaxy role install geerlingguy.postgresql

# Install collections
ansible-galaxy collection install amazon.aws community.general
```

Pin dependencies like a lockfile:

```yaml
# collections/requirements.yml
collections:
  - name: amazon.aws
    version: ">=9.0.0"
  - name: community.general
  - name: ansible.posix

roles:
  - name: geerlingguy.postgresql
    version: "3.5.2"
```

```bash
ansible-galaxy install -r collections/requirements.yml
```

Highly regarded community content worth studying: `geerlingguy.*` roles (nginx, docker, postgresql, mysql), `devsec.hardening` collection (OS/SSH hardening), `community.docker`, `kubernetes.core`.

---

## 14. Industry Project 1: Zero-Downtime Web App Deployment

**Scenario:** an e-commerce company deploys a Python app behind a load balancer to 4 web servers. Requirements: no downtime, deploy in batches, health-check each server, auto-rollback on failure.

Key technique: `serial` (rolling batches) + LB drain + `block/rescue`.

```yaml
# deploy.yml
---
- name: Rolling deploy of shop app
  hosts: webservers
  become: true
  serial: 2                        # 2 hosts at a time (can be "25%")
  max_fail_percentage: 25          # abort whole deploy if >25% of a batch fails

  vars:
    app_version: "{{ version | mandatory }}"   # require -e version=x.y.z
    release_dir: "/opt/shop/releases/{{ app_version }}"

  pre_tasks:
    - name: Take server out of the load balancer
      community.general.haproxy:
        state: disabled
        host: "{{ inventory_hostname }}"
        backend: shop_backend
      delegate_to: "{{ item }}"            # run ON the LB, not this host
      loop: "{{ groups['loadbalancers'] }}"

  tasks:
    - name: Deploy and verify
      block:
        - name: Create release directory
          ansible.builtin.file:
            path: "{{ release_dir }}"
            state: directory
            owner: shop
            mode: "0755"

        - name: Download release artifact
          ansible.builtin.unarchive:
            src: "https://artifacts.example.com/shop/shop-{{ app_version }}.tar.gz"
            dest: "{{ release_dir }}"
            remote_src: true

        - name: Install Python dependencies
          ansible.builtin.pip:
            requirements: "{{ release_dir }}/requirements.txt"
            virtualenv: "{{ release_dir }}/venv"

        - name: Render app config
          ansible.builtin.template:
            src: app-config.py.j2
            dest: "{{ release_dir }}/config.py"
            mode: "0640"

        - name: Record current release for rollback
          ansible.builtin.stat:
            path: /opt/shop/current
          register: current_link

        - name: Switch symlink to new release (atomic)
          ansible.builtin.file:
            src: "{{ release_dir }}"
            dest: /opt/shop/current
            state: link

        - name: Restart app
          ansible.builtin.systemd_service:
            name: shop
            state: restarted
            daemon_reload: true

        - name: Wait for health check to pass
          ansible.builtin.uri:
            url: "http://localhost:8000/health"
            status_code: 200
          register: health
          until: health.status == 200
          retries: 12
          delay: 5

      rescue:
        - name: ROLLBACK — restore previous symlink
          ansible.builtin.file:
            src: "{{ current_link.stat.lnk_target }}"
            dest: /opt/shop/current
            state: link
          when: current_link.stat.exists

        - name: Restart app on old version
          ansible.builtin.systemd_service:
            name: shop
            state: restarted

        - name: Fail this host so the deploy stops
          ansible.builtin.fail:
            msg: "Deploy of {{ app_version }} failed on {{ inventory_hostname }} — rolled back."

  post_tasks:
    - name: Put server back into the load balancer
      community.general.haproxy:
        state: enabled
        host: "{{ inventory_hostname }}"
        backend: shop_backend
      delegate_to: "{{ item }}"
      loop: "{{ groups['loadbalancers'] }}"

    - name: Keep only the 5 newest releases
      ansible.builtin.shell: |
        set -o pipefail
        ls -1dt /opt/shop/releases/* | tail -n +6 | xargs -r rm -rf
      args:
        executable: /bin/bash
      changed_when: false
```

```bash
ansible-playbook deploy.yml -e version=2.4.1
```

**Concepts practiced:** `serial` rolling updates, `delegate_to` (LB control), `block/rescue` rollback, `until/retries` health gates, atomic symlink releases, `mandatory` variables, `max_fail_percentage`.

---

## 15. Industry Project 2: AWS Cloud Provisioning with Dynamic Inventory

**Scenario:** a startup spins up an entire environment — VPC-attached EC2 instances tagged by role — then configures them, without ever hand-editing an inventory file.

### Step 1 — Provision infrastructure

```bash
ansible-galaxy collection install amazon.aws
pip install boto3
export AWS_ACCESS_KEY_ID=... AWS_SECRET_ACCESS_KEY=...
```

```yaml
# provision.yml
---
- name: Provision EC2 fleet
  hosts: localhost                 # cloud modules run FROM the control node
  connection: local
  gather_facts: false

  vars:
    region: eu-central-1
    instances:
      - { name: web-1, role: webserver, type: t3.small }
      - { name: web-2, role: webserver, type: t3.small }
      - { name: db-1,  role: dbserver,  type: t3.medium }

  tasks:
    - name: Create security group
      amazon.aws.ec2_security_group:
        name: app-sg
        description: Web + SSH
        region: "{{ region }}"
        rules:
          - { proto: tcp, ports: [22],  cidr_ip: "203.0.113.0/24" }  # office IP only
          - { proto: tcp, ports: [80, 443], cidr_ip: "0.0.0.0/0" }
      register: sg

    - name: Launch instances
      amazon.aws.ec2_instance:
        name: "{{ item.name }}"
        region: "{{ region }}"
        instance_type: "{{ item.type }}"
        image_id: ami-0abcdef1234567890        # Ubuntu 24.04 LTS in your region
        key_name: ansible-key
        security_group: "{{ sg.group_id }}"
        tags:
          role: "{{ item.role }}"
          env: staging
          managed_by: ansible
        state: running
        wait: true
      loop: "{{ instances }}"
```

### Step 2 — Dynamic inventory (hosts discovered by tag)

```yaml
# inventories/aws/aws_ec2.yml    ← filename MUST end in aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - eu-central-1
filters:
  tag:managed_by: ansible
  instance-state-name: running
keyed_groups:
  - key: tags.role            # creates groups: role_webserver, role_dbserver
    prefix: role
  - key: tags.env
    prefix: env
compose:
  ansible_host: public_ip_address
```

```bash
ansible-inventory -i inventories/aws/aws_ec2.yml --graph
# @all:
#   @role_webserver: web-1, web-2
#   @role_dbserver:  db-1
#   @env_staging:    web-1, web-2, db-1
```

### Step 3 — Configure by tag-derived group

```yaml
# configure.yml
---
- name: Configure web tier
  hosts: role_webserver
  become: true
  roles: [common, nginx, app]

- name: Configure db tier
  hosts: role_dbserver
  become: true
  roles: [common, postgresql]
```

```bash
ansible-playbook -i inventories/aws/aws_ec2.yml configure.yml
```

Scale from 3 servers to 50: launch more instances with the right tags — inventory updates itself. The same pattern exists for Azure (`azure.azcollection.azure_rm`), GCP (`google.cloud.gcp_compute`), and VMware.

> **Ansible vs Terraform:** many shops use Terraform to *create* infrastructure and Ansible to *configure* it. Ansible can do both, but knowing this division of labor is valuable in interviews.

---

## 16. Industry Project 3: Security Hardening & Patch Management

**Scenario:** compliance requires SSH hardening, a firewall baseline, automated patching with controlled reboots, and an audit trail — across every Linux server.

### Hardening playbook

```yaml
# harden.yml
---
- name: Baseline security hardening
  hosts: all
  become: true

  vars:
    allowed_ssh_users: [deploy, ansible]
    unwanted_packages: [telnet, rsh-client, xinetd]

  tasks:
    # --- SSH hardening ---
    - name: Harden sshd_config
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^#?{{ item.key }} "
        line: "{{ item.key }} {{ item.value }}"
        validate: "sshd -t -f %s"
      loop:
        - { key: PermitRootLogin,        value: "no" }
        - { key: PasswordAuthentication, value: "no" }
        - { key: MaxAuthTries,           value: "3" }
        - { key: X11Forwarding,          value: "no" }
        - { key: ClientAliveInterval,    value: "300" }
      notify: Restart sshd

    - name: Restrict SSH to allowed users
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "^AllowUsers"
        line: "AllowUsers {{ allowed_ssh_users | join(' ') }}"
        validate: "sshd -t -f %s"
      notify: Restart sshd

    # --- Firewall ---
    - name: Install ufw
      ansible.builtin.apt:
        name: ufw
        state: present

    - name: Allow SSH and web
      community.general.ufw:
        rule: allow
        port: "{{ item }}"
        proto: tcp
      loop: ["22", "80", "443"]

    - name: Enable firewall, default deny incoming
      community.general.ufw:
        state: enabled
        policy: deny
        direction: incoming

    # --- Remove insecure software ---
    - name: Purge unwanted packages
      ansible.builtin.apt:
        name: "{{ unwanted_packages }}"
        state: absent
        purge: true

    # --- Automatic security updates + fail2ban ---
    - name: Install unattended-upgrades and fail2ban
      ansible.builtin.apt:
        name: [unattended-upgrades, fail2ban]
        state: present

    - name: Ensure fail2ban running
      ansible.builtin.service:
        name: fail2ban
        state: started
        enabled: true

  handlers:
    - name: Restart sshd
      ansible.builtin.service:
        name: ssh
        state: restarted
```

> For full CIS-style baselines, use the community `devsec.hardening` collection (`os_hardening`, `ssh_hardening` roles) rather than writing hundreds of rules yourself.

### Patch management with controlled reboots

```yaml
# patch.yml
---
- name: Monthly patching window
  hosts: all
  become: true
  serial: "25%"                    # never take down more than a quarter of the fleet

  tasks:
    - name: Update all packages (Debian family)
      ansible.builtin.apt:
        upgrade: dist
        update_cache: true
        autoremove: true
      when: ansible_facts['os_family'] == "Debian"
      register: apt_result

    - name: Update all packages (RedHat family)
      ansible.builtin.dnf:
        name: "*"
        state: latest
      when: ansible_facts['os_family'] == "RedHat"

    - name: Check if reboot required
      ansible.builtin.stat:
        path: /var/run/reboot-required
      register: reboot_required

    - name: Reboot if needed (waits for host to return)
      ansible.builtin.reboot:
        msg: "Ansible patching reboot"
        reboot_timeout: 600
      when: reboot_required.stat.exists

    - name: Verify critical service after reboot
      ansible.builtin.service:
        name: nginx
        state: started
      when: "'webservers' in group_names"

    - name: Append to audit log on control node
      ansible.builtin.lineinfile:
        path: ./patch-audit.log
        line: "{{ ansible_date_time.iso8601 }} {{ inventory_hostname }} patched, reboot={{ reboot_required.stat.exists }}"
        create: true
      delegate_to: localhost
      become: false
```

**Concepts practiced:** `lineinfile` with `validate`, firewall modules, `serial` percentages, the `reboot` module (it waits and reconnects automatically), `group_names`, `delegate_to: localhost` for audit trails.

---

## 17. Industry Project 4: Database & Multi-Tier Orchestration

**Scenario:** stand up a complete 3-tier stack — PostgreSQL with an app database/user, app servers, LB — with correct ordering and cross-host data sharing.

```yaml
# site.yml
---
- name: Database tier
  hosts: dbservers
  become: true
  vars_files:
    - group_vars/production/vault.yml     # vault_db_password lives here

  tasks:
    - name: Install PostgreSQL + adapter deps
      ansible.builtin.apt:
        name: [postgresql, python3-psycopg2]
        state: present

    - name: Ensure PostgreSQL is running
      ansible.builtin.service:
        name: postgresql
        state: started
        enabled: true

    - name: Create app database
      community.postgresql.postgresql_db:
        name: shopdb
      become_user: postgres

    - name: Create app user
      community.postgresql.postgresql_user:
        db: shopdb
        name: shop_app
        password: "{{ vault_db_password }}"
      become_user: postgres

    - name: Listen on private network
      community.postgresql.postgresql_pg_hba:
        dest: /etc/postgresql/16/main/pg_hba.conf
        contype: host
        databases: shopdb
        users: shop_app
        source: "10.0.1.0/24"
        method: scram-sha-256
      notify: Restart postgresql

  handlers:
    - name: Restart postgresql
      ansible.builtin.service:
        name: postgresql
        state: restarted

- name: App tier (knows about DB via hostvars)
  hosts: appservers
  become: true
  tasks:
    - name: Render app config pointing at the DB
      ansible.builtin.template:
        src: templates/app.env.j2
        dest: /opt/shop/current/.env
        mode: "0640"
      vars:
        db_host: "{{ hostvars[groups['dbservers'][0]]['ansible_host'] }}"
      notify: Restart app

  handlers:
    - name: Restart app
      ansible.builtin.systemd_service:
        name: shop
        state: restarted

- name: Load balancer tier
  hosts: loadbalancers
  become: true
  roles:
    - haproxy      # its template loops over groups['appservers']
```

The key idea: **`hostvars` + `groups` let any play read facts and variables from any other host**, which is how tiers wire themselves together without hardcoded IPs.

---

## 18. Testing with Molecule & CI/CD Integration

### Linting — the minimum bar

```bash
pip install ansible-lint
ansible-lint site.yml       # catches deprecated syntax, missing FQCN, risky shell use…
```

### Molecule — unit tests for roles

Molecule spins up a container/VM, applies your role, runs it again to check idempotency, then verifies assertions.

```bash
pip install molecule molecule-plugins[docker]
cd roles/nginx
molecule init scenario -d docker
molecule test        # create → converge → idempotence → verify → destroy
```

```yaml
# roles/nginx/molecule/default/verify.yml
---
- name: Verify
  hosts: all
  tasks:
    - name: Nginx responds on port 80
      ansible.builtin.uri:
        url: http://localhost:80
        status_code: 200
```

### CI pipeline (GitHub Actions)

```yaml
# .github/workflows/ansible.yml
name: Ansible CI
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pipx install ansible-lint
      - run: ansible-lint

  molecule:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ansible molecule molecule-plugins[docker]
      - run: cd roles/nginx && molecule test

  deploy-staging:
    needs: [lint, molecule]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ansible
      - name: Deploy
        env:
          VAULT_PASS: ${{ secrets.VAULT_PASS }}
        run: |
          echo "$VAULT_PASS" > .vault_pass
          ansible-playbook -i inventories/staging site.yml \
            --vault-password-file .vault_pass
```

This **lint → test → deploy** pipeline is exactly how mature teams run Ansible: playbooks live in git, PRs are linted and Molecule-tested, merges to main auto-deploy to staging, production runs are gated.

---

## 19. AWX / Ansible Automation Platform

When teams grow, running playbooks from laptops stops scaling. **AWX** (open source) and **Red Hat Ansible Automation Platform** (commercial) add:

- a **web UI and REST API** for launching playbooks ("job templates")
- **RBAC** — developers can run the deploy job but can't read the vault secrets
- **scheduling** (nightly patching, drift checks) and **workflows** (chain: provision → configure → test → notify)
- centralized **logging/audit** of every run
- **surveys** — web forms that prompt for variables (e.g. pick a version to deploy)

Quick AWX install for exploration (Kubernetes via operator):

```bash
# minikube or any k8s cluster
kubectl apply -k github.com/ansible/awx-operator/config/default
```

You don't need AWX to learn Ansible — but knowing what it does matters in interviews: it's the answer to "how do you run Ansible as a team, safely, with audit trails?"

---

## 20. Best Practices & Production Checklist

**Code quality**

- Name every task. Use FQCN module names. Run `ansible-lint` in CI.
- Prefer specific modules over `command`/`shell`; when you must use them, set `changed_when`/`creates` so idempotency is preserved.
- Use `validate:` on templates/lineinfile for critical configs (sshd, nginx, sudoers).
- Keep playbooks thin; put logic in roles. One role = one responsibility.

**Safety**

- `--check --diff` before real runs on production; `--limit` to canary hosts first.
- `serial` + `max_fail_percentage` for anything fleet-wide.
- Vault (or an external secrets manager) for every secret; never plaintext in git.
- Pin collection/role versions in `requirements.yml`.
- Keep `host_key_checking = True` in production.

**Structure**

- Separate inventories per environment (staging/production) sharing the same roles.
- Variables in `group_vars`/`host_vars`, not in playbooks or inventory files.
- Tag tasks (`tags: [config, deploy]`) so you can run slices: `--tags config`.
- README in every role: what it does, its variables, an example.

**Operations**

- Everything in git. Every production run traceable (CI or AWX, not laptops).
- Facts caching (`fact_caching = jsonfile`) and `strategy: free` / higher `forks` for big fleets.
- Write playbooks so they're safe to re-run at any time — that's your drift correction and disaster recovery story.

**Common beginner mistakes to avoid**

| Mistake | Fix |
|---|---|
| Using `shell` for everything | Use the proper module (`apt`, `copy`, `user`…) |
| Secrets in plaintext vars | `ansible-vault encrypt_string` |
| One 800-line playbook | Split into roles |
| Ignoring `changed` vs `ok` | If a re-run shows `changed`, your task isn't idempotent — fix it |
| Restarting services in tasks | Use handlers |
| Hardcoded IPs in templates | Loop over `groups[...]` / `hostvars` |
| Testing in production | Molecule + a staging inventory |

---

## 21. Learning Path & Resources

**Suggested 6-week path (hands-on, ~5h/week):**

1. **Week 1** — Lab setup, inventory, ad-hoc commands, first playbook (sections 3–6).
2. **Week 2** — Variables, facts, loops, conditionals, templates, handlers (7–10).
3. **Week 3** — Refactor everything into roles; add Vault (11–12). Rebuild your lab from scratch with one `ansible-playbook site.yml` command.
4. **Week 4** — Industry Project 1 (rolling deploy) end to end, including deliberate failure + rollback.
5. **Week 5** — Industry Project 2 (cloud + dynamic inventory) and Project 3 (hardening/patching).
6. **Week 6** — ansible-lint + Molecule + a GitHub Actions pipeline; explore AWX.

**Resources**

- Official docs: https://docs.ansible.com — module index is your daily reference
- Jeff Geerling, *Ansible for DevOps* — the classic practical book; his YouTube series is free
- `devsec.hardening` and `geerlingguy.*` on Galaxy — read their source to learn role style
- Certification: **Red Hat RHCE (EX294)** is the recognized Ansible cert
- Practice ideas: automate your own homelab/dotfiles; every manual server task you ever do, write it as a playbook instead

**Muscle-memory commands**

```bash
ansible all -m ping
ansible-playbook site.yml --check --diff
ansible-playbook site.yml --limit web1 --tags config
ansible-doc ansible.builtin.template        # offline docs for any module
ansible-doc -l | grep -i postgres           # find modules
ansible-inventory --graph
ansible-lint
```

---

*Everything in this guide re-runs safely — that's the Ansible mindset: describe the state you want, let the engine converge to it, and version-control the description.*
