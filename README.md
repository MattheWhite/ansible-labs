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
┌─────────────────┐        SSH / WinRM        ┌──────────────┐
│  Control Node    │ ───────────────────────▶ │ Managed Node │ web1
│  (your laptop /  │ ───────────────────────▶ │ Managed Node │ web2
│   CI runner /    │ ───────────────────────▶ │ Managed Node │ db1
│   AWX server)    │                          └──────────────┘
│                  │
│  - ansible core  │   Ansible copies small Python "module"
│  - inventory     │   scripts to each node, runs them,
│  - playbooks     │   collects JSON results, deletes them.
└─────────────────┘
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
