# Kenobi Penetration Testing Report
## Executive Summary
This report documents the assessment of the TryHackMe Kenobi room.
The objective was to identify vulnerabilities, gain initial access,
escalate privileges, and document the findings.

## Scope
- Target: TryHackMe Kenobi
- Platform: TryHackMe
- Difficulty: Easy

## Reconnaissance
### Nmap scan
An initial Nmap scan was performed to identify open ports and running services.
Command:
 '''bash
 nmap -Pn <TARGET_IP>
 7 ports were found to be open:
 
PORT       STATE   SERVICE
21/tcp     open     ftp
22/tcp     open     ssh
80/tcp     open     http
111/tcp    open     rpcbind
139/tcp    open     netbios-ssn
445/tcp    open     microsoft-ds
2049/tcp   open      nfs
## Results


## Enumeration

## Initial Access

## Privilege Escalation

## Flags Obtained

## User Flag

## Root Flag

## Recommendations 
- Keep software updated.
- Apply the principle of least privilege.
- Disable unnecessary services.
- Regularly review system configurations.

  ## Conclusion
  This room provided practical experience
  in reconnaissance, service enumeration,
  explotation, and linux privilege escalation.
