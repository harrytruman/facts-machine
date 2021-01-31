# Facts Machine

## Network Facts, Configs, and Backups for Ansible

The Facts Machine parses network configs into a data model. This role gathers native Ansible Facts or sets custom facts to parse command output and convert device configurations into code.

Once gathered, facts can be used as backups/restores, called later as variables in other roles or playbooks, and used to define the state of a device. And most importantly, facts are used to build the framework of a network CMDB!

This role will is compatible with the following platforms:

```
eos
ios
iosxr
nxos
aruba
aireos
f5-os
fortimgr
junos
paloalto
vyos
```

--------------

### Role Variables

Ansible uses the `ansible_network_os` inventory variable to define the device OS. This should ideally be coming from an inventory or group variable.

At a minimum, Ansible needs these details to run against hosts:
```
ansible_hostname     hostname_fqdn
ansible_network_os   ios/nxos/etc
ansible_username     username
ansible_password     password
```

### Inventory

I *highly* recommend [vaulting your passwords/keys/creds](https://docs.ansible.com/ansible/latest/user_guide/vault.html#creating-encrypted-variables) instead of storing them plaintext! My inventories usually start like this:

```
[all]
ios-dc1-rtr
ios-dc2-rtr
ios-dc1-swt
ios-dc2-swt
nxos-dc1-rtr
nxos-dc2-rtr

[all:vars]
ansible_connection=network_cli
ansible_user=admin
ansible_password: !vault |
       $ANSIBLE_VAULT;1.2;AES256;ansible_user
       66386134653765386232383236303063623663343437643766386435663632343266393064373933
       3661666132363339303639353538316662616638356631650a316338316663666439383138353032
       63393934343937373637306162366265383461316334383132626462656463363630613832313562
       3837646266663835640a313164343535316666653031353763613037656362613535633538386539
       65656439626166666363323435613131643066353762333232326232323565376635

[ios]
ios-dc1-rtr
ios-dc2-rtr
ios-dc1-swt
ios-dc2-swt

[ios:vars]
ansible_become=yes
ansible_become_method=enable
ansible_network_os=ios

[nxos]
nxos-dc1-rtr
nxos-dc2-rtr

[nxos:vars]
ansible_become=yes
ansible_become_method=enable
ansible_network_os=nxos
```

--------------

## Ansible Network Fact Collection

Ansible's native fact gathering can be invoked by setting `gather_facts: true` in your top level playbook. And every major networking vendor has fact modules that you can use in a playbook task: `ios_facts`, `eos_facts`, `nxos_facts`, `junos_facts`, etc...

Just enable `gather_facts`, and you're on your way! Here's an example of gathering facts on a Cisco IOS device to create a backup of the full running config, and parse config subsets into a platform-agnostic data model:

```
- name: collect device facts and running configs
  hosts: all
  gather_facts: yes
  connection: network_cli
```

Or at the task level:

```
  tasks:
  - name: gather ios facts
    ios_facts:
      gather_subset: all
```

Either way, this is how you start down the path to true Config-to-Code!

```
ansible_facts:
  ansible_net_fqdn: ios-dc2-rtr.lab.vault112
  ansible_net_gather_subset:
  - interfaces
  ansible_net_hostname: ios-dc2-rtr
  ansible_net_serialnum: X11G14CLASSIFIED...
  ansible_net_system: nxos
  ansible_net_model: 93180yc-ex
  ansible_net_version: 14.22.0F
  ansible_network_resources:
    interfaces:
    - name: Ethernet1/1
      enabled: true
      mode: trunk
    - name: Ethernet1/2
      enabled: false
    ...
 ```

#### Fact Caching

Ansible Facts can be cached too! Options include local file, memcached, Redis, and a plethora of others, via Ansible's [Cache Plugins](https://docs.ansible.com/ansible/latest/plugins/cache.html). And caching can be enabled with just the click button [in AWX and Tower](https://docs.ansible.com/ansible-tower/latest/html/userguide/job_templates.html#benefits-of-fact-caching), where you can then view facts via UI and API both.

![fact cache](https://docs.ansible.com/ansible-tower/latest/html/userguide/_images/job-templates-options-use-factcache.png)

The combination of using network facts and fact caching can allow you to poll existing, in-memory data rather than parsing numerous additional commands to constantly check/refresh the device's running config.

### Making Custom Ansible Facts

You can also run custom commands, save the output, and parse the configuration later. Any command output can be parsed and set as a fact!

Here's an example of how to set a custom fact for Cisco IOS versions. This will run `show version`, find the version information, and save it as the variable `cisco-ios-version`.

```
- name: run `show version` command
  ios_command:
    commands:
      - show version
  register: output

- name: set version fact
  set_fact:
    cisco-ios-version: "{{ output.stdout[0] | regex_search('Version (\\S+)', '\\1') | first }}"
```

Setting custom facts works particularly well for building out infrastructure checks/verifications. For instance, F5 natively gathers the attached license, but you can identify additional content that will help you automate expiration/renewal processes. As an example, this will run one command (`show sys license`) and set two facts: one for when the device was licensed, and another for the service check date:

```
- name: get license information - {{ inventory_hostname }}
  bigip_command:
    commands:
      - show sys license
  register: license_output

- name: search for `licensed on` and `service check date`
  set_fact:
    licensed_on: "{{ license_output.stdout[0] | regex_search('Licensed On (.*)') }}"
    service_date: "{{ license_output.stdout[0] | regex_search('Service Check Date (.*)') }}"

- name: Get Licensed On and Service Check Date
  set_fact:
    licensed_on: "{{ licensed_on.split(' ') | last }}"
    service_date: "{{ service_date.split(' ') | last }}"

- debug:
    msg:
      - "Licensed On date is {{ licensed_on }}"
      - "Service Check Date is {{ service_date }}"
```

--------------

## Ansible at Scale

### How Ansible Works: Network vs OS

It’s important to differentiate that Ansible/Tower operates somewhat differently when configuring network and devices, compared to when performing traditional OS management. When Ansible runs against a proper OS like Linux/Windows, the remote hosts have Python/Bash, and they both receive commands and process their own data and state changes. As an example with a Linux host, a standard logging service configuration playbook would be fully executed on the remote host; upon completion, only task results are sent back to Ansible.

Network devices, on the other hand, rarely perform their own data processing. Until quite recently, very few network devices were built to have APIs -- much less Python. This presents a problem for any external configuration or management system. Things like SNMP address some parts of this problem by allowing some aspects of configuration and device state to be set or polled, but the vast majority of networks are managed via good ol' fashioned screen scrapes and command orchestration scripts.

--------------

### Network Facts: Speed and Performance

Get into a habit of routinely checking your playbook runtime. Basline peformance testing is your friend:

```
#ansible.cfg
callback_whitelist = profile_tasks, timer
```

Beyond that, here are some general principles and guidelines. To start, IOS and EOS are the fastest, XR and NXOS are the slowest. For a better example, here are some base numbers from my individual peformance testing results. I captured simple job run times from an array of fact collection and device configuration roles:

##### Facts - Single Host

Running fact collection against a single host - time in seconds:
```
IOS:        2
XE:         2
AireOS:     4
F5:         4
CiscoWLAN:  5
Fortinet:   5
EOS:        7
XR:         9
NXOS:       10
```

Fact collection takes 2-3 seconds for a single, modern IOS/XE device. That will vary, however, depending on the complexity of the config and the firmware versions. Older firmware versions perform far slower (5-10 seconds).

And other network families differ entirely. Compared to IOS completing a simple `show run all` in 2-3 seconds, NXOS can 10-15 seconds, IOS-XR often 8-12 seconds, EOS around 6-10 seconds, etc…

--------------

### Fact Collection at Scale

Ansible helps solve the problem of communicating with every device on your network. But even though this is the 21st century, network command orchestration is still accomplished primarily by sending commands to devices and having those devices return output back to Ansible. Over and over. Rather than being able to rely on remote devices to do their own work, Ansible handles all network data processing as it’s received from remote devices. For most network environments, all data processing will to be performed locally on Ansible/Tower. 

In the pursuit of scaling Ansible/Tower to manage large network device inventories, we must consider a number of factors that will directly impact job performance:

  1. Frequency and extent of orchestrating/scheduling device changes
  2. Device configuration size (raw text output from `show run`, etc..)
  3. Inventory sizes and devices families, e.g. IOS, NXOS, XR
  4. Ansible network facts modules, parsers, and fact caching

##### Frequency and extent of orchestrating/scheduling device changes
With any large inventory, there comes a balancing act between scheduling configuration changes and avoiding resource contention. At a high level, this can be as simple as benchmarking job run times with Tower resource loads, and setting job template forks accordingly. When creating new network automation roles, it’s important to establish solid development practices to avoid potentially significant processing times.

##### Device configuration size
Most network automation roles will be utilizing Ansible facts derived from device configs. By looking at the raw device config sizes, such as the text output from `show run`, we can establish a rough estimate of memory usage per-host during large jobs.

##### Inventory sizes and devices families, e.g. IOS, NXOS, XR
Due to the large inventory size and the likelihood of significant inventory metadata, it’s critical to ensure that inventories are broken into smaller groups -- group sizes of 5,000 or less are highly recommended. Additionally, it’s important to note that device types/families perform noticeably faster/slower than others. IOS, for instance, is often 3-4 faster than NXOS.

##### Implementation and availability of Ansible network facts
Ansible can collect device facts -- useful variables about remote hosts that can be used in playbooks. Additionally, these facts can be cached in Tower. The combination of using network facts with the fact cache can significantly increase Tower job speed and reduce processing loads.

--------------

## Storing and Using Facts

Unless your clusters have significant free resources to spare, Tower is not ideal for collecting, storing, *and* retrieving facts in environments with constant jobs running against large inventories. Simply put, processing all of these local facts **while** running a full cluster will put a tremendous strain on Tower.

Any CMDB and Source of Truth should be implemented external to Tower. I use something like an ELK cluster to store Tower logs and Ansible Facts:

https://github.com/harrytruman/elk-ansible
