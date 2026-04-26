# Ansible Docker Lab — Step-by-Step Guide

A hands-on lab that provisions a local Ansible control node and two managed nodes (a web server and a database server), all running as Docker containers on your machine.

---

## Prerequisites

- A Linux host (Ubuntu recommended)
- Internet access to pull packages and images
- A terminal with `sudo` access

---

## Step 1 — Install Docker and Docker Compose

```bash
apt-get install docker.io docker-compose -y
```

This installs the Docker engine and the Compose plugin needed to orchestrate multiple containers.

---

## Step 2 — Create the Project Directory Structure

```bash
mkdir ~/ansible-docker-lab && cd ~/ansible-docker-lab
mkdir control managed
```

Your project root is `~/ansible-docker-lab`. Inside it you will have:

```
ansible-docker-lab/
├── control/      ← Dockerfile for the Ansible control node
├── managed/      ← Dockerfile for the managed nodes (node1, node2)
├── playbooks/    ← Ansible inventory and playbooks (created later)
├── ansible_key   ← Private SSH key (generated later)
└── ansible_key.pub
```

---

## Step 3 — Create the Managed Node Dockerfile

```bash
cd managed
nano Dockerfile
```

Paste the following content:

```dockerfile
FROM ubuntu:22.04

# Avoid interactive prompts during install
ENV DEBIAN_FRONTEND=noninteractive

# Install SSH server, Python, and sudo
RUN apt-get update && apt-get install -y \
    openssh-server \
    python3 \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# Create an 'ansible' user with sudo access, no password needed
RUN useradd -m -s /bin/bash ansible && \
    echo "ansible ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers && \
    mkdir -p /home/ansible/.ssh && \
    chmod 700 /home/ansible/.ssh && \
    chown -R ansible:ansible /home/ansible/.ssh

# Prepare SSH daemon
RUN mkdir /var/run/sshd

EXPOSE 22

# Start SSH in foreground so the container stays alive
CMD ["/usr/sbin/sshd", "-D"]
```

This image will be used for **both** `node1` and `node2`. It runs an SSH daemon so Ansible can connect to it, and sets up a passwordless-sudo `ansible` user.

---

## Step 4 — Create the Control Node Dockerfile

```bash
cd ..
cd control
nano Dockerfile
```

Paste the following content:

```dockerfile
FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# Install Ansible, SSH client, and some useful extras
RUN apt-get update && apt-get install -y \
    software-properties-common \
    openssh-client \
    sshpass \
    nano \
    && add-apt-repository --yes --update ppa:ansible/ansible \
    && apt-get install -y ansible \
    && rm -rf /var/lib/apt/lists/*

# Create a workspace
WORKDIR /ansible

# Keep the container running so we can exec into it
CMD ["sleep", "infinity"]
```

This image installs Ansible from its official PPA and stays alive with `sleep infinity` so you can `exec` into it interactively.

---

## Step 5 — Generate the SSH Key Pair

Return to the project root and generate a key pair that Ansible will use to authenticate against the managed nodes:

```bash
cd ~/ansible-docker-lab
ssh-keygen -t ed25519 -f ./ansible_key -N ""
```

This creates two files:
- `ansible_key` — the private key (mounted into the control node)
- `ansible_key.pub` — the public key (mounted into each managed node's `authorized_keys`)

---

## Step 6 — Create the Docker Compose File

In the project root (`~/ansible-docker-lab`), create `docker-compose.yml`:

```bash
nano docker-compose.yml
```

Paste the following content:

```yaml
version: "3.8"

services:
  control:
    build: ./control
    container_name: control
    hostname: control
    networks:
      - ansible-net
    volumes:
      - ./playbooks:/ansible
      - ./ansible_key:/root/.ssh/id_ed25519:ro
    # Fix permissions on the mounted key, then sleep
    command: >
      bash -c "chmod 600 /root/.ssh/id_ed25519 &&
               mkdir -p /root/.ssh &&
               ssh-keyscan -H node1 node2 >> /root/.ssh/known_hosts 2>/dev/null;
               sleep infinity"
    depends_on:
      - node1
      - node2

  node1:
    build: ./managed
    container_name: node1
    hostname: node1
    networks:
      - ansible-net
    volumes:
      - ./ansible_key.pub:/home/ansible/.ssh/authorized_keys:ro

  node2:
    build: ./managed
    container_name: node2
    hostname: node2
    networks:
      - ansible-net
    volumes:
      - ./ansible_key.pub:/home/ansible/.ssh/authorized_keys:ro

networks:
  ansible-net:
    driver: bridge
```

Key points:
- All three containers share the `ansible-net` bridge network, so they can reach each other by hostname.
- The control node auto-runs `ssh-keyscan` on startup to pre-populate `known_hosts`, avoiding host-key prompts.
- `depends_on` ensures `node1` and `node2` start before the control node.

---

## Step 7 — Create the Ansible Inventory

```bash
mkdir playbooks
cd playbooks
nano inventory.ini
```

Paste the following content:

```ini
[webservers]
node1

[databases]
node2

[all:vars]
ansible_user=ansible
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

- `node1` is grouped under `webservers` (will get Nginx).
- `node2` is grouped under `databases` (will get PostgreSQL).
- The `ansible_user` matches the user created inside the managed node image.

---

## Step 8 — Build and Start the Lab

From the project root:

```bash
cd ~/ansible-docker-lab
docker-compose up -d --build
```

`--build` forces Docker to build all images from scratch. `-d` runs everything in the background.

Verify all three containers are running:

```bash
docker ps
```

You should see `control`, `node1`, and `node2` all with a status of `Up`.

---

## Step 9 — Test Connectivity with Ansible Ping

Enter the control node container:

```bash
docker exec -it control bash
```

Inside the container, run an ad-hoc ping against all hosts:

```bash
cd /ansible
ansible all -i inventory.ini -m ping
```

A successful response looks like:

```
node1 | SUCCESS => { "ping": "pong" }
node2 | SUCCESS => { "ping": "pong" }
```

Exit the container when done:

```bash
exit
```

---

## Step 10 — Write the Provisioning Playbook

Back on the host, inside the `playbooks` directory:

```bash
cd playbooks
nano setup.yml
```

Paste the following content:

```yaml
---
- name: Configure webservers
  hosts: webservers
  become: yes
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Create a custom homepage
      copy:
        content: "Hello from {{ inventory_hostname }} managed by Ansible!"
        dest: /var/www/html/index.html

- name: Configure databases
  hosts: databases
  become: yes
  tasks:
    - name: Install PostgreSQL
      apt:
        name: postgresql
        state: present
        update_cache: yes
```

This playbook has two plays:
1. **Webservers play** — installs Nginx on `node1` and drops a custom `index.html`.
2. **Databases play** — installs PostgreSQL on `node2`.

---

## Step 11 — Run the Playbook

Enter the control node again:

```bash
docker exec -it control bash
```

Execute the playbook:

```bash
ansible-playbook -i inventory.ini setup.yml
```

Ansible will connect to each node over SSH, escalate privileges with `sudo`, and apply the tasks. You will see a play recap at the end showing `changed` and `ok` counts for each host.

---

## Summary

| Container | Role | Software Installed |
|-----------|------|--------------------|
| `control` | Ansible control node | Ansible, SSH client |
| `node1` | Managed — webserver | Nginx + custom homepage |
| `node2` | Managed — database | PostgreSQL |

You now have a fully functional local Ansible lab running entirely in Docker, with no cloud resources or physical machines required.