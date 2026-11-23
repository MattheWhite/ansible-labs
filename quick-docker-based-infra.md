# Define and create a Docker based infrastucture to learn and test Ansible usage and capacities
## Requirements
This lab assumes you have installed on your laptop: Python, Ansible

Using Docker containers on your own laptop is the fastest and most resource-efficient way to simulate servers for learning Ansible. Since containers start up in seconds and are disposable, you can experiment with them without any risk.

1. Preparing the Environment

• For Linux or macOS: Install Docker (Docker Engine or Docker Desktop) and Ansible itself on your machine (e.g., `sudo apt install ansible` or `brew install ansible`).
• For Windows: Install WSL2 (e.g., with an Ubuntu distribution) and Docker Desktop (ensuring the WSL2 integration feature is enabled). Install Ansible within the WSL Ubuntu terminal, as it cannot run as a control node on native Windows.

• Sudo privileges: If you are not logged in as the root user but need to perform system-level modifications (such as installing packages), add the `-b` (`--become`) flag to your commands, along with the `-K` flag if you need to be prompted for a sudo password.
• Missing Python: On minimal Linux installations, Python might not be present. In such cases, the `ping` module will throw an error; you will need to install the `python3` package on the target server.

# Steps
## 1. Create test servers in Docker
Before that, we create custom docker network for this ansible lab and the containers which will simulate the servers (managed nodes)
```docker network create ansible-net```


