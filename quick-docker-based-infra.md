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

## 1. Creating and copying an SSH key pair
To connect servers (clients/managed nodes) to the Ansible server (control node), you need to set up SSH key-based authentication and an inventory file. Since Ansible is agentless, there is no need to install additional software on the target servers; *only SSH access and Python are required*.

On the Ansible server, create an SSH key pair (if you do not already have one), then copy the public key to the servers you wish to connect so that you can log in without a password.

1. Preparing the Working Directory and SSH Key - Generating a key on the Ansible server:
```ssh-keygen -t ed25519```
    Or more explicit:
```bash
mkdir -p ~/ansible-lab && cd ~/ansible-lab

# If you already have a key (~/.ssh/id_ed25519.pub or id_rsa.pub), skip this line:
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
```
Use this code with caution.
(Press Enter for every prompt to ensure the key does not have a passphrase).

2. Copying the key ***(enter 'yes' for both, followed by the password: root)***
If you do not yet have an SSH key on this Ubuntu system, generate one and then copy it to the two containers:
```bash
ssh-copy-id -p 2221 root@127.0.0.1
ssh-copy-id -p 2222 root@127.0.0.1
```

If you succeeded, you have to see:
```
Number of key(s) added: 1

Now try logging into the machine, with:   "ssh -p 2221 'root@127.0.0.1'"
and check to make sure that only the key(s) you wanted were added.
```


## 2. Create test servers in Docker
Before that, we create custom docker network for this ansible lab and the containers which will simulate the servers (managed nodes)
```docker network create ansible-net```

Launch 2 test "servers" (from a pre-prepared Ubuntu + SSH image):bash
```bash
docker run -d --name server1 --network ansible-net -p 2221:22 rastasheep/ubuntu-sshd:18.04
docker run -d --name server2 --network ansible-net -p 2222:22 rastasheep/ubuntu-sshd:18.04
```
**(The default password for the root user for this image is: root).**

Check run
```docker ps -f network=ansible-net```


## 3. Create the inventory.ini file:
```ini
[test_servers]
server1 ansible_host=127.0.0.1 ansible_port=2221 ansible_user=root
server2 ansible_host=127.0.0.1 ansible_port=2222 ansible_user=root
```


## 4. Test the connection:bash
```ansible all -i inventory.ini -m ping```

