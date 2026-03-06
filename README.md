# Dev Machine – Linux Exploitation Lab

This lab demonstrates a full penetration testing workflow against a Linux target.
The attack chain includes **network enumeration, NFS exploitation, CMS vulnerability exploitation, SSH access using exposed credentials, and privilege escalation via misconfigured sudo permissions**.

The exercise highlights the importance of **proper service configuration, credential protection, and least-privilege access control**.

---

# Initial Enumeration

After obtaining the target IP address, an **Nmap scan** was performed to identify exposed services.

```bash
nmap -T4 -p- -A <target-ip>
```

The scan revealed the following key services:

```
80/tcp    open  http     Apache httpd 2.4.38 ((Debian))
|_http-title: Bolt - Installation error
|_http-server-header: Apache/2.4.38 (Debian)

8080/tcp  open  http     Apache httpd 2.4.38 ((Debian))
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported: CONNECTION
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: PHP 7.3.27-1~deb10u1 - phpinfo()

2049/tcp  open  nfs      3-4 (RPC #100003)
```

Important findings:

* **Port 80** — Web server (Apache)
* **Port 8080** — Secondary web server
* **Port 2049** — NFS (Network File System)

---

# Web Enumeration

Both web services were explored through a browser.

```
http://<target-ip>:80
http://<target-ip>:8080
```

* screenshots/01-port-80.png
* screenshots/02-port-8080.png

---

# Directory Enumeration

To discover hidden directories, **ffuf** was used.

Example command:

```bash
ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt:FUZZ -u http://<target-ip>/FUZZ
```

Results:

**Port 80 discovered directories**

```
/public
/src
/app
/vendor
```

**Port 8080 discovered directory**

```
/dev
```

* screenshots/03-fuff-dirbust-port-80.png
* screenshots/04-fuff-dirbust-port-8080.png

---

# NFS Enumeration

Since **NFS (port 2049)** was exposed, the available mount points were enumerated.

```bash
showmount -e <target-ip>
```

A directory was created locally to mount the share:

```bash
mkdir /mount/dev
```

* screenshots/04-show-mountpoint-and-dir.png

The share was mounted:

```bash
mount -t nfs <target-ip>:/srv/nfs /mount/dev
```

Inside the mounted directory a **password-protected zip file** was discovered.

* screenshots/05-mounted.png

---

# Cracking the ZIP Archive

The ZIP archive was cracked using **fcrackzip**.

```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt save.zip
```

After cracking the password, two files were extracted:

* `todo.txt`
* `id_rsa` (SSH private key)

- screenshots/06-fcrack-crack-unzip.png
- screenshots/07-view-2-files-and-ssh.png

The `todo.txt` file contained a signature referencing **jp**.

However attempting SSH using **jp** did not work.

---

# Investigating Web Directories

Next, the previously discovered directories were investigated.

```
/public
/src
/app
```

* screenshots/08-public.png
* screenshots/09-scr.png
* screenshots/10-app.png

Inside the `/app` directory several useful folders were discovered including:

```
/cache
/config
```

Within `/config`, the file **config.yml** contained credentials.

* screenshots/11-config-yml-file.png
* screenshots/12-config-yml-file2.png

These credentials were saved for later use.

---

# BoltWire CMS Discovery

From the **port 8080 scan**, the `/dev` directory was explored.

* screenshots/13-dev-path.png

This revealed a **BoltWire CMS installation**.

The CMS allowed user registration.

* screenshots/14-regiser-page.png
* screenshots/15-registered.png

---

# Searching for Vulnerabilities

A search for **BoltWire exploits** was performed using:

```bash
searchsploit boltwire
```

* screenshots/16-searchsploit-result.png

An exploit involving **directory traversal** was discovered.

* screenshots/17-the-exploit.png

---

# Exploiting Directory Traversal

The following payload was used:

```
index.php?p=action.search&action=../../../../../../../etc/passwd
```

Note: authentication was required before executing the exploit.

* screenshots/18-traversal-attack-executed-1.png
* screenshots/19-traversal-attack-executed-2.png
* screenshots/20-traversal-attack-executed-3.png

The exploit successfully exposed the **/etc/passwd** file.

---

# Discovering Valid User

From `/etc/passwd`, the correct username was identified as:

```
jeanpaul
```

Using the previously obtained SSH key:

```bash
ssh -i id_rsa jeanpaul@<target-ip>
```

* screenshots/21-ssh-using-etc-passwd-found-credential.png

A **passphrase** was required.

The password discovered earlier in `config.yml` was tested:

```
I_love_java
```

* screenshots/22-passwd-yml-credential.png

Authentication succeeded.

* screenshots/23-gained-low-level-shell.png

A low-privilege shell was obtained.

---

# Privilege Escalation Enumeration

Checking sudo permissions:

```bash
sudo -l
```

Result:

The user **jeanpaul** could run:

```
zip
```

as **root without a password**.

* screenshots/24-test-sudo-ability-and-confirmed.png

---

# Exploiting Sudo ZIP Privilege

Searching for privilege escalation methods led to **GTFOBins**.

* screenshots/25-exploit-gtfo-1.png

Filtering for **sudo exploits** revealed a technique for abusing **zip**.

* screenshots/26-exploit-gtfo-2.png
* screenshots/27-exploit-gtfo-3.png

Executing the GTFOBins command allowed a root shell.

* screenshots/28-root-access-to-dev-machine.png

---

# Root Access Achieved

Privilege escalation was successful, resulting in **root access** on the system.

---

# Key Lessons Learned

This lab demonstrates several important penetration testing techniques:

• Exploiting exposed **NFS shares**
• Cracking password-protected archives
• Extracting credentials from **misconfigured web applications**
• Exploiting **directory traversal vulnerabilities**
• Leveraging **SSH keys for authentication**
• Escalating privileges via **misconfigured sudo permissions**

---

# Skills Demonstrated

* Network Enumeration
* Web Application Enumeration
* NFS Exploitation
* Credential Discovery
* Directory Traversal Attacks
* SSH Key Authentication
* Linux Privilege Escalation
* GTFOBins Exploitation

---
