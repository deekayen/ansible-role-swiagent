Solarwinds Agent
=========

[![Build Status](https://travis-ci.org/deekayen/ansible-role-swiagent.svg?branch=master)](https://travis-ci.org/deekayen/ansible-role-swiagent)

Install the Solarwinds Agent from Orion to communicate outbound to a Solarwinds poller.

You can get the latest RPMs from URLs on Solarwinds:

* https://solarwinds.example.com/Orion/AgentManagement/DownloadLinuxPackage.ashx?packageId=rhel-7.0-x86
* https://solarwinds.example.com/Orion/AgentManagement/DownloadLinuxPackage.ashx?packageId=rhel-7.0-x64

Instructions for offline installation:

https://support.solarwinds.com/Success_Center/Orion_Platform/Knowledgebase_Articles/Manual_offline_installation_of_an_Orion_agent

Firewalling and security group rules are discussed on:

https://support.solarwinds.com/Success_Center/Network_Automation_Manager/NAM_Install_Guide/030/020,


Requirements
------------

Perl will be installed in addition to swiagent as part of supporting monitoring extensions.

Role Variables
--------------

```
orion_hosts: []
```

Dependencies
------------

None.

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

    - hosts: all:!platform_windows

      vars:
        orion_hosts:
          - https://solarwinds.example.com
          - https://10.13.251.123:443
          - https://10.13.251.34

      roles:
        - deekayen.swiagent

License
-------

BSD

Author Information
------------------

David Norman
