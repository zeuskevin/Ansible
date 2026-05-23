FROM ubuntu:22.04

# Avoid interactive prompts during build
ENV DEBIAN_FRONTEND=noninteractive

# Install SSH, sudo, systemd dependencies, and build essentials for Redis compilation
RUN apt-get update && apt-get install -y \
    openssh-server \
    sudo \
    curl \
    build-essential \
    tcl \
    iproute2 \
    procps \
    && rm -rf /var/lib/apt/lists/*

# Configure SSH Server
RUN mkdir /var/run/sshd
RUN echo 'root:root' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config

# Set up SSH directory for the root user
RUN mkdir -p /root/.ssh && chmod 700 /root/.ssh

# Copy your local public key into the container to enable passwordless SSH
# Ensure you have generated a key pair locally via `ssh-keygen -t rsa -b 4000`
COPY id_rsa.pub /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys

EXPOSE 22 6379 16379

CMD ["/usr/sbin/sshd", "-D"]
