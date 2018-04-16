# BSides 2018 Vulnerable VM

192.168.56.101

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.5
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.2.22 ((Ubuntu))

Apache is default content. FTP server anonymous webroot has no write access, but there is a public directory containing a backup of usernames.

abatchy
john
mai
anne
doomguy

I'm guessing I'm supposed to bruteforce? The FTP server is anonymous-only, so I am probably supposed to bruteforce SSH logins.

[ERROR] target ssh://192.168.56.101:22/ does not support password authentication.

The only user who is not authenticated via public key is anne. We will try her. Using the rockyou list, we get a hit:

DATA] attacking ssh://192.168.56.101:22/
[22][ssh] host: 192.168.56.101   login: anne   password: princess

We have a shell ;)

Anne has sudo access, so... I have root? That was easy.

Congratulations!

If you can read this, that means you were able to obtain root permissions on this VM.
You should be proud!

There are multiple ways to gain access remotely, as well as for privilege escalation.
Did you find them all?

@abatchy17

Since it was so easy, I decided to poke around a bit to find other ways to gain access..

* /backup_wordpress on the server is running an old and outdated version of WordPress, which is vulnerable to RCE via PHPMailer amongst other things. The RCE vulnerability is trivially exploited just by modifying your Host header when resetting a user's password.

* The wordpress installation can also be bruteforced fairly easily.
