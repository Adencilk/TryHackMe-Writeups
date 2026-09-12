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

![Nmap Scan Results](sreenshots/domino_nmap.png]

The scan identified the following open ports:

| PORT    |STATE | SERVICE | VERSION
|---------|------|---------|-----------------------------------------------------------------
| 22/tcp  |open  | ssh     | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
|---------|------|---------|-----------------------------------------------------------------
| 80/tcp  |open  | http    | Apache httpd 2.4.58 ((Ubuntu))
|---------|------|---------|---------------------------------------------------------------

### 3.2 Open Ports
## 4. Enumeration
### 4.1 Service Enumeration
### 4.2 Web Enumeration
## 5. Vulnerability Analysis
## 6. Exploitation
## 7. Initial Access
## 8. Privilege Escalation
## 9. Proof/Flags
## 10. Attack Chain Summary
## 11. Lessons Learned
## 12. Tools Used
