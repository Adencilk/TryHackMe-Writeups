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

Exposed emails which helped us get their usernames, based on their naming convention of firstname.lastname.

The following emails were exposed:
- Laura.hayes@nexus.corp
- Michael.chen@nexus.corp
- Sarah.johson@nexus.corp
- Robert.wilson@nexus.corp
- David.brown@nexus.corp
- James.wright@nexus.corp

 ### Directory Discovery
 ```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

![Gobuster Results](screenshots/gobuster_domino.png)

| Endpoint     | Status Code    | Size (Bytes) | Potential Significance / Function |
| :---         | :---           | :---           | :---                         |
| `/admin`     | 301 (Redirect) | 314  | Restricted administrative login or control panel. |
| `/api`       | 301 (Redirect) | 312  | Application Programming Interface backend routing. |
| `/backup`    | 301 (Redirect) | 315  | Exposed configuration or database archive directory. |
| `/index.php` | 200 (Success)  | 861  | Primary login interface. |
| `/javascript`| 301 (Redirect) | 319  | Client-side application script repository. |
| `/static`    | 301 (Redirect) | 315  | Static media, CSS stylesheets, and asset store. |
| `/support`   | 301 (Redirect) | 316  | Customer service or ticketing portal subsystem. |
| `/.htaccess` | 403 (Forbidden)| 278  | Server configuration file (access denied). |
| `/.htpasswd` | 403 (Forbidden)| 278  | Apache basic authentication credentials file (access denied). |


## 5. Vulnerability Analysis
## 6. Exploitation
## 7. Initial Access
## 8. Privilege Escalation
## 9. Proof/Flags
## 10. Attack Chain Summary
## 11. Lessons Learned
## 12. Tools Used
