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
    proxychains-ng \
    git \
    vim

# Install impacket for SMB exploitation
RUN pip3 install --break-system-packages impacket

# Configure proxychains so tools can be tunnelled through a compromised host's
# SSH SOCKS proxy (ssh -D 1080 user@workstation) to reach the internal segment.
RUN mkdir -p /etc/proxychains && \
    printf '%s\n' \
        'strict_chain' \
        'proxy_dns' \
        'remote_dns_subnet 224' \
        'tcp_read_time_out 15000' \
        'tcp_connect_time_out 8000' \
        '[ProxyList]' \
        'socks5 127.0.0.1 1080' \
        > /etc/proxychains.conf && \
    cp /etc/proxychains.conf /etc/proxychains/proxychains.conf && \
    ln -sf "$(command -v proxychains4 2>/dev/null || echo /usr/bin/proxychains4)" /usr/local/bin/proxychains

# Create a working directory
WORKDIR /root

# Add some helpful aliases
RUN echo 'alias ll="ls -la"' >> /root/.bashrc && \
    echo 'alias scan="nmap -sn 172.20.1.0/24"' >> /root/.bashrc && \
    echo 'alias pnmap="proxychains nmap -sT -Pn"' >> /root/.bashrc && \
    echo 'PS1="\[\033[01;32m\]secutils\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ "' >> /root/.bashrc

CMD ["/bin/bash"]
