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

# Pentest-workstation feel: a welcome banner (motd), a host-aware `labhelp` that
# lists the REAL attack chain from the lab guide, a `netmap` topology command, and
# the themed prompt + scan/pnmap aliases — all on every interactive shell.
COPY motd /etc/motd-lab
COPY labhelp /usr/local/bin/labhelp
COPY netmap /usr/local/bin/netmap
# Install the banner+aliases+PS1 snippet for BOTH interactive entry points:
# /etc/profile.d for login shells, and /etc/bash/bashrc (Alpine's system bashrc)
# for the `docker exec -it secutils bash` home shell. The snippet is $--guarded so
# a one-off non-interactive `docker exec secutils sh -c ...` / `ssh host cmd` stays
# clean (no banner spam) — that's why it goes here and NOT in ~/.bashrc. exFAT gives
# copied files mode 0700, so set the modes explicitly.
COPY lab-bashrc /etc/profile.d/00-lab-workstation.sh
RUN mkdir -p /etc/bash \
    && chmod 644 /etc/motd-lab /etc/profile.d/00-lab-workstation.sh \
    && chmod 755 /usr/local/bin/labhelp /usr/local/bin/netmap \
    && cat /etc/profile.d/00-lab-workstation.sh >> /etc/bash/bashrc

CMD ["/bin/bash"]
