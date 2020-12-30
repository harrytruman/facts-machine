# Facts Machine: Network Facts, Configs, and Backups for Ansible/Tower
-------------

The Facts Machine is a role gathers Ansible Facts, sets custom facts, and parses command output to turn device configurations into code. Once gathered, facts can be used as backups/restores, or called later as variables in other roles or playbooks. And most importantly, they can be used to build the framework of a network CMDB.

This role will pass through the `ansible_network_os` inventory variable to a series of playbooks based on that device OS. Currently, this role will gather facts and perform backups from the following platforms:

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

### Role Variables
--------------

The `ansible_network_os` variable defines our inventory hosts' OS. We use this to decide which tasks and templates to run, since each OS can have OS specific commands. This should ideally be coming from an inventory or group variable.

At a minimum, Ansible needs these inventory details:
```
ansible_hostname     hostname_fqdn
ansible_network_os   ios/nxos/etc
ansible_username     username
ansible_password     password
```

Your initial inventory file can be setup like this:

```
[all]
hostname_fqdn  ansible_network_os=ios  ansible_username=<username>  ansible_password=<password>
hostname_fqdn  ansible_network_os=nxos  ansible_username=<username>  ansible_password=<password>
```

## Ansible Network Fact Collection

Ansible's native fact gathering can be invoked by setting `gather_facts: true` in your top level playbook. And every major networking vendor has fact modules that you can use in a playbook task: `ios_facts`, `eos_facts`, `nxos_facts`, `junos_facts`, etc...

Here's an example of gathering facts on a Cisco IOS device. This will create a backup of the full running config, and parse config subsets into a platform-agnostic data model:

```
- name: collect device facts and running configs
  hosts: all
  gather_facts: yes
  connection: network_cli

  tasks:
  - name: gather ios facts
    ios_facts:
      gather_subset: all
```

## Setting Custom Ansible Facts

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


## Ansible at Scale

### How Ansible Works: Network vs OS

It’s important to differentiate that Ansible/Tower operates somewhat differently when configuring network and devices, compared to when performing traditional OS management. When Ansible runs against a proper OS like Linux/Windows, the remote hosts have Python/Bash, and they both receive commands and process their own data and state changes. As an example with a Linux host, a standard logging service configuration playbook would be fully executed on the remote host; upon completion, only task results are sent back to Ansible.

Network devices, on the other hand, rarely perform their own data processing. Until quite recently, very few network devices were built to have APIs -- much less Python. This presents a problem for any external configuration or management system. Things like SNMP address some parts of this problem by allowing some aspects of configuration and device state to be set or polled, but the vast majority of networks are managed via good ol' fashioned screen scrapes and command orchestration scripts.

Ansible helps solve the problem of communicating with every device on your network. But even though this is the 21st century, network command orchestration is still accomplished primarily by sending commands to devices and having those devices return output back to Ansible. Over and over. Rather than being able to rely on remote devices to do their own work, Ansible handles all network data processing as it’s received from remote devices. For most network environments, all data processing will to be performed locally on Ansible/Tower. 

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

### Network Facts: Speed and Performance

Get into a habit of routinely checking your playbook runtime. Basline peformance testing is your friend.

```
#ansible.cfg
callback_whitelist = profile_tasks, timer
```

That said, there are some general principles and guidelines. To start, IOS and EOS are the fastest, XR and NXOS are the slowest. For a better example, here are some base numbers from my individual peformance testing results. I captured simple job run times from an array of fact collection and device configuration roles:

##### Facts - Single Host

Running fact collection against a single host - time in seconds:
```
IOS:        3
XE:         3
AireOS:     5
F5:         5
CiscoWLAN:  6
Fortinet:   6
EOS:        8
XR:         10
NXOS:       12
```

From the results at the time of testing, doing `show run` to create a config fact/backup takes 2-3 seconds for a single, modern IOS/XE device. That will vary, however, depending on the complexity of the config and the firmware versions. Older firmware versions perform far slower (5-10 seconds). And other network families differ entirely. Compared to IOS completing a simple config backup in 2-3 seconds, NXOS takes 10-15 seconds, IOS-XR takes 8-12 seconds, EOS takes 6-10 seconds, etc…


##### Facts - Large Inventory Groups

Running fact collection against large inventory groups - time in minutes:

Job inventories were broken down into groups of 500 hosts, 100 forks
```
IOS:    4m 8sec
XR:     4m 25sec
NXOS:   15m 35sec
EOS:    8m 9sec
---
Full:   2h 3m 15sec
```

With a full inventory size of 15k+ devices, (10k ios, 3k nxos, 700 xr, 500 eos, etc...), a full fact collection run took just over two hours.


## Storing and Using Facts

Tower is not ideal for collecting, storing, and retrieving facts in environments with large inventories. Simply put, processing all of these local facts will put a tremendous strain on Tower. Additionally, any CMDB and Source of Truth should be implemented external to Tower.

Use something like an ELK cluster to store Tower logs and Ansible Facts:

https://github.com/harrytruman/elk-ansible

