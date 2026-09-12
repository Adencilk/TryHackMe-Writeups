# TryHackMe - Domino

## Overview

- Platform: TryHackMe
- Room: Domino
- Category: Penetration Testing
- Target System: Linux (NexusCorp Employee Portal)


## Objective

Document the methodology used to assess the Domino machine, including reconnaissance, enumeration, exploitation, and privilege escalation.

## Methodology

1. Reconnaisance
2. Enumeration
3. Vulnerability Identification
4. Initial Access
5. Privilege Escalation
6. Lessons Learned

## 1. Reconnaissance

## Nmap Scan
```bash
nmap -sC -sV <TARGET_IP>
```
**Nmap Scan Results:**

![Nmap Scan Results](sreenshots/domino_nmap.png]

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.58 ((Ubuntu))
|_http-title: NexusCorp Portal
|_http-server-header: Apache/2.4.58 (Ubuntu)
```



## 2. Enumeration

## 3. Vulnerability Identification

## 4.Initial Access

## 5. Privilege Escalation

## 6. Lessons Learned

