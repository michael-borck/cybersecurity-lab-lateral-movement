# CYB204 Lateral Movement Lab Environment

<!-- BADGES:START -->
[![cybersecurity](https://img.shields.io/badge/-cybersecurity-f44336?style=flat-square)](https://github.com/topics/cybersecurity) [![docker](https://img.shields.io/badge/-docker-2496ed?style=flat-square)](https://github.com/topics/docker) [![docker-compose](https://img.shields.io/badge/-docker--compose-blue?style=flat-square)](https://github.com/topics/docker-compose) [![edtech](https://img.shields.io/badge/-edtech-4caf50?style=flat-square)](https://github.com/topics/edtech) [![ethical-hacking](https://img.shields.io/badge/-ethical--hacking-blue?style=flat-square)](https://github.com/topics/ethical-hacking) [![html](https://img.shields.io/badge/-html-e34f26?style=flat-square)](https://github.com/topics/html) [![lateral-movement](https://img.shields.io/badge/-lateral--movement-blue?style=flat-square)](https://github.com/topics/lateral-movement) [![penetration-testing](https://img.shields.io/badge/-penetration--testing-blue?style=flat-square)](https://github.com/topics/penetration-testing) [![security-lab](https://img.shields.io/badge/-security--lab-blue?style=flat-square)](https://github.com/topics/security-lab) [![vulnerable-applications](https://img.shields.io/badge/-vulnerable--applications-blue?style=flat-square)](https://github.com/topics/vulnerable-applications)
<!-- BADGES:END -->

This Docker-based lab environment provides a safe, isolated network for practicing lateral movement techniques as part of the CYB204 Ethical Hacking course.

## System Requirements

This lab has been designed to run on low-powered laptops across Linux, Windows, and Mac platforms.

### Minimum Requirements:
- 4GB RAM (8GB recommended)
- 10GB free disk space
- Docker Desktop or Docker Engine installed
- Docker Compose v2.0 or higher

### Tested On:
- Windows 10/11 with Docker Desktop
- macOS (Intel and Apple Silicon)
- Ubuntu Linux 20.04+

## Installation

### 1. Install Docker

**Windows/Mac:**
- Download and install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Ensure Docker Desktop is running before proceeding

**Linux:**
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Install Docker Compose
sudo apt-get install docker-compose-plugin

# Add your user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Clone or Download Lab Files

If you received these files as a zip, extract them. Otherwise:
```bash
cd ~/Projects
# Files should be in the 'movement' directory
```

### 3. Start the Lab Environment

```bash
cd movement
docker-compose up -d
```

First-time startup will take 5-10 minutes as Docker downloads and builds the images. Subsequent startups will be much faster.

### 4. Verify All Containers Are Running

```bash
docker-compose ps
```

You should see 8 containers running:
- secutils
- workstation
- ubuntu-desktop
- ldap
- mysql
- telnet
- juice-shop
- dvwa

## Lab Architecture

### Network Configuration
- **Subnet:** 172.20.0.0/24
- **Network Name:** cyb204_net

### Container IPs and Services

| Container | IP Address | Services | Purpose |
|-----------|------------|----------|---------|
| secutils | 172.20.0.10 | Attack tools | Primary attack client with nmap, impacket, etc. |
| workstation | 172.20.0.20 | SSH (22), SMB (139/445) | Linux target with shared resources |
| ubuntu-desktop | 172.20.0.30 | SSH (22) | Ubuntu target with config files |
| ldap | 172.20.0.40 | LDAP (389) | Directory service target |
| mysql | 172.20.0.50 | MySQL (3306) | Database target |
| telnet | 172.20.0.60 | Telnet (23) | Telnet service target |
| juice-shop | 172.20.0.70 | HTTP (3000) | Vulnerable web application |
| dvwa | 172.20.0.80 | HTTP (80) | Damn Vulnerable Web App |

### Default Credentials

**Workstation:**
- labuser / Password123
- admin / Admin123!
- root / RootPass123

**Ubuntu-Desktop:**
- user1 / User123
- developer / Dev2023!

**Telnet:**
- telnetuser / Telnet123
- guest / guest

**MySQL:**
- root / root123
- dbuser / dbpass123

**LDAP:**
- Admin DN: cn=admin,dc=cyb204,dc=local
- Password: admin123

## Quick Start Guide

### Access the Attack Container (secutils)

```bash
docker exec -it secutils /bin/bash
```

### Basic Commands

**Scan the network:**
```bash
nmap -sP 172.20.0.0/24
```

**Detailed service scan:**
```bash
nmap -sV 172.20.0.20
```

**Check container IPs:**
```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_name>
```

**Access other containers directly:**
```bash
docker exec -it workstation /bin/sh
docker exec -it ubuntu-desktop /bin/bash
```

**View web applications:**
- Juice Shop: http://localhost:3000
- DVWA: http://localhost:8080

## Lab Exercises

Follow the exercises in `week09-worksheet-movement.docx` to:
1. Enumerate the network
2. Dump credentials from Linux systems
3. Exploit trust relationships via SSH and SMB
4. Practice remote code execution
5. Understand lateral movement techniques

## Troubleshooting

### Containers won't start
```bash
# Stop all containers
docker-compose down

# Remove old containers and networks
docker-compose down -v

# Rebuild and start fresh
docker-compose build --no-cache
docker-compose up -d
```

### Out of memory errors
```bash
# On Docker Desktop: Increase memory allocation in Settings > Resources
# Minimum 4GB recommended, 6-8GB ideal for all containers

# Stop unnecessary containers
docker-compose stop juice-shop dvwa
```

### Can't connect between containers
```bash
# Verify network exists
docker network ls | grep cyb204

# Check container IPs
docker network inspect cyb204_net
```

### Permission denied errors
```bash
# Linux users: ensure you're in docker group
sudo usermod -aG docker $USER
newgrp docker
```

## Stopping the Lab

### Stop all containers (preserves data):
```bash
docker-compose stop
```

### Stop and remove containers:
```bash
docker-compose down
```

### Complete cleanup (removes everything):
```bash
docker-compose down -v
docker system prune -a
```

## Security Notes

**IMPORTANT:** This lab environment contains intentionally vulnerable systems and should ONLY be used for educational purposes in an isolated environment.

- Do NOT expose these containers to the internet
- Do NOT use these configurations in production
- All passwords are intentionally weak for educational purposes
- Practice ethical hacking principles at all times

## Resource Usage

Typical resource usage with all containers running:
- **Memory:** 2-4GB RAM
- **CPU:** 10-20% on modern systems
- **Disk:** ~2GB for images

To reduce resource usage, stop unused containers:
```bash
docker-compose stop juice-shop dvwa
```

## Support

If you encounter issues:
1. Check the troubleshooting section above
2. Review Docker logs: `docker-compose logs <container_name>`
3. Verify Docker Desktop is running and allocated sufficient resources
4. Contact your instructor with error messages and system specifications

## License

This lab environment is created for educational purposes for CYB204 Ethical Hacking course.
