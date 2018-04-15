# TrollCave

Trollcave is a VM available from [VulnHub](https://www.vulnhub.com/entry/trollcave-12,230/) - a  "vulnerable VM, in the tradition of Vulnhub and infosec wargames in general. You start with a virtual machine which you know nothing about – no usernames, no passwords, just what you can see on the network."

As with many traditional wargaming VMs, your objective is simple: gain access to the root account and retrieve a flag to prove you did it. Although it didn't offer a great many surprises, I found it pretty enjoyable. What follows is a write-up of my experience of trying to exploit this VM. It's not exhaustive documentation - from my understanding, there are more than a couple ways to go about exploiting this VM.

## Getting Started

After importing and setting up the VM on a host-only network and confirming that it had obtained an IP address properly, I was ready to begin poking at it. Knowing nothing about the system or what I was in for, I started with a simple netdiscover to find it on the local network, followed by a quick NMap scan to see which ports are open.

