FROM alpine:3.19

# Install security and networking tools
RUN apk update && apk add --no-cache \
    bash \
    nmap \
    nmap-scripts \
    openssh-client \
    curl \
    wget \
    netcat-openbsd \
    bind-tools \
    python3 \
    py3-pip \
    samba-client \
    mysql-client \
    busybox-extras \
    hydra \
    john \
    nikto \
    tcpdump \
    net-tools \
    iproute2 \
    iptables \
    git \
    vim \
    telnet

# Install impacket for SMB exploitation
RUN pip3 install --break-system-packages impacket

# Create a working directory
WORKDIR /root

# Add some helpful aliases
RUN echo 'alias ll="ls -la"' >> /root/.bashrc && \
    echo 'alias scan="nmap -sP 172.20.0.0/24"' >> /root/.bashrc && \
    echo 'PS1="\[\033[01;32m\]secutils\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ "' >> /root/.bashrc

CMD ["/bin/bash"]
