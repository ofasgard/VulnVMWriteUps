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

## Getting RCE

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

*/var/log/apache2/access.log (and variants)
*/var/log/apache2/error.log (and variants)
*/proc/self/environ
*The php:// scheme is available, but does not seem to work for sending code via POSTDATA.
*The data:// scheme is available, but also does not seem to work for code inclusion.
