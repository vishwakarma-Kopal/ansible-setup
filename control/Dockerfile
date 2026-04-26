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