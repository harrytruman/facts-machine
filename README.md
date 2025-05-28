# Facts Machine

## Network Facts, Configs, and Backups with Ansible

The Facts Machine parses network configs into a data model. This role gathers native Ansible Facts or sets custom facts to parse command output and convert device configurations into code.

Once gathered, facts can be used as backups/restores, called later as variables in other roles or playbooks, and used to define the state of a device. And most importantly, facts are used to build the framework of a network CMDB!

This role will is compatible with the following platforms:

```
Linux
Windows
--
IOS
IOS-XR
NX-OS
AireOS
EOS
JunOS
Aruba
F5
Palo Alto
Fortigate
VYOS
```

--------------

### Find Your Host Login Details

Ansible needs a few minimum details to get started. In particular, the `ansible_os` and `ansible_network_os` inventory variables to define the respective server or device OS (which should ideally be coming from a proper CMDB).

At a minimum, you need to define these for each of your inventory hosts:
```
ansible_hostname     hostname_fqdn
ansible_username     username
ansible_password     password
       and...
ansible_os           redhat/ubuntu/windows/etc...
       and/or...
ansible_network_os   ios/nxos/eos/etc
```

### Buid an Inventory File

And then you need an inventory file that lists the hosts you're connecting to:

P.S. I *highly* recommend [vaulting your passwords/keys/creds](https://docs.ansible.com/ansible/latest/user_guide/vault.html#creating-encrypted-variables) instead of storing them plaintext! My inventories usually start like this:

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

## Ansible Fact Collection

Ansible's native fact gathering can be invoked by setting `gather_facts: true` in your top level playbook. And every major networking vendor has fact modules that you can use in a playbook task: `ios_facts`, `eos_facts`, `nxos_facts`, `junos_facts`, etc... Just enable `gather_facts`, and you're on your way!

Here's an example of gathering facts from a Cisco IOS device to create a backup of the full running config, and parse config subsets into a platform-agnostic data model:

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
    cisco.ios.ios_facts:
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

Ansible Facts can be cached too! Options include local file, memcached, Redis, and a plethora of others, via Ansible's [Cache Plugins](https://docs.ansible.com/ansible/latest/plugins/cache.html). And caching can be enabled with just the click button in AWX/AAP Job Templates, where you can then view facts via UI and API both.

![fact cache](https://docs.ansible.com/ansible-tower/latest/html/userguide/_images/job-templates-options-use-factcache.png)

The combination of using network facts and fact caching can allow you to poll existing, in-memory data rather than parsing numerous additional commands to constantly check/refresh the device's running config.

When using AAP, you can access cached facts for an individual host via:
```https://{{ aap_fqdn }}/api/v2/hosts/{{ inventory_host }}/ansible_facts```


### Backups and Restores

I consider device backups part of the fact collection process. If you're already connecting to a device and parsing its config, you might as well make a backup too. In the same time that Ansible is parsing config lines, you can easily have it dump the full running-config to a backup location of any kind -- local file, external share, git repo, etc...
```
- cisco.ios.ios_config:
  backup: yes
  backup_options:
    filename: "{{ ansible_network_os }}-{{ inventory_hostname }}.cfg"
    dir_path: /var/tmp/backup/
```

And if you want to restore these configs, just grab the most recent backup file:
```
- name: restore config
   cisco.ios.ios_config:
    src: /var/tmp/backup/{{ ansible_network_os }}-{{inventory_hostname}}.cfg
```

### Create Your Own Custom Ansible Facts

You can also run custom commands, save the output, and parse the configuration later. Any output can be parsed and saved as a fact!

Here's an example of how to set a custom fact for Cisco IOS versions. This will run `show version`, find the version details, and save it as the variable `cisco-ios-version`.

```
- name: run `show version` command
  cisco.ios.ios_command:
    commands:
      - show version
  register: output

- name: set version fact
  ansible.builtin.set_fact:
    cisco-ios-version: "{{ output.stdout[0] | regex_search('Version (\\S+)', '\\1') | first }}"
```

Setting custom facts works particularly well for building out infrastructure checks/verifications. A good example of this is how F5 natively gathers the attached license. But you also can identify additional content that will help you automate expiration/renewal processes. For instance, this will run one command (`show sys license`) and set two facts: one for when the device was licensed, and another for the service check date:

```
- name: get license information - {{ inventory_hostname }}
  f5.bigip_command:
    commands:
      - show sys license
  register: license_output

- name: search for `licensed on` and `service check date`
  ansible.builtin.set_fact:
    licensed_on: "{{ license_output.stdout[0] | regex_search('Licensed On (.*)') }}"
    service_date: "{{ license_output.stdout[0] | regex_search('Service Check Date (.*)') }}"

- name: Get Licensed On and Service Check Date
  ansible.builtin.set_fact:
    licensed_on: "{{ licensed_on.split(' ') | last }}"
    service_date: "{{ service_date.split(' ') | last }}"

- ansible.builtin.debug:
    msg:
      - "Licensed On date is {{ licensed_on }}"
      - "Service Check Date is {{ service_date }}"
```

--------------

## Ansible at Scale

### Network vs OS

It’s important to differentiate that Ansible operates somewhat differently when running against network devices and cloud/API endpoints. When Ansible runs against a full OS like Linux, the remote hosts have Python/Bash, and they both receive commands and process their own data and state changes. As an example with a Linux host, a standard logging service configuration playbook would be fully executed on the remote host; upon completion, only task results are sent back to Ansible.

Network devices, on the other hand, rarely perform their own data processing. Until quite recently, very few network devices were built to have software APIs -- much less run Python. This presents a problem for any external configuration or management system. Things like SNMP may address some parts of this problem, by allowing some aspects of configuration and device state to be set or polled, but the reality is that a vast majority of networks are still managed via good ol' fashioned screen scrapes and command orchestration scripts...and that doesn't have to be the only option!

--------------

### Speed and Performance

Get into a habit of routinely checking your playbook runtimes. Basline peformance testing is your friend! This will display run times for individual tasks and the whole playbook run:

```
#ansible.cfg
callback_whitelist = profile_tasks, timer
```

--------------

### Ansible and Fact Collection at Scale

Ansible helps solve the problem of communicating with every device on your network. But even though this is the 21st century, network command orchestration is still accomplished primarily by sending commands to devices and having those devices return output back to Ansible. Over and over. Rather than being able to rely on remote devices to do their own work, Ansible handles all network data processing as it’s received from remote devices. For most network environments, all data processing will to be performed locally on Ansible/Tower/AAP. 

In the pursuit of scaling Ansible to manage large network device inventories, we must consider a number of factors that will directly impact job performance:

  1. Frequency and extent of orchestrating/scheduling device changes
  2. Device configuration size (raw text output from `show run`, etc..)
  3. Inventory sizes and devices families, e.g. IOS, NXOS, XR, Linux, etc...
  4. Ansible network facts modules, parsers, and fact caching

##### Frequency and extent of orchestrating/scheduling device changes
With any large inventory, there comes a balancing act between scheduling configuration changes and avoiding resource contention. At a high level, this can be as simple as benchmarking job run times with AAP resource loads, and setting job template forks accordingly. When creating new network automation roles, it’s important to establish solid development practices to avoid potentially significant processing times.

##### Device configuration size
Most network automation roles will be utilizing Ansible facts derived from device configs. By looking at the raw device config sizes, such as the text output from `show run`, we can establish a rough estimate of memory usage per-host during large jobs.

##### Inventory sizes and devices families, e.g. IOS, NXOS, XR
Due to the large inventory size and the likelihood of significant inventory metadata, it’s critical to ensure that inventories are broken into smaller groups -- group sizes of 5,000 or less are highly recommended. Additionally, it’s important to note that device types/families perform noticeably faster/slower than others. IOS, for instance, is often 3-4 faster than NXOS.

##### Implementation and availability of Ansible network facts
Ansible can collect device facts -- useful variables about remote hosts that can be used in playbooks. Additionally, these facts can be cached in AAP. The combination of using network facts with the fact cache can significantly increase AAP job speed and reduce processing loads.

--------------

## Storing and Using Facts

Unless your clusters have significant free resources to spare, AAP is not ideal for collecting, storing, *and* retrieving facts in environments with constant jobs running against large inventories. Simply put, processing all of these local facts **while** running a full cluster will put a tremendous strain on AAP.

Any CMDB and Source of Truth should be implemented external to AAP. And it's highly suggested to use something like an ELK cluster to store AAP logs and Ansible Facts:

https://github.com/harrytruman/elk-ansible
