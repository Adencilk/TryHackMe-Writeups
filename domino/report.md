# Domino - Detailed Penetration Testing Report

## 1. Introduction
## 2. Scope and Target
## 3. Reconnaissance
### 3.1 Nmap Scan
I began by scanning the target to identify exposed services.

```bash
nmap -sC -sV <TARGET_IP>
```
**Nmap Scan Results:**

![Nmap Scan Results](sreenshots/domino_nmap.png)

### 3.2 Open Ports

The scan identified the following open ports:

| PORT    |STATE | SERVICE | VERSION
|---------|------|---------|-----------------------------------------------------------------
| 22/tcp  |open  | ssh     | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
|         |      |         |
| 80/tcp  |open  | http    | Apache httpd 2.4.58 ((Ubuntu))


## 4. Enumeration

### 4.1 Web Enumeration

```bash
 curl -I http://<TARGET_IP>
```

- Web Server: Apache/2.4.58
- Operating System: Ubuntu
- Port: 80/tcp
- Service: HTTP

```bash
curl http://<TARGET_IP>
```

Identified the following active endpoints:

* **Authentication Pages:**
  * `/index.php` - Primary user authentication login portal.
  * `/forgot.php` - Account recovery/password reset form (potential for username enumeration).
* **Information & Styling Pages:**
  * `/team.php` - Public corporate directory (utilized for username harvesting).
  * `/static/style.css` - Global cascading stylesheet for portal aesthetics.

  ### 4.2 Service Enumeration

Running:
```bash
curl http://<TARGET_IP>/team.php
```

Exposed emails which helped us get their usernames also based on their naming convention of firstname.lastname.


## 5. Vulnerability Analysis
## 6. Exploitation
## 7. Initial Access
## 8. Privilege Escalation
## 9. Proof/Flags
## 10. Attack Chain Summary
## 11. Lessons Learned
## 12. Tools Used
