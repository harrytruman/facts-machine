# Facts Machine
=========

## Network Facts
-------------

This role gathers Ansible Facts, sets custom facts, and parses command output to find useful config bits about network devices. Once gathered, Ansible Facts can be used as backups/restores, they can be called later in other roles or playbooks, and they can be used to build the framework of a network CMDB.

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


###Role Variables
--------------

The `cli` variable holds the credentials and transport type to connect to the device.
The `device_os` variable is the type of device OS that the current host is. We use this to decide which tasks and templates to run since each OS can have OS specific commands. This should be coming from a group var.
