# Lateral Movement Lab

<!-- BADGES:START -->
[![cybersecurity](https://img.shields.io/badge/-cybersecurity-f44336?style=flat-square)](https://github.com/topics/cybersecurity) [![docker](https://img.shields.io/badge/-docker-2496ed?style=flat-square)](https://github.com/topics/docker) [![docker-compose](https://img.shields.io/badge/-docker--compose-blue?style=flat-square)](https://github.com/topics/docker-compose) [![edtech](https://img.shields.io/badge/-edtech-4caf50?style=flat-square)](https://github.com/topics/edtech) [![ethical-hacking](https://img.shields.io/badge/-ethical--hacking-blue?style=flat-square)](https://github.com/topics/ethical-hacking) [![html](https://img.shields.io/badge/-html-e34f26?style=flat-square)](https://github.com/topics/html) [![lateral-movement](https://img.shields.io/badge/-lateral--movement-blue?style=flat-square)](https://github.com/topics/lateral-movement) [![penetration-testing](https://img.shields.io/badge/-penetration--testing-blue?style=flat-square)](https://github.com/topics/penetration-testing) [![security-lab](https://img.shields.io/badge/-security--lab-blue?style=flat-square)](https://github.com/topics/security-lab) [![vulnerable-applications](https://img.shields.io/badge/-vulnerable--applications-blue?style=flat-square)](https://github.com/topics/vulnerable-applications)
<!-- BADGES:END -->

> **Part of the [Assume-Breach series](https://michael-borck.github.io/security-labs/)** — five hands-on security labs, two companion books, and a game. Browse them all at the [series hub](https://github.com/michael-borck/security-labs).

**▶ Start here: https://michael-borck.github.io/cybersecurity-lab-lateral-movement/** — or run `./start.sh`.

A hands-on **pivoting** lab in Docker. You start on an attacker box that can only
see one network segment. To reach the sensitive services, you have to break into
a host, discover that it is dual-homed, and **tunnel through it** to the segment
you were never meant to touch. That is lateral movement — and here it is real,
not simulated on a flat network.

## The scenario

You have been dropped onto the **corp** segment with an attack box. Somewhere on
the internal segment sits a directory server, a database, and a legacy telnet
service — but you cannot route to any of them. Your only way in is a corporate
**workstation** that happens to sit on both networks. Compromise it, pivot
through it, and loot what lies beyond.

## System Requirements

This lab is designed to run on low-powered laptops across Linux, Windows, and macOS.

### Minimum Requirements
- 4GB RAM (8GB recommended)
- 10GB free disk space
- Docker Desktop or Docker Engine installed
- Docker Compose v2.0 or higher

### Tested On
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

### 2. Get the Lab

The lab images are pre-built and published to GitHub Container Registry (GHCR),
so you don't need to build anything — just grab the repo:

```bash
git clone https://github.com/michael-borck/cybersecurity-lab-lateral-movement.git
cd cybersecurity-lab-lateral-movement
```

### 3. Start the Lab

The friendly way — a single command that boots the network and logs you straight
into a real shell on the attacker box (no Docker commands to memorise):

```bash
./start.sh
```

On macOS you can double-click `start.command`; on Windows, `start.bat`.

Prefer raw Docker? That works too:

```bash
docker compose up -d
```

The first run pulls the pre-built images from GHCR (a few minutes depending on
your connection); subsequent startups are near-instant. The images are published
as multi-architecture, so the correct build is pulled automatically on Intel/AMD
machines and on Apple Silicon Macs.

> **Developers:** to build the custom images locally instead of pulling them,
> layer the build override file on top of the base compose:
>
> ```bash
> docker compose -f docker-compose.yml -f docker-compose.build.yml up -d --build
> ```
>
> Pushes to `main` rebuild and republish the images automatically via the
> `.github/workflows/build.yml` GitHub Actions workflow.

### 4. Verify Everything Is Running

```bash
docker compose ps
```

You should see 8 containers running: secutils, workstation, ubuntu-desktop,
ldap, mysql, telnet, juice-shop, dvwa.

## Lab Architecture

This is a genuine **two-segment** network. That split is the whole point: the
attacker cannot reach the sensitive services without pivoting.

### Networks

| Network | Subnet | Who lives here |
|---------|--------|----------------|
| **corp** | 172.20.1.0/24 | secutils (attacker), workstation, juice-shop, dvwa |
| **internal** | 172.20.2.0/24 | workstation (2nd NIC), ubuntu-desktop, ldap, mysql, telnet |

The **workstation is dual-homed** — it sits on *both* segments (172.20.1.20 on
corp, 172.20.2.20 on internal). That makes it the **pivot**. `secutils` is on
**corp only**; it *cannot* talk to ldap, mysql, telnet, or ubuntu-desktop until
you compromise the workstation and tunnel through it.

### Hosts and Services

| Host | corp IP | internal IP | Services | Role |
|------|---------|-------------|----------|------|
| secutils | 172.20.1.10 | — | nmap, hydra, john, impacket, proxychains, clients | Attacker box |
| workstation | 172.20.1.20 | 172.20.2.20 | SSH (22), SMB (139/445) | Foothold **and pivot** |
| ubuntu-desktop | — | 172.20.2.30 | SSH (22) | Second workstation |
| ldap | — | 172.20.2.40 | LDAP (389) | Directory service |
| mysql | — | 172.20.2.50 | MySQL (3306) | Database |
| telnet | — | 172.20.2.60 | Telnet (23) | Legacy service |
| juice-shop | 172.20.1.70 | — | HTTP (3000) | Public web app |
| dvwa | 172.20.1.80 | — | HTTP (80 → localhost:8080) | Public web app |

### Default Credentials

These are intentionally weak — discovering and reusing them *is* the lab.

**Workstation (SSH):**
- labuser / Password123
- admin / Admin123!
- root / RootPass123

**Ubuntu-Desktop (SSH):**
- user1 / User123
- developer / Dev2023!

**Telnet:**
- telnetuser / Telnet123
- guest / guest

**MySQL:**
- root / root123
- dbuser / dbpass123

**LDAP** (organisation: Northwind, domain: corp.local):
- Admin DN: cn=admin,dc=corp,dc=local
- Password: admin123

## Quick Start Guide

`./start.sh` hides Docker: it boots the network and drops you straight into a
real interactive shell on the attacker box (secutils). A welcome banner greets
you — from that shell:

- `labhelp` — the attack chain, straight from the lab guide (scan → foothold → pivot → loot)
- `netmap` — redraw the two-segment network map
- `scan` / `pnmap` — handy aliases for the corp sweep and a proxychained nmap
- `exit` — leave the shell (you'll be asked whether to shut the lab down)

Browser targets are published to your host: Juice Shop at `http://localhost:3000`
and DVWA at `http://localhost:8080` (or run `./start.sh open juice` / `./start.sh
open dvwa`).

Prefer to drive Docker yourself? Get onto the attacker box with:

```bash
docker exec -it secutils /bin/bash
```

From there, a taste of the flow:

```bash
# 1. Scan the reachable corp segment
nmap -sn 172.20.1.0/24
nmap -sV 172.20.1.20

# 2. Get a foothold on the workstation (weak SSH creds), then...
# 3. ...open a SOCKS tunnel through it and reach the internal segment
ssh -D 1080 labuser@172.20.1.20
proxychains nmap -sT -Pn 172.20.2.0/24
proxychains mysql -h 172.20.2.50 -u root -proot123
```

**Web applications (published to your host):**
- Juice Shop: http://localhost:3000
- DVWA: http://localhost:8080 (first visit `/setup.php` and click Create / Reset Database)

## Lab Guide

Work through **[LAB-GUIDE.md](LAB-GUIDE.md)** for the full walkthrough — a five-phase
pivot: scan corp, get a foothold, discover the pivot, tunnel through it, and
compromise the internal services.

## Troubleshooting

### Containers won't start
```bash
docker compose down        # stop everything
docker compose down -v      # remove containers and networks
docker compose up -d        # start fresh
```

### Out of memory errors
```bash
# Docker Desktop: increase memory in Settings > Resources (4GB min, 6-8GB ideal).
# Or stop the heavier web apps if you're not using them:
docker compose stop juice-shop dvwa
```

### Can't connect between segments
Remember: that's by design for the internal segment. `secutils` reaching ldap,
mysql, telnet, or ubuntu-desktop directly should **fail** — you have to pivot
through the workstation. If corp-segment hosts can't see each other:

```bash
docker network ls | grep -E 'corp|internal'
docker network inspect cybersecurity-lab-lateral-movement_corp
docker network inspect cybersecurity-lab-lateral-movement_internal
```

### Permission denied errors
```bash
# Linux users: ensure you're in the docker group
sudo usermod -aG docker $USER
newgrp docker
```

## Stopping the Lab

```bash
docker compose stop    # stop containers (preserves data)
docker compose down    # stop and remove containers
docker compose down -v && docker system prune -a   # complete cleanup
```

## Security Notes

**IMPORTANT:** This lab contains intentionally vulnerable systems and should ONLY
be used for educational purposes in an isolated environment.

- Do NOT expose these containers to the internet
- Do NOT use these configurations in production
- All passwords are intentionally weak for educational purposes
- Practice ethical hacking principles at all times

## License

MIT — see [LICENSE](LICENSE). Unit-agnostic teaching material; part of the
Assume-Breach series of hands-on security labs.
