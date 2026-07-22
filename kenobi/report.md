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
 '''

 7 ports were found to be open:
 ### Open ports
|PORT  |    STATE |   SERVICE         |
|------|----------|-------------------|
|21    |    open  |    ftp            |
|22    |    open  |    ssh            |
|80    |    open  |    http           |
|111   |    open  |    rpcbind        |
|139   |    open  |    netbios-ssn    |
|445   |    open  |    microsoft-ds   |
|2049  |     open |    nfs            |

 **Screenshot.**
 ![Nmap Scan](screenshots/kenobi_nmap.png)

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
