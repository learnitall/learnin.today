---
title: "I Tried OpenStack Ansible and Had a Bad Time. Here's Why."
date: 2020-07-08
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

> This post will be continually updated as I submit more bug reports and change requests. Last updated on: July 10th, 2020

The goal of this post is not to be toxic or hurtful in any way- I have a ton of respect for the OpenStack team and all the work they do to put together such a large project all in the name of open and free software. But damn, my experience with OpenStack Ansible was a nightmare.

I wanted to spend a week this summer putting together an OpenStack cloud so I can have my own personal environment for practicing Red Team and Blue Team operations. Three weeks later, my cluster is still broken, buggy and my patience has run out.

OpenStack seems so awesome and "sexy", but can be a pain to manage and install. [stratoscale](https://www.stratoscale.com/), a major competitor in the cloud industry, published a report entitled [2019 Escaping OpenStack Survey Report](https://www.stratoscale.com/resources/ebooks/2019-escaping-openstack-survey-report/) which finds:

* "88% of companies reported having teams of over 20 people dedicated to their OpenStack deployments"
* "47% of them reported needing more than 60 people"

(These quotes were lifted from a blog post stratoscale published on OpenStack alternatives. It can be found at: [https://www.stratoscale.com/blog/openstack/openstack-alternatives/](https://www.stratoscale.com/blog/openstack/openstack-alternatives/).)

Unfortunately, after working with OpenStack Ansible I can say that these statistics don't surprise me at all. OpenStack Ansible has such amazing potential to ease the management overhead of an OpenStack cluster, but the number of manual source code changes I had to go through to get this thing to not even work was unreal. In light of letting go of OpenStack Ansible for the time being, here are all the changes, which I can at least remember, that I had to make to OpenStack Ansible to actually get some momentum. For reference, I'm rocking a CentOS 7 Train deployment.

## Inventory Generation

> Relevant File: [osa_toolkit/generate.py](https://opendev.org/openstack/openstack-ansible/src/commit/8760541cedd7bda736449007accf235f2b5fea23/osa_toolkit/generate.py), line 374 and 386.
>
> Bug report on launchpad: [https://bugs.launchpad.net/openstack-ansible/+bug/1886905](https://bugs.launchpad.net/openstack-ansible/+bug/1886905).
>
> Change review on opendev: [https://review.opendev.org/#/c/740343/](https://review.opendev.org/#/c/740343/).

When choosing the names for my hosts under `openstack_user_config.yml`, I used the FQDN for each of my hosts. Apprently, there is a max length limit on these names that the dynamic inventory script does a sanity check for before playbook execution. There is a classic `KeyError` bug however that impacts this sanity check, where if you use too long of a hostname you'll get a `KeyError` rather than the more appropriate `SystemExit`. Here's why:

```python
"""
osa_toolkit/generate.py
Starting at line 373
"""

if len(type_and_name) > max_hostname_len and \
        not properties['is_metal']:
    raise SystemExit(
        'The resulting combination of [ "{}" + "{}" ] is longer than'
        ' {} characters. This combination will result in a container'
        ' name that is longer than the maximum allowable hostname of'
        ' 63 characters. Before this process can continue please'
        ' adjust the host entries in your "openstack_user_config.yml"'
        ' to use a short hostname. The recommended hostname length is'
        ' < 20 characters long.'.format(
            host_type, container_name, max_hostname_len
        )
    )
elif len(host_type) > 63 and properties['is_metal']:
    raise SystemExit(
        'The resulting hostname "{0}" is longer than 63 characters.'
        ' This combination may result in a name that is longer than'
        ' the maximum allowable hostname of 63 characters. Before'
        ' this process can continue please adjust the host entries'
        ' in your "openstack_user_config.yml" to use a short hostname'
        '.'.format(host_type)
    )
```

Do you see why a `KeyError` can be raised? This, I think, is the only place in this script where bracket notation is used to access the `is_metal` property of the `properties` `dict`. Everywhere else it's a:

```python
properties.get('is_metal', False)
```

Classic example of the pitfalls of short-circuit logic and why rigerous unit testing is so helpful.

## HA Deployment

I have three, old retired Dell Poweredges and they are actively dying, therefore I wanted to have my cluster cope with any one of them failing. Additionally, I wanted to have a responsive OpenStack cluster, so the more load that could be distributed the better. There were quite a few issues with getting a HA cluster setup though.

### HAProxy Galera Backend Configuration

> Change review on opendev: [https://review.opendev.org/#/c/740355/](https://review.opendev.org/#/c/740355/)

Openstack Ansible will setup a [galera](https://galeracluster.com/) cluster with one instance of [mariadb](https://mariadb.org/) running on each of your shared infrastructure hosts. The amazingly magic aspect to galera is it allows for read/write operations on any node in your cluster, so when using [haproxy](https://www.haproxy.org/) as a loadbalancer, galera recommends a configuration similar to this sample:

```
# Load Balancing for Galera Cluster
listen galera 192.168.1.10:3306
     balance source
     mode tcp
     option tcpka
     option mysql-check user haproxy
     server node1 192.168.1.1:3306 check weight 1
     server node2 192.168.1.2:3306 check weight 1
     server node2 192.168.1.3:3306 check weight 1
```

> See [https://galeracluster.com/library/documentation/ha-proxy.html](https://galeracluster.com/library/documentation/ha-proxy.html)

This configuration will essentially cause haproxy to distribute any requests it gets for the galera cluster to each of the galera nodes, distributing load pretty nicely. Unfortunately however, this is the configuration that Openstack Ansible deploys by default:

```yaml
proxy_default_services:
  - service:
      haproxy_service_name: galera
      haproxy_backend_nodes: "{{ (groups['galera_all'] | default([]))[:1] }}"  # list expected
      haproxy_backup_nodes: "{{ (groups['galera_all'] | default([]))[1:] }}"
```

> See [https://opendev.org/openstack/openstack-ansible/src/commit/6430540a4a0c909719bdddda0152195fbae6dc31/inventory/group_vars/haproxy/haproxy.yml](https://opendev.org/openstack/openstack-ansible/src/commit/6430540a4a0c909719bdddda0152195fbae6dc31/inventory/group_vars/haproxy/haproxy.yml)

This tunnels all galera traffic to the first galera node in the cluster, leaving the other two as just backends in case of failure. The issue this caused for me is that my galera nodes weren't capable of handling all of this traffic independently. This showed itself in three ways:

1. I started to see that the configured max connection limit was being reached and had to manually boost it during troubleshooting. 
2. I starting receiving gateway timeouts during API requests.
3. haproxy wasn't able to perform successful sequential health checks for the single backend galera node being used because the node was essentially being DOSed. 

To fix these issues, I had to change the OpenStack Ansible configuration to the following:

```yaml
proxy_default_services:
  - service:
      haproxy_service_name: galera
      haproxy_backend_nodes: "{{ (groups['galera_all'] | default([])) }}" # list expected
```

One tradeoff of this change however is that after I implemented it in my deployment, I was receiving quite a few deadlock errors from my OpenStack services. Check out the change review I linked above for more discussion on this.

### Syntax Error in Lsyncd Configuration

It is to my understanding that when more than one repository node is used, only one of them is actually selected to build Python wheels. This is setup within the [python_venv_build](https://opendev.org/openstack/ansible-role-python_venv_build) role like so:

* **[vars/main.yml](https://opendev.org/openstack/ansible-role-python_venv_build/src/commit/aabd3c07c29fd89a56cadc01ec3a2e3b682e5e14/vars/main.yml)**: The variable `venv_build_targets` is populated with hosts from the `repo_all` group, sorting them based on the platform their running.
* **[defaults/main.yml](https://opendev.org/openstack/ansible-role-python_venv_build/src/commit/aabd3c07c29fd89a56cadc01ec3a2e3b682e5e14/defaults/main.yml)**: The variable `venv_build_host` is populated with the appropriate host from `venv_build_targets` based on the needed host platform. 
* **[tasks/python_venv_wheel_build.yml](https://opendev.org/openstack/ansible-role-python_venv_build/src/commit/aabd3c07c29fd89a56cadc01ec3a2e3b682e5e14/tasks/python_venv_wheel_build.yml)**: The task "Build the wheels on the build host" is delegated to the host found within the `venv_build_host` variable.  

The problem that this creates though is that if the wheels are built on only one of the repository nodes, then they need to be distributed to all the other repository nodes. OpenStack Ansible ensures this is done by setting up [Lsyncd](https://github.com/axkibe/lsyncd) onto one of the repository nodes, which then uses rsync to sync all of the Python wheels between each repository node.

However, during my deployment I kept receiving issues about invalid package versions and unresolved dependencies for my venvs anyways. Turns out that for [27 days](https://github.com/openstack/openstack-ansible-repo_server/compare/748d86...220a481), a quotation mark was missing from the lsyncd configuration file that caused the service to fail to start on the grounds of a syntax error. I had to manually go in and add this quotation mark in order to get any of my venvs to install.

### Memcached Configuration

Ok this one is totally my fault. In OSA's memcached configuration documentation for **Ussuri**, there is a page on configuring haproxy to act as a loadbalancer for memcached. That page can be found at [this link here](https://docs.openstack.org/openstack-ansible-memcached_server/ussuri/configure-ha.html#configuring-memcached-through-haproxy). I found this page through Google, followed its instructions for my **Train** deployment and had horrible results. This is because those instructions cannot be found within the Train release of OSA's memcached configuration documentation, which can be found at [this link here](https://docs.openstack.org/openstack-ansible-memcached_server/train/). 

To say the least, one problem I was having by load balancing memcached on Train was that Horizon stores cached user sessions in memcached, so depending on which memcached backend server Horizon hits through haproxy will determine if Horizon thinks you're logged in or not. Do I think this may be fixed in Ussuri? Sure. Do I want to find out? Not really.

Ok I think that's all the issues I had related to a HA deployment. Let's move on to storage issues!

## Storage Bugs

In my deployment I wanted a ceph-backed cinder and a swift-backed glance. This was way easier said than done though, and in fact, I never actually got it working. Here's everything I had to do in order to at least get to the point in which my patience ran out.

### Swift Invalid systemctl Service Template

When swift is deployed, the following bit of yaml is used to tell Ansible to deploy the systemd service files for each of the swift services that need to be started up and managed:

```yaml
- name: Run the systemd service role
  import_role:
    name: systemd_service
  vars:
    systemd_user_name: "{{ swift_system_user_name }}"
    systemd_group_name: "{{ swift_system_group_name }}"
    systemd_tempd_prefix: openstack
    systemd_slice_name: swift
    systemd_lock_path: /var/lock/swift
    systemd_CPUAccounting: true
    systemd_BlockIOAccounting: true
    systemd_MemoryAccounting: true
    systemd_TasksAccounting: true
    systemd_services: |-
      {% set services = [] %}
      {% for service in filtered_swift_services %}
      {%
        set _ = service.update(
          {
            'enabled': 'yes',
            'state': 'started',
            'config_overrides': swift_service_defaults | combine(service.init_config_overrides)
          }
        )
      %}
      {%   set _ = service.pop('init_config_overrides') -%}
      {%   set _ = services.append(service) -%}
      {% endfor %}
      {{ services }}
  tags:
    - swift-config
    - systemd-service

```

> See: [https://opendev.org/openstack/openstack-ansible-os_swift/src/commit/abb1c721e50258a0fede787b810e3b17e59d85f7/tasks/main.yml](https://opendev.org/openstack/openstack-ansible-os_swift/src/commit/abb1c721e50258a0fede787b810e3b17e59d85f7/tasks/main.yml)

The important bit to note here is that this will configure each of the deployed swift services to run as the `swift_system_user_name` user, which on my deployment was set to `swift`. Creating a system user that becomes dedicated for running a service and managing its files is a common practice that helps to foster healthy security hygiene, and we all know the world needs as much hygiene as it can get right now. Although this is a good practice, as you may have guessed by now doing this with the swift service itself creates a problem. Check out [swift.common.utils.drop_privileges](https://opendev.org/openstack/swift/src/commit/2e001431fd0907c97d44b81dd1f3d2d14ae1208b/swift/common/utils.py):

```python
def drop_privileges(user):
    """
    Sets the userid/groupid of the current process, get session leader, etc.

    :param user: User name to change privileges to
    """
    if os.geteuid() == 0:
        groups = [g.gr_gid for g in grp.getgrall() if user in g.gr_mem]
        os.setgroups(groups)
    user = pwd.getpwnam(user)
    os.setgid(user[3])
    os.setuid(user[2])
    os.environ['HOME'] = user[5]
```

This function will cause the swift processes to switch to over to their dedicated system user `swift`, however the function doesn't first check if it actually needs to switch over to the `swift` user or if it actually has the permissions to do so. 

Changing the effective user of a process requires root permissions, therefore since the swift services in an OSA deployment are configured to run as `swift`, then the calls to`os.setgid` and `os.setuid` [will always raise](https://stackoverflow.com/questions/7529252/operation-not-permitted-on-using-os-setuid-python) an `OSError` with `Errno 1`. If you want your swift services to actually start, you need to tell systemd to run them as `root` so they can change to the `swift` on their own. This can be done by changing the `systemd_user_name` to `root`, so the above yaml will now look like the following:

```yaml
- name: Run the systemd service role
  import_role:
    name: systemd_service
  vars:
    systemd_user_name: "root"
    systemd_group_name: "{{ swift_system_group_name }}"
    ...
...
```

### Swift Proxy Containers Need openrc File

This one I can't quite remember the exact reasoning for, but I had to add this bit of yaml in the playbook [os-swift-install](https://opendev.org/openstack/openstack-ansible/src/commit/8fa414e1f8e2528c2bcaae706d1a4b893b591c04/playbooks/os-swift-install.yml) right after the task "Gather swift facts":

```yaml
- name: Add openrc file to proxy hosts
  hosts: swift_proxy_container
  gather_facts: true
  user: root
  roles:
    - role: "openstack_openrc"
      tags:
        - openrc
```

If I really stretch my brain, I think I had to add this in there because the swift proxy containers were configuring stuff in keystone and couldn't to the API (check out [os_swift/tasks/service_setup.yml](https://opendev.org/openstack/openstack-ansible-os_swift/src/commit/abb1c721e50258a0fede787b810e3b17e59d85f7/tasks/service_setup.yml)). Other openstack services that do keystone configuration stuff had this bit of yaml in their playbooks, so I added into swift's for the proxy containers and lo-and-behold, the playbook runs successfully.


### Cinder oslo SSL Errors

> Bug on launchpad: [https://bugs.launchpad.net/cinder/+bug/1885616](https://bugs.launchpad.net/cinder/+bug/1885616) 

This one I spent a couple days on and could never quite figure out. Essentially my cinder services would refused to connect to my [rabbitmq](https://www.rabbitmq.com/) cluster over SSL, failing with a `ssl.SSLError: [SSL: UNABLE_TO_LOAD_SSL2_MD5_ROUTINES] unknown error (_ssl.c:2830)`. In fact, there are so few Google results for the string `UNABLE_TO_LOAD_SSL2_MD5_ROUTINES`, that I bet a week after this post goes up you'll be able to find it by searching Google for that error. From what I could tell, this error can pop up if two non-thread-safe SSL contexts are created one after another. My assumption would be that this error has to do with the client cinder uses to connect to rabbit, but weirdly enough I only saw these kinds of logs on my cinder-volumes nodes. Eventually I had to disable SSL in my oslo config to get anywhere.

### Ceph Deployment Missing Dependencies

The changes I had to make to get this playbook to work are a little fuzzy because I made them literally a month ago, but the changes I specifically remember having to make have to do with package management. Boy oh boy what a time.

First off, there's a bit of yaml in [playbooks/comon-tasks/ceph-server.yml](https://opendev.org/openstack/openstack-ansible/src/commit/8fa414e1f8e2528c2bcaae706d1a4b893b591c04/playbooks/common-tasks/ceph-server.yml) that tries to install [PyYAML](https://pypi.org/project/PyYAML/) for python3:

```yaml
- block:
    - name: Install python3-yaml
      package:
        name: "{{ (ansible_os_family | lower == 'debian') | ternary('python3-yaml', 'python3-pyyaml') }}"
        state: present
  # Rescue is mainly for CentOS 7
  rescue:
    # Installing both pip's not to fail
    - name: Installing pip
      package:
        name:
          - python-pip
          - python3-pip
        state: present

    - name: Install PyYAML
      pip:
        name: PyYAML
``` 

On CentOS this will:

1. Try to instal the `python3-pyyaml` package, which will fail
2. Try to install the `python-pip` and `python3-pip` package, which will fail
3. Task fails and the world explodes

The `python-pip` and `python3-pip` packages are provided through EPEL (see [https://linuxize.com/post/how-to-install-pip-on-centos-7/](https://linuxize.com/post/how-to-install-pip-on-centos-7/)), and for some reason in the containers that this playbook targets, EPEL isn't installed or configured correctly so it isn't used. In order to install the PyYAML module, I modified the playbook to use the `get-pip.py` script, which actually worked pretty well.

```yaml
# Rescue is mainly for CentOS 7
rescue:
  # Installing both pip's not to fail
  - name: Downloading get_pip.py
    get_url:
      url: https://bootstrap.pypa.io/get-pip.py
      dest: /opt/get-pip.py
      mode: 0755
  - name: Install pip
    command: /usr/bin/python /opt/get-pip.py
  - name: Install PyYAML
    pip:
      name: PyYAML
```

Additionally, on all of my ceph-osd nodes, I needed to manually install the following packages as `root` using `pip3`:

* [pecan](https://www.pecanpy.org/)
* [cherrypy](https://cherrypy.org/)
* [werkzueg](https://pypi.org/project/Werkzeug/)

This is because whenver I ran a `ceph status`, ceph would complain with a health warning and said these packages were missing.

Ok, that's it for storage bugs! Here are some run miscellaneous bugs that I also discovered.

## Neutron Missing Dependencies

While running the [os-neutron-install](https://opendev.org/openstack/openstack-ansible/src/commit/8fa414e1f8e2528c2bcaae706d1a4b893b591c04/playbooks/os-neutron-install.yml) playbook, I kept receiving an error that `gcc` was missing on my target nodes, so I had to install `@Development Tools` on each of them. Additionally, I can't quite remember which playbook this is relevant to but I'm faily confident it has to do with neutron, at some point I had to load the `br-netfilter` kernel module on all my target nodes.

## Zun Playbook Typos

Alright. I saved the best for last. The issues with the Zun playbook hurt the most because they're so stupid, yet so impactful, I can't believe no one else has caught them.

Check out the first two tasks in the [os-zun-install](https://opendev.org/openstack/openstack-ansible/src/commit/8fa414e1f8e2528c2bcaae706d1a4b893b591c04/playbooks/os-zun-install.yml) playbook:

```yaml
- name: Gather zun facts
  hosts: zun
  gather_facts: "{{ osa_gather_facts | default(True) }}"
  tags:
    - always

- name: Install the zun components
  hosts: zun_all
  gather_facts: false
  user: root
  vars_files:
  ...
```

The `Gather zun facts` task is applied to the host group `zun`, which doesn't exist. Luckily the task **right below it** called `Install the zun components` is applied to the right group entitled `zun_all`, so before running this playbook please do yourself a favor and replace the non-existent `zun` host group with the existant `zun_all` host group.

Here's another case of bad host group names. In [tasks/zun_pre_flight.yml](https://opendev.org/openstack/openstack-ansible-os_zun/src/branch/master/tasks/zun_pre_flight.yml):

```yaml
- name: Check for oslomsg_rpc_all group
  fail:
    msg: >-
      The group `oslomsg_rpc_all` is undefined. Before moving forward
      set this group within inventory with at least one host.
  when:
    - (groups['oslomsg_rpc_all'] | length) < 1
```

It's too bad the `oslomsg_rpc_all` group doesn't exist anymore.

```
[root@osadep os_zun]# grep -i 'oslomsg_rpc_all' /etc/openstack_deploy/openstack_inventory.json 
[root@osadep os_zun]#
```

Here's the bit of yaml I wrote to do the same sanity check:

```yaml
- name: Check for oslomsg-rpc_hosts group or rabbitmq_all group
   fail:
     msg: >-
       The groups `oslomsg-rpc_hosts` and `rabbitmq_all` are undefined. Before
       moving forward set these groups within inventory with at least one host.
   when:
     - (groups['oslomsg-rpc_hosts'] | length) < 1
     - (groups['rabbitmq_all'] | length) < 1
```

I'm pretty sure this `when` conditional needed a logical AND on those two conditions, but I unfortunately I can't remember why. Next time I'll comment my code better. At least the check actually passes, so works for me.

Ok now check this one out- this one is my favorite in the whole post. Tell me what's wrong with the following snippet of this configuration file [os_zun/defaults/main.yml](https://opendev.org/openstack/openstack-ansible-os_zun/src/commit/bc3de56005fb9a69a5cadb2313f3a8518d857bb0/defaults/main.yml):

```yaml
zun_service_publicuri: "{{ zun_service_publicuri_proto }}://{{ external_lb_vip_address }}:{{ zun_service_port }}"
zun_service_publicurl: "{{ zun_service_publicuri }}"
zun_service_adminuri: "{{ zun_service_adminuri_proto }}//{{ internal_lb_vip_address }}:{{ zun_service_port }}"
zun_service_adminurl: "{{ zun_service_adminuri }}"
zun_service_internaluri: "{{ zun_service_internaluri_proto }}://{{ internal_lb_vip_address }}:{{ zun_service_port }}"
zun_service_internalurl: "{{ zun_service_internaluri }}"
```

Can you see it? What if I told you each of these variables is a URL constructed as "protocol://ip:port"? Can you see now that the `zun_service_adminuri` is missing a damn colon after the protocl?

Oh and I sh\*t you not, it's been missing that colon for over two years. Look at this comparison UI that shows a diff between the first commit in the [openstack-ansible-os_zun](https://opendev.org/openstack/openstack-ansible-os_zun) repository to the last commit in the master branch (look for line 158 in the file `defaults/main.yml`): [https://github.com/openstack/openstack-ansible-os_zun/compare/daf9f9d60a00edf5c874a35b621acc7d0e5a8e06..master](https://github.com/openstack/openstack-ansible-os_zun/compare/daf9f9d60a00edf5c874a35b621acc7d0e5a8e06..master). I could not stop laughing for so long, made my day.

## Takeaway

Part of me is being a bit dramatic here just due to my frustration, but if you really want to use OpenStack Ansible then go for it. I ran out of patience with it and want to try something different that'll be eaiser to maintain for a development lab (`<cough>` [kolla-ansible](https://docs.openstack.org/kolla-ansible/latest/index.html), [K8s](https://kubernetes.io/), [Apache CloudStack](https://cloudstack.apache.org/downloads.html) `</cough>`), but if you want to still give it a chance here are my recommendations:

 * There are three main playbooks that you run sequentially during the deployment entitled: `setup-hosts.yml`, `setup-infrastructure.yml` and `setup-openstack.yml`. The only thing these playbooks do is import other playbooks, so please **don't use them.** Instead, open up each of these "setup" playbooks and run the playbooks they include one by one. After each playbook you run, look through your journal logs and verify the operation of the thing you just deployed. You'll thank me later.
 * Use the legacy rsyslog rather than the new remote journal service. I had issue with the remote journal service not sending logs fast enough to actually be useful in real time debugging. Could've just been my own fault, but there you go.
 * Always run with a `-vvv` and pipe your output to a log file. If a playbook fails, you don't have to run it again to get valuable debugging information or scroll up in your console like crazy.
 * Check out the [deubg](https://docs.ansible.com/ansible/latest/modules/debug_module.html) module and the [playbook debugger](https://docs.ansible.com/ansible/latest/user_guide/playbooks_debugger.html). Very helpful in figuring out what's going on.
 * Invest in a stress ball and tell your SO that you love them.

OpenStack Ansible has given me the wondeful gift of questioning my own sanity. Am I going crazy? I don't think so, but why hasn't anyone else noticed most of these bugs before? Are they only impacting me? What's the deal?

If you've have experience working with OpenStack Ansible or OpenStack in general, please reach out to me and let's chat.
