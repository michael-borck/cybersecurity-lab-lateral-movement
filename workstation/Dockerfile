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

# Plant loot that reveals the internal segment and the creds to reach it.
# This box is dual-homed (172.20.1.20 on corp, 172.20.2.20 on internal), so once
# an attacker lands here they can pivot to the 172.20.2.0/24 services.
RUN printf '%s\n' \
    'ssh developer@172.20.2.30' \
    'mysql -h 172.20.2.50 -u root -proot123' \
    'ldapsearch -x -H ldap://172.20.2.40 -D "cn=admin,dc=corp,dc=local" -w admin123' \
    'telnet 172.20.2.60' \
    > /home/labuser/.bash_history && \
    chown labuser:labuser /home/labuser/.bash_history

RUN printf '%s\n' \
    'Internal services live on 172.20.2.0/24 and are only reachable from this box.' \
    '' \
    '  ldap     172.20.2.40   cn=admin,dc=corp,dc=local / admin123' \
    '  mysql    172.20.2.50   root / root123' \
    '  telnet   172.20.2.60   telnetuser / Telnet123' \
    '  dev-box  172.20.2.30   developer / Dev2023!' \
    > /home/labuser/internal-hosts.txt && \
    chown labuser:labuser /home/labuser/internal-hosts.txt

# Create startup script
RUN echo '#!/bin/bash\n\
service ssh start\n\
service smbd start\n\
tail -f /dev/null' > /start.sh && \
chmod +x /start.sh

EXPOSE 22 139 445

CMD ["/start.sh"]
