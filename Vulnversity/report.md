# Vulnversity Report

## Executive Summary
This report documents the successful completion of the TryHackMe Vulnversity room. The assessment covered reconnaissance, enumeration, exploitation, and privilege escalation against the target machine.

## Scope
- Platform: TryHackMe
- Room: Vulnversity
- Objective: Gain user and root access while documenting the methodology.

## Methodology

### 1. Reconnaissance

### Objective
Identify open ports and exposed network services on the target machine.

### Command Used

```bash
nmap -Pn <TARGET_IP>
```

### Findings

The Nmap scan identified six open TCP ports:

| Port | Service         | Purpose |
|------|-----------------|---------|
| 21   | FTP             | File Transfer Protocol |
| 22   | SSH             |  Secure Shell remote access |
| 139  | NetBIOS-SSN     | SMB/Windows file sharing |
| 445  | Microsoft-DS    | SMB file sharing |
| 3128 | Squid HTTP Proxy| Web proxy service |
| 3333 | DEC Notes       | Additional network service requiring further enumeration |

### Analysis

The scan revealed multiple exposed services that could provide potential attack vectors. FTP and SMB services may allow anonymous access or misconfigurations, while the Squid proxy and other services should be investigated further during the enumeration phase.
### 2. Enumeration

### Objective
Identify service versions and gather additional information about the exposed services.

### Command Used

```bash
nmap -sC -sV <TARGET_IP>
```

### Findings

Service detection identified the following services:

| Port | Service    | Version |
|------|------------|---------|
| 21   | FTP        | vsftpd 3.0.5 |
| 22   | SSH        | OpenSSH 8.2p1 |
| 139  | SMB        | Samba smbd 4.6.2 |
| 445  | SMB        | Samba smbd 4.6.2 |
| 3128 | HTTP Proxy | Squid Proxy 4.10 |
| 3333 | HTTP       | Apache httpd 2.4.41 |

### Analysis

The service detection scan provided version information for each exposed service. The HTTP service on port 3333 and the SMB services on ports 139 and 445 were identified as key targets for further enumeration. Additional testing was planned to identify potential vulnerabilities or misconfigurations.
### Web Directory Enumeration

#### Objective
Discover hidden directories and web resources that are not directly linked from the web application.

#### Command Used

```bash
gobuster dir -e -u http://<TARGET_IP>:3333 -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
```

#### Findings

The directory enumeration identified several accessible directories:

| Directory | HTTP Status |
|-----------|-------------|
| /images   | 301 Moved Permanently |
| /css      | 301 Moved Permanently |
| /internal | 301 Moved Permanently |

#### Analysis

The discovery of the `/internal` directory was particularly significant because it indicated the presence of content not intended for direct navigation. This directory was selected for further investigation as a potential entry point into the target system.

### File Upload Extension Testing

#### Objective
Determine which PHP file extensions are accepted by the application's upload filter.

#### Tool Used
- Burp Suite Intruder

#### Method
The file upload request was intercepted and sent to Burp Suite Intruder. A list of common PHP extensions was tested to identify which extensions bypassed the upload restrictions.

#### Outcome
The Intruder attack identified differences in server responses, indicating that certain PHP extensions were handled differently by the application. This information was used to select an extension for uploading a PHP reverse shell during the exploitation phase.

### 3. Exploitation
- Initial access through the identified vulnerability
- Reverse shell establishment

### 4. Privilege Escalation
- Local enumeration
- Privilege escalation to root

## Tools Used
- Nmap
- Gobuster
- Burp Suite
- Netcat
- Python
- Linux Terminal

## Key Skills Demonstrated
- Network reconnaissance
- Web enumeration
- Exploitation
- Linux privilege escalation
- Technical documentation

## Lessons Learned
This room strengthened practical penetration testing skills and reinforced the importance of systematic enumeration before exploitation.
