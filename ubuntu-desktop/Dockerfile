FROM ubuntu:22.04

# Set environment to non-interactive
ENV DEBIAN_FRONTEND=noninteractive

# Install basic system and some GUI-like tools (keeping it lightweight)
RUN apt-get update && apt-get install -y \
    bash \
    vim \
    nano \
    curl \
    wget \
    net-tools \
    iproute2 \
    openssh-server \
    sudo \
    procps \
    && rm -rf /var/lib/apt/lists/*

# Create users
RUN useradd -m -s /bin/bash user1 && \
    echo 'user1:User123' | chpasswd && \
    useradd -m -s /bin/bash developer && \
    echo 'developer:Dev2023!' | chpasswd && \
    usermod -aG sudo developer

# Configure SSH
RUN mkdir /var/run/sshd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# Create vulnerable configuration files with stored credentials
RUN mkdir -p /home/developer/.config/app && \
    echo "[database]\nhost=172.20.0.50\nusername=dbuser\npassword=dbpass123" > /home/developer/.config/app/config.ini && \
    chown -R developer:developer /home/developer/.config && \
    chmod 644 /home/developer/.config/app/config.ini

# Create a credentials file in home directory
RUN echo "=== Application Credentials ===\nLDAP Admin: cn=admin,dc=cyb204,dc=local / admin123\nMySQL: root / root123" > /home/developer/credentials.txt && \
    chown developer:developer /home/developer/credentials.txt && \
    chmod 644 /home/developer/credentials.txt

# Add some history for realistic enumeration
RUN mkdir -p /home/developer && \
    echo "ssh admin@172.20.0.20" > /home/developer/.bash_history && \
    echo "mysql -h 172.20.0.50 -u root -p" >> /home/developer/.bash_history && \
    echo "cat /etc/shadow" >> /home/developer/.bash_history && \
    chown developer:developer /home/developer/.bash_history

# Create startup script
RUN echo '#!/bin/bash\nservice ssh start\ntail -f /dev/null' > /start.sh && \
    chmod +x /start.sh

EXPOSE 22

CMD ["/start.sh"]
