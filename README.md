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
