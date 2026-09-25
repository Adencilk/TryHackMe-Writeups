# Silent Monitor

## 1. Introduction
## 2. Reconnaissance
I did Nmap scan on the target to identify the open ports, services running and their versions.

```bash
   nmap -sC -sV 10.49.133.55
```

![Nmap Results](screenshots/nmap_silent_monitor.png)

From the scan I found out that two ports were open :

 1. 22/tcp ssh OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
  
 2. 5050/tcp http Werkzeug httpd 2.0.2 (Python 3.10.12)


## 3. Enumeration
I look into the web running on port 5050 :

![Web](screenshots/web_silent_monitor.png)
## 4. Vulnerability Identification
## 5. Initial Access
## 6. Privilege Escalation
## 7. Evidence
## 8. Findings
## 9. Lessons Learned
## 10. Conclusion
