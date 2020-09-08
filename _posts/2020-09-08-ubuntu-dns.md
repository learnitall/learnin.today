---
title: "Setting Permanent and Persistent DNS on Ubuntu 20.04"
categories:
  - Linux
tags:
  - DNS
  - network
  - configuration
  - ubuntu
  - linux
---

> Tl;dr: [https://askubuntu.com/questions/820667/how-do-i-permanently-configure-the-dns-server-list-in-16-04](https://askubuntu.com/questions/820667/how-do-i-permanently-configure-the-dns-server-list-in-16-04).

This post is going to be one of my favorites, as its contents will save me hundreds of dollars in popping Tylenol to treat all the migraines this problem has caused me. You might be wondering: *It's DNS, what's the big deal?* Let me explain:

## The Problem

I have a little open-source, private cloud environment that I've been building for a couple years now. I've setup a FreeIPA instance to provide centralized user authentication and authorization, which is such a blessing, as well as a little [bastion host](https://en.wikipedia.org/wiki/Bastion_host) running Ubuntu that I can SSH into from a VPN tunnel to access the environment. This means that whenver I login over the VPN and then SSH to the bastion host, it'll contact the FreeIPA instance to auth my domain credentials. This 'contacting' process involves resolving the hostname of the FreeIPA instance, which is where our problem starts.

Our problem continues with the fact that I've had a strange experience with setting DNS in Ubuntu. My DHCP server tells clients to use a local DNS resolver that is authoritative for the environment's domain, as well as Google's public DNS as a backup. For some reason my Ubuntu-running bastion host will *sometimes* prioritize using Google's public DNS over my local resolver, causing it to not be able to resolve the name of any of the servers in my environment, including the FreeIPA instance.
 
This hasn't been too crippling since my servers are setup to use FreeIPA for auth through the System Security Services Daemon ([SSSD](https://sssd.io/)). It's this sweet little tool that plugs right into the Pluggable Authentication Modules ([PAM](https://en.wikipedia.org/wiki/Linux_PAM)) and manages LDAP and Kerberos integration with one config file. SSSD provides a caching mechanism for users, sudo rules and the like, meaning if your network goes down, users can still temporarily authenticate while the cache is still valid.

This means that when the bastion host in my environment decides to suddenly start using Google's DNS, I can still have the necessary authorization to fix it since that information is cached.

Recently however, I migrated my FreeIPA instance from Docker to Podman, causing this cache to expire on my bastion host. I was still somehow able to login to my bastion host, but sudo was crippled and each command took ages to execute.

That's what inspired, err, forced me to write this article.

## The Solution

If you want to set persistent and permanent DNS servers for your Ubuntu box that uses DHCP, don't use `resolv.conf`, don't use `netplan`, use `dhclient`. Ubuntu will respect the set of DNS servers received from DHCP over anything else, but for some reason I've been having trouble forcing Ubuntu to also respect the prescribed *order* of the DNS servers it receives through DHCP. (If you know anything about this, email me!) What's great though is we can tell our dhclient exactly what DNS servers we want and at what priority. In `/etc/dhcp/dhclient.conf` add a line like this:

```
supersede domain-name-servers dns1, dns2...;
```

And voila, welcome to bliss.
