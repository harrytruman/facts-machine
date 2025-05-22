# Collections

Collections are a distribution format for Ansible content that can include playbooks, roles, modules, and plugins. You can install and use collections through a distribution server, such as Ansible Galaxy, or a Pulp 3 Galaxy server.

## Installing collections with `ansible-galaxy`

By default, ansible-galaxy collection install uses https://galaxy.ansible.com as the Galaxy server (as listed in the `ansible.cfg` file under `GALAXY_SERVER`). You do not need any further configuration.

* To **install** a collection hosted in Galaxy:

```
ansible-galaxy collection install my_namespace.my_collection
```

* To **upgrade** a collection to the latest available version from the Galaxy server, you can use the --upgrade option:

```
ansible-galaxy collection install my_namespace.my_collection --upgrade
```

* You can also directly use the **tarball** from your build:

```
ansible-galaxy collection install my_namespace-my_collection-1.0.0.tar.gz -p ./collections
```

* You can build and install a collection from a **local source directory**.
The ansible-galaxy utility builds the collection using the MANIFEST.json or galaxy.yml metadata in the directory.

```
ansible-galaxy collection install /path/to/collection -p ./collections
```

* Finally, you can also install multiple collections in a namespace directory.
```
ns/
├── collection1/
│   ├── MANIFEST.json
│   └── plugins/
└── collection2/
    ├── galaxy.yml
    └── plugins/
```
```
ansible-galaxy collection install /path/to/ns -p ./collections
```

## Using collections in a playbook

Once installed, you can reference a collection content by its fully qualified collection name (FQCN):
```
- hosts: all
  tasks:
    - my_namespace.my_collection.mymodule:
        option1: value
```

This works for roles or any type of plugin distributed within the collection:

```
- hosts: all
  tasks:
    - import_role:
        name: my_namespace.my_collection.role1

    - my_namespace.mycollection.mymodule:
        option1: value

    - debug:
        msg: '{{ lookup("my_namespace.my_collection.lookup1", 'param1')| my_namespace.my_collection.filter1 }}'
```

## Install multiple collections with a requirements file

You can set up a requirements.yml file to install multiple collections in one command. This file is a YAML file in the format:
```
---
collections:
# With just the collection name
- my_namespace.my_collection

# With the collection name, version, and source options
- name: my_namespace.my_other_collection
  version: 'version range identifiers (default: ``*``)'
  source: 'The Galaxy URL to pull the collection from (default: ``--api-server`` from cmdline)'
```

You can specify four keys for each collection entry:

```
name
version
source
type
```

The version key uses the same range identifier format documented in Installing an older version of a collection.

The type key can be set to `file`, `galaxy`, `git`, `url`, `dir`, or `subdirs`. If type is omitted, the name key is used to implicitly determine the source of the collection.

When you install a collection with `type: git`, the version key can refer to a branch or to a git commit-ish object (commit or tag). For example:

```
collections:
  - name: https://github.com/organization/repo_name.git
    type: git
    version: devel
```

You can also add roles to a requirements.yml file, under the roles key. The values follow the same format as a requirements file used in older Ansible releases.

```
---
roles:
  # Install a role from Ansible Galaxy.
  - name: geerlingguy.java
    version: 1.9.6

collections:
  # Install a collection from Ansible Galaxy.
  - name: geerlingguy.php_roles
    version: 0.9.3
    source: https://galaxy.ansible.com
```

To install both roles and collections at the same time with one command, run the following:
```
$ ansible-galaxy install -r requirements.yml
```

Running `ansible-galaxy collection install -r` or `ansible-galaxy role install -r` will only install collections, or roles, respectively.
