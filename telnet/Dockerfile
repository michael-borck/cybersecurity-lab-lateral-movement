FROM debian:bullseye-slim

# Install telnet server and basic tools
RUN apt-get update && apt-get install -y \
    telnetd \
    xinetd \
    vim \
    nano \
    net-tools \
    && rm -rf /var/lib/apt/lists/*

# Create users
RUN useradd -m -s /bin/bash telnetuser && \
    echo 'telnetuser:Telnet123' | chpasswd && \
    useradd -m -s /bin/bash guest && \
    echo 'guest:guest' | chpasswd

# Configure xinetd for telnet
RUN echo "service telnet\n\
{\n\
    disable = no\n\
    flags = REUSE\n\
    socket_type = stream\n\
    wait = no\n\
    user = root\n\
    server = /usr/sbin/in.telnetd\n\
    log_on_failure += USERID\n\
}" > /etc/xinetd.d/telnet

# Create startup script
RUN echo '#!/bin/bash\nxinetd -stayalive\n' > /start.sh && \
    chmod +x /start.sh

EXPOSE 23

CMD ["/start.sh"]
