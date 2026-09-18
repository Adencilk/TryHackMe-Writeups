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


### Discovered Backup Assets

A successful directory index dump of `http://<TARGET_IP>/backup/` revealed two high-value configuration assets:

| File Name | Modification Date | Size (Bytes) | Description / Assessment |
| :---      | :---              | :---         | :---                     |
| `README.txt` | 2026-04-29 | 191 | Plaintext developer notice or instructions. |
| `config.enc` | 2026-05-09 | 112 | Encrypted binary containing system or application configurations. |

Server Banner Information: `Apache/2.4.58 (Ubuntu) Server at 10.49.176.104 Port 80`

### Cryptographic Discovery via README.txt

Reading the plaintext `README.txt` asset exposed the cryptographic algorithm and the location of the key material:

* **Target File:** `config.enc`
* **Encryption Algorithm:** `AES-128-ECB`
* **Key Location Reference:** Found inside client-side asset `static/app.js` (under deployment notes).

This confirms a major configuration flaw: cryptographic keys are referenced within client-side reachable scripts.

###  Extracting the Cryptographic Key

Based on the instructions discovered in the deployment `README.txt`, a direct application request was sent to retrieve the client-side JavaScript asset containing the decryption parameters.

* **Target URL:** `http://10.49.176`
* **Objective:** Locate the hardcoded AES key material.

### Access Control Analysis on Static Directories

Probing the application directories revealed inconsistent security posture configurations across different asset endpoints:

* **Endpoint `/javascript/`:** Returned `403 Forbidden`. Directory browsing is properly disabled on this route.
* **Troubleshooting Step:** Shifted enumeration focus to the public assets directory (`/static/`) to inspect for file exposure or exposed client scripts.

### Identification of Client-Side Script Asset

A directory list audit of the `/static/` folder confirmed the presence and size properties of the targeted script:

* **Asset Identified:** `app.js`
* **File Size:** 1.3 KB (1.3K)
* **Status:** Fully reachable via the public path

This validates that the application logic file mentioned in the developer notes is accessible for static extraction.

### Resolution of File Path Identification Errors

Direct resource mapping against client-side script assets requires strict adherence to file naming rules. Appending structural directory delimiters (a trailing slash `/`) to file names causes internal resource resolution errors:

* **Malformed Path Attempt:** ` curl -s http://10.49.176.104/static/app.js/` -> Returns `404 Not Found` (Server interprets the file target as a directory path).
* **Validated Resource Path:** ` curl -s http://10.49.176.104/static/app.js` -> Requests the raw script code directly.

### Investigation into Empty Script Payloads

While directory auditing confirmed the presence of `app.js` with a file size weight of 1.3K, a direct request returned a zero-byte response. 

* **Symptom:** `curl -s` generates an empty response body.
* **Potential Root Causes:** 
  1. High-volume server connections or network drops.
  2. Server side mime-type filtering rules altering output stream delivery.
* **Remediation & Testing Strategy:** Pivoted from basic `curl` pipes to localized `wget` asset downloads to handle persistent cache transfers.

### Cryptographic Decryption & Configuration Extraction

Following the acquisition of the plaintext AES-128-ECB key material (`N3xusK3y2024!!`) from the exposed client-side script asset, an exploitation pipeline was established to extract the contents of the protected configuration archive.

#### 1. Asset Acquisition
The encrypted configuration binary was downloaded locally via `wget` targeting the unauthenticated backup pathway:
```bash
wget http://<TARGET_IP>/static/app.js
```

#### 2. OpenSSL Decryption Routine
Using the parameters discovered in the developer's deployment documentation, an OpenSSL invocation was performed using a symmetric decryption pass to crack the raw ciphertext stream:
```bash
openssl enc -d -aes-128-ecb -nosalt -in config.enc -pass pass:N3xusK3y2024!!
```

#### 3. Extracted Configuration Analysis
The resulting plaintext revealed internal application parameters and environment secrets critical to escalating credentials within the NexusCorp Portal.

### 📥 Automated Asset Acquisition Strategy

To transition from active network enumeration to offline cryptographic exploitation, remote targets must be transferred to local disk arrays. The command-line retrieval tool `wget` was deployed to handle this interaction.

#### Technical Execution Layout:

```bash
wget http://10.49.176
```

* **Utility Function (`wget`):** Initiates a headless HTTP standard socket request to mirror remote payloads natively.
* **Network Target Mapping (`10.49.176.104`):** Defines the active hosting boundary of the vulnerable target infrastructure.
* **Resource Path Resolution (`/backup/config.enc`):** Directs the web service daemon to access the unauthenticated backup file array directly.
* **Exploitation Purpose:** Moves the encrypted ciphertext repository into local execution memory, enabling symmetric cipher analysis via OpenSSL.

### Offline Cryptographic Decryption

With the encrypted payload (`config.enc`) successfully downloaded locally, an offline decryption pipeline was executed using the OpenSSL toolkit to reverse the symmetric cipher.

#### Execution Syntax:
```bash
openssl enc -d -aes-128-ecb -nosalt -in config.enc -pass pass:N3xusK3y2024!!
```

* **Cipher Specification (`-aes-128-ecb`):** Selected to match the parameters disclosed in the developer configuration files.
* **Key Material Passphrase (`pass:N3xusK3y2024!!`):** Injected natively to satisfy the symmetric block key constraint without interactive prompting.
* **Exploitation Objective:** Expose the unencrypted backend environment configuration parameters to compromise upstream application components.

### Handling Shell Character Conflicts (History Expansion)

When executing cryptographic commands with passwords containing consecutive exclamation marks (`!!`), standard Bash shells interpret the characters as a shortcut to repeat the last command in history. This causes input duplication and terminal hanging.

* **Symptom:** Shell automatically retypes or loops old commands continuously.
* **Mitigation applied:** Enclosed the password string in absolute single quotes (`'N3xusK3y2024!!'`) to instruct the shell interpreter to treat the characters as pure text instead of functional bash commands.

###  JWT Authentication Bypass via Algorithm 'None' Vulnerability

Direct authentication attempts via the front-door portal form (`/index.php`) failed due to environment access control restrictions. Analysis pivoted to abusing broken cookie verification handling within the backend API profiles subsystem.

#### Vulnerability Mechanics
The application verifies user states via JSON Web Tokens (JWT) inside the `nexus_session` cookie string. However, the cryptographic verification library suffers from an **Algorithm None** flaw, allowing unauthenticated API requests if the signature parameter is stripped and the header algorithm flag is explicitly declared as `none`.

#### Execution Command (Profile Notes Extraction):
```bash
curl -s http://10.49.176 -H "Cookie: nexus_session=eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyIjoibGF1cmEuaGF5ZXMiLCJyb2xlIjoidXNlciJ9."
```
* **Target Objective:** Impersonate a low-privileged session index to interact with backend endpoints and extract user profile notes.













## 5. Vulnerability Analysis
## 6. Exploitation
## 7. Initial Access
## 8. Privilege Escalation
## 9. Proof/Flags
## 10. Attack Chain Summary
## 11. Lessons Learned
## 12. Tools Used
