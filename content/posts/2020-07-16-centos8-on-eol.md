---
title: "Installing CentOS 8 on Old Hardware"
date: 2020-07-16
categories:
  - Linux
tags:
  - Linux
  - PXE
  - Foreman
  - Drivers
  - Automation
  - DevOPS
---

> TL;DR: [https://gist.github.com/learnitall/80d5474045a019eba30c3738bd3c57ca#file-upgrade_centos_8-yml](https://gist.github.com/learnitall/80d5474045a019eba30c3738bd3c57ca#file-upgrade_centos_8-yml)

There's the classic story of a teenager becoming dissapointed when they realize their first car is going to be a piece of shit. It makes sense: new drivers, kids especially, are more likely to get into an accident so why give them a super fancy car if it's going to be banged-up anyway? That same motif ironically also applied to my first server. I'm fortuante enough to have relatives in 'the business' who donated some equipment to help me get started in creating my own home development lab, but like my first car, the equipment isn't brand new and doesn't have all the bells and whistles found in a modern datacenter.

For instance, I've had a power supply fail, a network card fail, I have a drive that's currently failing, and now my raid controllers are no longer supported in CentOS 8.

You can see how that would make installing CentOS 8 fairly difficult.

## The Problem

I had configured my [Foreman](https://theforeman.org/) deployment server to install CentOS 8 over PXE, but had varying levels of success and failure. The install would fail at two places:

1. A `dracut-initqueue` timeout right after the network interfaces come online.
2. Kickstart would fail to recognize storage devices and wait for user input before continuing.

I tried redownloading cached files on my `tftpboot` server, reinitializing my disks, turning off raid, etc. but unfortunately had all the same results. After some Googling, I came across [this page from RedHat](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/considerations_in_adopting_rhel_8/hardware-enablement_considerations-in-adopting-rhel-8#removed-adapters_hardware-enablement), which lists removed hardware support in CentOS 8. Luckily, my raid controller's PCI ID is listed there, so least I had discovered the the root of the problem. I needed to somehow manually add in driver support for my raid controller during the install.

## Solutions

### Idea 1: Rebuild Initial Ramdisk

I had messed around a bit with modifying the [Initial Ramdisk](https://en.wikipedia.org/wiki/Initial_ramdisk) while working for the ITLL at the University of Colorado Boulder, and while trying to install Nvidia drivers on linux back in high school, but my knowledge of how it works is still pretty limited. The idea though was to rebuild the initramfs that Foreman downloads for PXE-booting CentOS 7 and add the driver for my raid controllers manually. This way the driver will be loaded during the install and then persist each time the kernel is updated. Luckily, I didn't have to go this route because:

1. It would've been a pain to debug since the PXE-booting process takes about five minutes on each install.
2. I had no idea how to do this and the process seemed to have a lot of potential areas of failure.
3. There's already something build into Anaconda that achieves this.

### Idea 2: inst.dd

While doing more Googling, I came across [this blog post from ELRepo](https://elrepoproject.blogspot.com/2019/08/rhel-80-and-support-for-removed-adapters.html) which talks about addressing this exact issue. [ELRepo](http://elrepo.org/) has a repository of driver update disks (DUD) which contain rpms that provide drivers which support legacy hardware. These disks can be provided to the anaconda installer through the [`inst.dd`](https://github.com/rhinstaller/anaconda/blob/rhel-8.0/docs/boot-options.rst/#instdd)  boot option. Anaconda will then search through the provided disk for available driver updates and load those needed, either through a manual or automated procedure.

To test this, out, I used [ELRepo's list of supported devices](http://elrepo.org/tiki/DeviceIDs) and the previously mentioned RedHat documentation on removed hardware support in order to determine which DUD I needed to download (turned out to be `megaraid_sas`). I downloaded the ISO, flashed it onto a USB drive, plugged it into one of my servers, PXE-booted that server and manually added the `inst.dd` option to the Anaconda boot parameters. During Anaconda's execution, I was given a prompt for selecting drivers to install and we were on our way!

```
[  OK  ] Created slice system-driver\x2dupdates.slice.
[  OK  ] Started Setup Virtual Console.
         Starting Driver Update Disk UI on tty1...
DD: starting interactive mode

(Page 1 of 1) Driver disk device selection
    /DEVICE  TYPE     LABEL                 UUID
 1) sda      iso9660  OEMDRV                2020-05-01-16-29-07-00
# to select, 'r'-refresh, or 'c'-continue: 1
DD: Examining /dev/sda
mount: /media/DD-1: WARNING: device write-protected, mounted read-only.

(Page 1 of 1) Select drivers to install
  1) [ ] /media/DD-1/rpms/x86_64/kmod-megaraid_sas-07.710.50.00-1.el8_2.elrepo.x86_64.rpm
# to toggle selection, or 'c'-continue: 1

(Page 1 of 1) Select drivers to install
  1) [x] /media/DD-1/rpms/x86_64/kmod-megaraid_sas-07.710.50.00-1.el8_2.elrepo.x86_64.rpm
# to toggle selection, or 'c'-continue: c
```

The driver was installed and kickstart began to recognize my disks! Unfortunately however, since I installed the DUD with a USB drive, and it was the first drive recognized by the system, it came up as `/dev/sda`. This messed up my partition tables because I used a scheme which assumed `/dev/sda` was the location where the operating system was to be installed. Not good.

Luckily, `inst.dd` is powerful enough to load DUDs from nfs file shares, which is great because that's a +1 for automation. Now I could inject the `inst.dd` boot parameter into my PXELinux template and have my servers automatically load the DUD they need from an nfs server during boot, without manual intervention with a USB drive on my part. After setting up a nfs share at `/srv/nfs4/rpms` on my deployment host and placing the DUD inside, I set the `kernel-cmd` host parameter in Foreman to `inst.dd=nfs:nfsvers=4:<deployment host ip>:/rpms/dd-megaraid_sas-07.710.50.00-1.el8_2.elrepo.iso`. Genius right?

Eh, this worked OK at best. I had varying success in my environment. The `dracut-initqueue timeout` error was still an ongoing issue, although less frequent. I came to believe that there was just another driver missing in CentOS 8 that my old hardware needed, but I really didn't want to go through the effor to figure out which one(s). Another approach was needed.

### Idea 3: Install CentOS 7 and Upgrade

Even though I couldn't install CentOS 8, I could still install CentOS 7 like slicing butter. I found a couple guides that work through upgrading CentOS 7 to 8, so I used them to create an ansible playbook that automates the process and installs a new kernel from ELRepo, which has legacy driver support built in:

> Link in case the JS fails: [https://gist.github.com/learnitall/80d5474045a019eba30c3738bd3c57ca#file-upgrade_centos_8-yml](https://gist.github.com/learnitall/80d5474045a019eba30c3738bd3c57ca#file-upgrade_centos_8-yml)

{{< gist learnitall 80d5474045a019eba30c3738bd3c57ca >}}

I'm able to assume a lot in this playbook because I knew it was going to be run against systems that just got done with a fresh, minimal install of CentOS 7. So far it works pretty well! I'm concerned that there are going to be some package dependency issues down the line that I'll discover as I start to use my servers, but we'll just have to wait and see.

## Conclusion

Ansible is a beast and I've really enjoyed my time working with it. I've found that a lot of the time when I need to deploy my servers, it's easier to just spin up a VM, test out an ansible playbook and then run it against a super minimal/basic installation of CentOS in my environment, rather than trying to 'deploy it right the first time' with kickstart templates.

If you have any questions or comments about this post, please feel free to reach out!
