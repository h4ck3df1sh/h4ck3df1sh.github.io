---
title: My Nmap - Function
date: 2023-06-05 18:06 +0200
categories: [Resources,Function]
tags: [my-nmap, function]
---
I have included a function in my .zshrc file that allows me to easily identify the open ports using ***[Nmap](https://nmap.org/)*** tool and the services running on the host. This function presents the information in a clean and organized format, which I find very convenient.
```bash
#!/bin/bash

if [ $# -eq 0 ]; then
  echo "Usage: automated_nmap.sh targetIP"
  exit
fi

# Function to trap CTRL+C and perform cleanup
ctrl_c() {
    echo "Caught CTRL+C. Performing cleanup..."
    clear
    tput cnorm
    exit 1
}

# Set up trap to call ctrl_c() function on CTRL+C
trap ctrl_c INT

clear

tput civis

check_os() {
    if ttl=$(ping -c 1 "$1" | grep -oE 'ttl=[0-9]+ ' | awk '{print $2}' FS="=" 2>/dev/null); then
        if [ "$ttl" -le 64 ]; then
            echo "Linux"
        elif [ "$ttl" -le 128 ]; then
            echo "Windows"
        else
            echo "Unknown operating system"
        fi
    else
        echo "Error: Unable to ping "$1". Maybe ping requests are disabled on target."
    fi
}

targetIP="$1"

echo -e "\n[*] Extracting information...\n"
echo -e "\n\t[*] Target Ip: $targetIP\n"

# Check operating system using TTL
operating_system=$(check_os $targetIP)
echo -e "\t[*] Operating system: $operating_system\n"

# Map all open ports
sudo nmap -sS -Pn -n -T5 --min-rate 5000 -p- --open -oN allPorts $targetIP > /dev/null

# Get open ports into a variable
ports="$(cat allPorts | awk /PORT/,/\n\n/| head -n -2 | tail -n +2 | awk '{print $1}' FS="/" | sed 's/$/\,/g' | tr -d '\n' | sed 's/.$//')"
echo -e "\t[*] Open ports: $ports\n"

# Retrieve services info 
sudo nmap -sV -Pn -n -p$ports -oN openPorts $targetIP > /dev/null
echo -e "\t[*] Services information: \n"
cat openPorts | awk /PORT/,/Warning:/|  head -n -1 | sed 's/^/\t\t/' | awk '/PORT/,/1 service/{if (/1 service/) {gsub(/^\n/,""); print; exit} else print}' | sed '/1 service/d' | awk '/PORT/{flag=1;print}flag && /[1-9][0-9]{0,4}\/tcp/{print
;next}';  echo;

 rm allPorts openPorts
 
 tput cnorm
```
