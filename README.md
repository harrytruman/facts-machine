Facts Machine
=========

Network Facts
-------------

This role gathers facts about network devices to be used as backups/restores, later in other roles or playbooks, and in the creation of a network CMDB.

Role Variables
--------------

The `cli` variable holds the credentials and transport type to connect to the device.
The `device_os` variable is the type of device OS that the current host is. We use this to decide which tasks and templates to run since each OS can have OS specific commands. This should be coming from a group var.
