FROM debian:bullseye-slim

# Install basic services and tools
RUN apt-get update && apt-get install -y \
    openssh-server \
    samba \
    vim \
    nano \
    curl \
    wget \
    net-tools \
    iproute2 \
    procps \
    sudo \
    && rm -rf /var/lib/apt/lists/*

# Create users for the lab
RUN useradd -m -s /bin/bash labuser && \
    echo 'labuser:Password123' | chpasswd && \
    useradd -m -s /bin/bash admin && \
    echo 'admin:Admin123!' | chpasswd && \
    usermod -aG sudo admin

# Configure SSH
RUN mkdir /var/run/sshd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    echo 'root:RootPass123' | chpasswd

# Generate SSH keys for labuser (intentionally weak for the lab)
RUN mkdir -p /home/labuser/.ssh && \
    ssh-keygen -t rsa -b 2048 -f /home/labuser/.ssh/id_rsa -N "" && \
    chown -R labuser:labuser /home/labuser/.ssh

# Create some files with weak permissions for discovery
RUN echo "DB_PASSWORD=SuperSecret123" > /home/labuser/config.txt && \
    chmod 644 /home/labuser/config.txt && \
    chown labuser:labuser /home/labuser/config.txt

# Configure Samba share
RUN mkdir -p /shared && \
    echo "Shared file with sensitive info" > /shared/notes.txt && \
    echo "API_KEY=abc123xyz789" > /shared/credentials.txt && \
    chmod 777 /shared

# Add Samba configuration
RUN echo "[shared]\n\
    path = /shared\n\
    browseable = yes\n\
    read only = no\n\
    guest ok = yes\n\
    create mask = 0777\n\
    directory mask = 0777" >> /etc/samba/smb.conf

# Create startup script
RUN echo '#!/bin/bash\n\
service ssh start\n\
service smbd start\n\
tail -f /dev/null' > /start.sh && \
chmod +x /start.sh

EXPOSE 22 139 445

CMD ["/start.sh"]
