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