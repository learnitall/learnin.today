---
title: "OSA: Preparing Target Hosts with Ansible Playbooks"
categories:
  - OpenStack Ansible
tags:
  - OpenStack Ansible
  - OSA
  - Linux
  - OpenStack
  - Ansible
  - Automation
  - DevOPS
---

Like most humans my age, I make a sh\*t load of mistakes. I'm a young adult making his way through the world and it takes time to become competent enough to, for instance, remember to feed yourself.

In particular though, through the course of setting up a development OpenStack cluster I've had to start over quite a few times. In my defense, it takes a hot minute to learn the ins-and-outs of OpenStack and sometimes [a hard reset is all you need](https://www.youtube.com/watch?v=nn2FB1P_Mn8)... or at least a hard reset is what you're forced to do when you've, uh, messed up that bad.

> Eh it's not prod, so, water under the bridge yeah?

Starting from scratch with [OpenStack Ansible (OSA)](https://docs.openstack.org/openstack-ansible/latest/) isn't the worst thing in the world because your process of trial-and-error involves just modifying your configuration files and then running your playbooks again. However, I've been at a point where I've messed up so poorly that I've needed to reinstall my target nodes entirely, as in, start with a fresh install of linux. This can become tedious because you can't straight up run OSA on fresh nodes because you have to bootstrap them first. (This process can be found in the OpenStack Ansible Deployment Guide on the page ["Prepare the target hosts"](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html).)

This bootstrapping process is like the walk of shame; it can be tedious as hell to do the same set of steps on all your nodes everytime you need to start over, therefore, I put together a playbook that automates the process. This way the next time I mess up my deployment and need to start over, I can fully automate the bootstrapping process for each target node in my cluster. It works fairly well, but I'd like to walk through it because there are some really important steps I've included in the playbook that aren't actually mentioned on the ["Prepare the target hosts"](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html) page.

> This playbook is super tailored towards my deployment needs, which is why it's here on a blog post rather than on Github.

## Configuring Linux

I'm using CentOS 7 for my target nodes. These set of steps in the playbook just setup CentOS by installing needed/wanted packages, configuring kernel modules and system services.

### Package Management

Here we just perform a standard upgrade of all packages and then install needed prerequisites. These are the packages not included in the documentation that I've installed anyway:

|Package(s)|Reason|
|---|---|
|`tmux`, `vim`|I enjoy having these tools on my nodes.|
|`gdisk`|Going to use this later on to format drives.|
|`libsemanage-python`, `policycoreutils-python`|Needed to allow Ansible to manage/check on SELinux.|
|`NetworkManager-glib`, `nm-connection-editor`|Needed for the [nmcli](https://docs.ansible.com/ansible/latest/modules/nmcli_module.html) Ansible module.|
|`@Development Tools`|I had to install this to get the [neutron playbook](https://opendev.org/openstack/openstack-ansible-os_neutron) to run successfully. I was receiving an error stating `gcc` was missing.|

{% raw %}
```yaml
- name: Perform an upgrade on all packages
  yum:
    name: "*"
    state: latest
- name: Install additional software packages
  yum:
    name: "{{ packages }}"
    state: latest
  vars:
    packages:
    - bridge-utils
    - chrony
    - '@Development Tools'
    - gdisk
    - iputils
    - libsemanage-python
    - lsof
    - lvm2
    - NetworkManager-glib
    - nm-connection-editor
    - openssh-server
    - policycoreutils-python
    - python3
    - rsync
    - sudo
    - tcpdump
    - tmux
    - vim
```
{% endraw %}

### Disabling SELinux and firewalld

As of OpenStack Train, the latest supported version of OpenStack that OpenStack Ansible supports, SELinux and firewall rules haven't been created so it's recommended to just turn them off. Note however that **it's important SELinux is set to permissive rather than disabled**. One of the playbooks that OSA runs ([ansible-hardening](https://opendev.org/openstack/ansible-hardening)) will check by default to see if SELinux is running and fail if it isn't. Disabling SELinux will interrupt your playbook, but setting to permissive will pass the relevant task while making sure SELinux doesn't get in the way of your deployment later on.

```yaml
- name: Ensure SELinux is disabled
  selinux:
    policy: targeted
    state: permissive
- name: Ensure firewall is disabled
  service:
    name: firewalld
    state: stopped
    enabled: no
```

### Loading Kernel Modules, Setting NTP, Adding root SSH Key.

This is all straight from the documentation, with the exception of the `br_netfilter` module. I can't remember which one of the OSA playbooks requires this to be loaded, but if it isn't loaded then your `setup-everything.yml` is going to fail at some point. 

{% raw %}
```yaml
- name: Load additional kernel modules
  copy:
    dest: "/etc/modules-load.d/openstack-ansible.conf"
    content: |
      bonding
      br_netfilter
      8021q
- name: Set timezone
  timezone:
    name: America/Denver
- name: Start ntpd
  service:
    enabled: yes
    state: started
    name: ntpd
- name: Enable ntp for time sync
  command:
    cmd: timedatectl set-ntp true
- name: Add root ssh-key
  authorized_key:
    user: root
    state: present
    manage_dir: yes
    exclusive: yes
    comment: 'openstack-ansible'
    key: "{{ lookup('file', '/root/.ssh/osa/id_rsa.pub') }}"
```
{% endraw %}

## Interface Configurations

Alright here is where things get weird. For my deployment I was setting up the following *bridge* interfaces:

|Interface|Vlan ID|Subnet Range|Purpose|
|---|---|---|---|
|br-mgmt|121|10.0.21.0/24|Management interface for service communciation.|
|br-vxlan|122|10.0.22.0/24|Tunnel network for tenants.|
|br-storage|123|10.0.23.0/24|Dedicated storage network.|
|br-ext|124|10.0.24.0/24|Dedicated network for HAProxy external LB.|
|br-vlan|n/a|n/a|L2 Network for tenants.|

My goal was to configure all of these interfaces with the use of the [nmcli](https://docs.ansible.com/ansible/latest/modules/nmcli_module.html) Ansible module, however I had some trouble (see [ansible-collections](https://github.com/ansible-collections)/[community.general](https://github.com/ansible-collections/community.general) issues [#472](https://github.com/ansible-collections/community.general/issues/472) and [#473](https://github.com/ansible-collections/community.general/issues/473)), so I had to do it by hand by executing `nmcli` commands with the classtic [command](https://docs.ansible.com/ansible/latest/modules/command_module.html#command-module) module.

To make this happen I created a few host variables:

{% raw %}

* `{{ opsint }}`: Short for 'openstack interface'. This is the one interface that all of the bridge interfaces listed in the table above will use for their connection. An example of this value is `em1` or `p2p1`.
* `{{ ip }}`: Each bridge interface above is on its own subnet and each of my nodes has the same static host ID in each subnet. This host ID is stored in `ip`. I found that using static addressing was way easier to manage than DHCP because some of the services in the OSA deployment won't wait for the DHCP lease to be acquired before starting, therefore requiring a manual reset.

Firstly, in order to make sure this playbook is [idempotent](https://en.wikipedia.org/wiki/Idempotence) we start by cleaning up existing interfaces:

```yaml
- name: Remove existing interface configurations
  file:
    state: absent
    path: '/etc/sysconfig/network-scripts/ifcfg-{{ item }}'
  with_items:
  - br-mgmt
  - br-storage
  - br-vxlan
  - br-ext
  - '{{ opsint }}121'
  - '{{ opsint }}122'
  - '{{ opsint }}123'
  - '{{ opsint }}124'
  - '{{ opsint }}l2'
```

Next we go ahead and create each bridge interface and the underlying worker interface it uses:

```yaml
- name: Add a br-mgmt interface
  command:
    cmd: 'nmcli connection add type bridge con-name br-mgmt ifname br-mgmt autoconnect yes save yes ipv4.method manual ip4 10.0.21.{{ ip }}/24 gw4 10.0.21.1 ipv4.dns 10.0.1.17'
- name: Add vlan interface to br-mgmt interface
  command:
    cmd: 'nmcli connection add type vlan con-name {{ opsint }}121 ifname {{ opsint }}.121 dev {{ opsint }} id 121 master br-mgmt slave-type bridge autoconnect yes save yes'

# This format repeats for br-vxlan, br-storage and br-ext

# For the br-vlan interface we don't need L3 setup

- name: Add a br-vlan interface
  command:
    cmd: 'nmcli connection add type bridge con-name br-vlan ifname br-vlan autoconnect yes save yes ipv4.method disabled ipv6.method ignore'
- name: Add worker int to br-vlan
  command:
    cmd: 'nmcli connection add type bridge-slave con-name {{ opsint }}l2 ifname {{ opsint }} master br-vlan'
```

How ugly is that right?

Finally we can go ahead and restart `NetworkManager` for giggles:

```yaml
- name: Restart NetworkManager
  command:
    cmd: systemctl restart NetworkManager.service
```

Now our interfaces are ready to go.

{% endraw %}

## Storage Configuration

The ["Prepare the target hosts"](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/targethosts.html) page includes a little section on configuring storage, however it isn't extensive in the slightest. It only covers setting up LVM in case you're using it as a backend for cinder and container file systems. For ceph clusters and swift, I had to dig a bit deeper to figure out disk configuration requirements.

### Ceph Devices

For my deployment I created a ceph cluster as a backing volume store for Cinder. I found out through the [ceph-ansible](https://docs.ceph.com/ceph-ansible/master/osds/scenarios.html) documentation that any devices used for ceph, by default, need to be "clean" and therefore not contain a gpt partition table. We can use the [shell](https://docs.ansible.com/ansible/latest/modules/shell_module.html) module to perform a bit of `gdisk` magic:

```yaml
- name: Run gdisk to wipe /dev/sdb
  shell:
    cmd: "gdisk /dev/sdb"
    stdin: |
      x
      z
      Y
      Y
    stdin_add_newline: yes
    register: gdisk_out
    failed_when: "'Blank out MBR? (Y/N): ' not in gdisk_out.stdout"
```

Note that this will assume that `/dev/sdb` is not corrupt. If `/dev/sdb` is corrupted, gdisk will add an additional, unexpected prompt that will cause this task to fail. If you are expecting your disks to be corrupted, then I'd modify the `stdin` or use the [expect](https://docs.ansible.com/ansible/latest/modules/expect_module.html) module.


### Swift Devices

Swift requires that you have it's storage disks mounted and ready to go before the playbook is run, otherwise it will fail. On each of my nodes I use the disks `/dev/sdc` and `/dev/sdd` for Swift and configure them based on this piece of documentation [here](https://docs.openstack.org/openstack-ansible-os_swift/latest/configure-swift-devices.html):

```yaml
- name: Create xfs on sdc
  filesystem:
    fstype: xfs
    dev: /dev/sdc
    # Adding this option makes this task idempotent 
    force: no
- name: Create mount location /srv/node/sdc
  file:
    state: directory
    path: /srv/node/sdc
- name: Mount sdc to /srv/node/sdc
  mount:
    path: /srv/node/sdc
    src: /dev/sdc
    fstype: xfs
    state: mounted
    opts: "noatime,nodiratime,nobarrier,logbufs=8,auto"
    dump: '0'
    passno: '0'
- name: Create xfs on sdd
  filesystem:
    fstype: xfs
    dev: /dev/sdd
    force: no
- name: Create mount location /srv/node/sdd
  file:
    state: directory
    path: /srv/node/sdd
- name: Mount sdd to /srv/node/sdd
  mount:
    path: /srv/node/sdd
    src: /dev/sdd
    fstype: xfs
    state: mounted
    opts: "noatime,nodiratime,nobarrier,logbufs=8,auto"
    dump: '0'
    passno: '0'
```

Of course these blocks could be condensed using some ansible `with_items` magic or some kind of loop, but I had a good time writing it all out to practice using these modules.

## In Conclusion

Taking the time to automate seemingly one-off tasks can make for quick bootstrapping of new nodes in the case of failure or scaling up. And with Ansible's really dynamic and powerful pre-built set of modules, it's super easy to do. If you have any suggestions or possible additions, please feel free to reach out to me and I'd love to talk about it.
