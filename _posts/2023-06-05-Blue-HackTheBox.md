---
title: Blue - HackTheBox
date: 2023-06-05 18:06 +0200
categories: [Easy,Windows]
tags: [blue, hackthebox]
---

![Blue](/assets/img/obsidian/Blue.png)

Hello! Today we will be tackling the machine Blue on HackTheBox. In this walk-through, we will cover the following points:

1. [Introduction to the machine](#introduction-to-the-machine)
2. [Reconnaissance and information gathering](#reconnaissance-and-information-gathering)
3. [Exploitation and gaining initial access](#exploitation-and-gaining-initial-access)
4. [Privilege escalation techniques](#privilege-escalation-techniques)
5. [Post-exploitation](#post-exploitation)
6. [Conclusion and key takeaways](#conclusion-and-key-takeaways)

Throughout this walk-through, we will be utilizing various tools and techniques to overcome challenges and progress towards rooting the machine. So, let's dive in and get started with the HackTheBox machine named Blue. Happy hacking!

## Introduction to the machine
This Windows host is running SMB and RPC services. The SMB service on this host has a known vulnerability called EternalBlue. Exploiting this vulnerability allows an attacker to gain direct access to the system as NT Authority/System, granting them full privileges. The exploit for EternalBlue is well-known and relatively easy to execute.
## Reconnaissance and information gathering
After executing a customized [Nmap](/posts/My-Nmap-Function/) we obtained the following information from the victim machine:
```bash
[*] Target Ip: 10.10.10.40

[*] Operating system: Windows

[*] Open ports: 135,139,445,49152,49153,49154,49155,49156,49157
[*] Services information: 

	PORT      STATE SERVICE      VERSION
	135/tcp   open  msrpc        Microsoft Windows RPC
	139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
	445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
	49152/tcp open  msrpc        Microsoft Windows RPC
	49153/tcp open  msrpc        Microsoft Windows RPC
	49154/tcp open  msrpc        Microsoft Windows RPC
	49155/tcp open  msrpc        Microsoft Windows RPC
	49156/tcp open  msrpc        Microsoft Windows RPC
	49157/tcp open  msrpc        Microsoft Windows RPC
```

```bash
Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
|_smb-vuln-ms10-054: false

```
## Exploitation and gaining initial access
According to the Nmap report, a known vulnerability has been identified, which can be exploited using a specific CVE (Common Vulnerabilities and Exposures) reference. After researching the provided CVE, it was discovered that the vulnerability is referred to as EternalBlue. This particular vulnerability affects Windows servers running SMBv1 and can be exploited using either public exploits or the Metasploit framework.

To perform manual exploitation without relying on the Metasploit framework, there is a publicly available tool called [AutoBlue](https://github.com/3ndG4me/AutoBlue-MS17-010) available on GitHub. This tool, located at AutoBlue GitHub repository, enables the generation of shell code and exploitation of the vulnerable service, eliminating the need to execute the entire Metasploit framework.

By utilizing the Metasploit framework and the 'ms17-010' exploit module, you will be able to automate the exploitation process and gain unauthorized access to systems affected by the EternalBlue vulnerability. Here are the steps you can follow:
1. Open the Metasploit framework.
2. Search for the 'ms17-010' exploit module, which specifically targets the EternalBlue vulnerability (CVE-2017-0144).
3. Select the first exploit that appears in the search results for 'ms17-010' and set up the required options, including the remote host IP address and the payload you wish to use. You can leave the other options at their default settings for now.

Once the options are set, you can proceed to run the exploit and initiate the exploitation process.

> In order to obtain the shell, it may be necessary to run the exploit multiple times
{: .prompt-tip }

```bash
msf6 > search ms17-010

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection
   4  exploit/windows/smb/smb_doublepulsar_rce  2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution
```
## Privilege escalation techniques
Given that the vulnerability grants us a NT Authority\System shell, there is no requirement for vertical or horizontal escalation as we already possess the highest level of privileges. As the NT Authority\System user, we have full control and extensive capabilities over the system, eliminating the necessity for any further privilege escalations.
```bash
meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
meterpreter > sysinfo
Computer        : HARIS-PC
OS              : Windows 7 (6.1 Build 7601, Service Pack 1).
Architecture    : x64
System Language : en_GB
Domain          : WORKGROUP
Logged On Users : 2
Meterpreter     : x64/windows
```

## Post-exploitation
It's time to search for intriguing files within the computer. Begin by examining the network environment and exploring methods for establishing a permanent backdoor, such as creating a new user or utilizing alternative techniques.
For the purpose of this post and the target machine, the primary focus will be on acquiring both the user and root flags. These flags are located at: 
- `c:\Users\haris\Desktop\user.txt`
- `c:\Users\Administrator\Desktop\root.txt`
## Conclusion and key takeaways
As demonstrated with this machine, exploiting EternalBlue is relatively straightforward, highlighting the importance of keeping systems up to date and applying recent patches. It is crucial to recognize the significance of returning to the fundamentals, as they serve as a solid foundation for maintaining a secure system.