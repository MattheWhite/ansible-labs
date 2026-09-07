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
