# Facts Machine: Network Fact Collection for Ansible/Tower
-------------

The Facts Machine is a role gathers Ansible Facts, sets custom facts, and parses command output to find useful config bits about network devices.

Once gathered, Ansible Facts can be used as backups/restores, they can be called later in other roles or playbooks, and they can be used to build the framework of a network CMDB.

## Setting Custom Ansible Facts

Any command output can be parsed and set as a fact. Here's an example of how to set a custom fact for Cisco IOS versions. This will run `show version`, strip out everything except the firmware version info, and save it as the variable `cisco-ios-version`.

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

Setting custom facts works particularly well for device info that isn't gathered by default through native Ansible fact collection modules. For instance, F5 natively gathers the attached license, but not much else. So you may want to find and set a fact for custom F5 license info.

This will run one command (`show sys license`) and set two facts: one for when the device was licensed, and another for the service check date. 

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


### Role Variables
--------------

The `cli` variable holds the credentials and transport type to connect to the device.
The `device_os` variable is the type of device OS that the current host is. We use this to decide which tasks and templates to run since each OS can have OS specific commands. This should be coming from a group var.


## Ansible at Scale

### How Ansible Works: Network vs OS

It’s important to differentiate that Ansible/Tower operates somewhat differently when configuring network and devices, compared to when performing traditional OS management. When Ansible runs against a proper OS like Linux/Windows, the remote hosts have Python/Bash, and they both receive commands and process their own data and state changes. As an example with a Linux host, a standard logging service configuration playbook would be fully executed on the remote host; upon completion, only task results are sent back to Ansible.

Network devices, on the other hand, rarely perform their own data processing. Until quite recently, very few network devices were built to have APIs -- much less Python. This presents a problem for any external configuration or management system. Things like SNMP address some parts of this problem by allowing some aspects of configuration and device state to be set or polled, but the vast majority of networks are managed via good ol' fashioned screen scrapes and command orchestration scripts.

Ansible helps solve the problem of communicating with every device on your network. But even though this is the 21st century, network command orchestration is still accomplished primarily by sending commands to devices and having those devices return output back to Ansible. Over and over. Rather than being able to rely on remote devices to do their own work, Ansible handles all network data processing as it’s received from remote devices. For most network environments, all data processing will to be performed locally on Ansible/Tower. 

#### Fact Collection at Scale

Ansible helps solve the problem of communicating with every device on your network. But even though this is the 21st century, network command orchestration is still accomplished primarily by sending commands to devices and having those devices return output back to Ansible. Over and over. Rather than being able to rely on remote devices to do their own work, Ansible handles all network data processing as it’s received from remote devices. For most network environments, all data processing will to be performed locally on Ansible/Tower. 

In the pursuit of scaling Ansible/Tower to manage large network device inventories, we must consider a number of factors that will directly impact job performance:

  1. Frequency and extent of orchestrating/scheduling device changes
  2. Device configuration size (raw text output from `show run`, etc..)
  3. Inventory sizes and devices families, e.g. IOS, NXOS, XR
  4. Implementation and availability of Ansible network facts modules, parsers, and fact caching

### Storing and Using Facts

Tower is not ideal for storing/retrieving facts in large scale environments. Simply put, processing all of these local facts will tremendously slow down Tower. Additionally, any CMDB and Source of Truth should be implemented external to Tower.

Because of this, I use an ELK cluster to store Tower logs and Ansible Facts:
https://github.com/harrytruman/elk-ansible
