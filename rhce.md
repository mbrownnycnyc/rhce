* https://app.pluralsight.com/course-player?clipId=e6c752e7-eb41-4f71-80e6-6edbdcd275f2


* https://app.pluralsight.com/explore/certifications/topics/linux?trackId=43182a5e-279a-4ef6-808c-c3f62d01ed57&examPrepId=0e4f07d3-5443-4d9c-9331-2a886c87bdb6

# Linux Administration with Ansible: Getting Started with Ansible Automation

# Managing a growing linux estate

## introduction
* managing systems
* scripting solutions
* building lab system

## lab environment
* three VMs
* virtualbox, vagrant
* RHEL 8, ubuntu 20.04, centos stream

### vagrant for lab builds
* on host system
```
mkdir -p ~/vagrant/ansible
vim ~/vagrant/ansible/Vagrantfile
cd ~/vagrant/ansible
vagrant up
```

## building the vagrant labs

### summary
* on host OS
* create the required directories in the host file system
* create/download the Vagrantfile

### setup
1. sign up for redhat developer subscription
* https://developers.redhat.com/

2. install vagrant
* https://developer.hashicorp.com/vagrant/downloads and install

3. install virtualbox
* https://www.virtualbox.org/wiki/Downloads

4. create directories (in pwsh)
* having downloaded the course files, copy the `Vagrantfile` from there.
```
mkdir -f ~/vagrant/ansible
cd ~/vagrant/ansible
copy C:\repos\rhce\linux-administration-ansible-automation-getting-started\02\demos\Vagrantfile .\
cat .\Vagrantfile | % {$_ -replace "centos/stream8","bento/centos-stream-8"} | set-content .\Vagrantfile
```

5. run vagrant up, which will download all three distros, and create VMs.
* this will take a while (an hour or so) and may require initial interaction.
```
cd ~/vagrant/ansible
vagrant up
```

6. register the RHEL VM with redhat
```
cd ~/vagrant/ansible
vagrant ssh rhel8
# there is no longer a need to use auto-attach as simple content access is enabled by default on new dev subscriptions
sudo subscription-manager register --username mbrowndc --password []
# validate
[root@rhel8 vagrant]# subscription-manager status
+-------------------------------------------+
   System Status Details
+-------------------------------------------+
Overall Status: Disabled
Content Access Mode is set to Simple Content Access. This host has access to content, regardless of subscription status.

System Purpose Status: Disabled
```

7. some vagrant commands:
```
cd ~/vagrant/ansible

#check status of VMs
vagrant status

#shutdown VMs
vagrant halt

#power up VMs
vagrant up

#destroy VMs
vagrant destroy

#get shell on a system
vagrant ssh rhel8
#exit
```

## not all OSes are the same
* managing systems theory
  * connecting to each server in turn and esecutign commands needed to bring the system into the required stat
* scripting solutions was okay, but not great
* System differences
  * while discrete commands or scripts can managed the estate, as an administrator of more than one linux distro
  * various software packaging tools (apt, yum)
  * editor (vim, vim-enhanced)
  * time config (/etc/chrony/chrony.conf, /etc/chrony.conf)

# installing ansible
1. adding ansible repos and installing ansible
* we'll only really use ansible on the RHEL instance
```
# vagrant ssh rhel8
#check repos
sudo subscription-manager repos --list | grep ansible
sudo subscription-manager repos --enable ansible-2.9-for-rhel-8-x86_64-rpms
sudo yum repolist
#disable epel repo (which also has the ansible package) so that ansible repo is prioritized
sudo yum install -y dnf-plugins-core
sudo yum config-manager --disable epel epel-modular
sudo yum repolist
sudo yum install -y ansible

# vagrant ssh stream
sudo yum install -y epel-release
sudo yum install -y ansible

# vagrant ssh ubuntu
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

2. versioning ansible
```
[vagrant@rhel8 ~]$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.6.8 (default, Jun 14 2022, 09:19:35) [GCC 8.5.0 20210514 (Red Hat 8.5.0-13)]
```

3. investigating ansible from the CLI
```
[vagrant@rhel8 ~]$ ls /usr/lib/python3.6/site-packages/ansible/modules
cloud       commands  database  identity     inventory  monitoring  network       packaging    remote_management  storage  utilities           windows
clustering  crypto    files     __init__.py  messaging  net_tools   notification  __pycache__  source_control     system   web_infrastructure

[vagrant@rhel8 ~]$ ls /usr/lib/python3.6/site-packages/ansible/playbook
attribute.py  block.py             conditional.py  handler_task_include.py  included_file.py  loop_control.py      play_context.py  __pycache__  role_include.py  task_include.py
base.py       collectionsearch.py  handler.py      helpers.py               __init__.py       playbook_include.py  play.py          role         taggable.py      task.py
```

# understanding ansible components

## what makes up ansible?

### `ansible` vs `ansible-playbook`
* issue ad-hoc commands (just with `ansible` invocations)
```
#ansible defaults to use ansible_connection=ssh
ansible_connection=local ansible localhost -m ping
```
  * optional variable: `ansible_connection=local`
  * command: `ansible`
  * target: `localhost`
  * module: `ping`
* use the package module
```
# `-b` == become, refers to /etc/sudoers
[vagrant@rhel8 ~]$ ansible_connection=local ansible localhost -b -m package -a 'name=zsh'
localhost | CHANGED => {
    "changed": true,
    "msg": "",
    "rc": 0,
    "results": [
        "Installed: zsh-5.5.1-10.el8.x86_64"
    ]
}
[vagrant@rhel8 ~]$ ansible_connection=local ansible localhost -b -m package -a 'name=zsh'
localhost | SUCCESS => {
    "changed": false,
    "msg": "Nothing to do",
    "rc": 0,
    "results": []
}
```

### finding documentation
```
ansible-doc -l

# shows options and examples
ansible-doc ping
ansible-doc package #note the use of `state`

#removing a package
[vagrant@rhel8 ~]$ ansible_connection=local ansible localhost -b -m package -a 'name=zsh state=absent'
localhost | CHANGED => {
    "changed": true,
    "msg": "",
    "rc": 0,
    "results": [
        "Removed: zsh-5.5.1-10.el8.x86_64"
    ]
}
[vagrant@rhel8 ~]$ ansible_connection=local ansible localhost -b -m package -a 'name=zsh state=absent'
localhost | SUCCESS => {
    "changed": false,
    "msg": "Nothing to do",
    "rc": 0,
    "results": []
}
```

### ansible playbooks
* create a playbook
  * note the use of jinja variable `{{ ansible_os_family }}`
```
vim my.yml
- name: Simple Play
  hosts: localhost
  connection: local
  tasks:
    - name: Ping me
      ping:
    - name: print os
      debug:
        msg: "{{ ansible_os_family }}"
```

* execute this playbook
```
[vagrant@rhel8 ~]$ ansible-playbook my.yml
[WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'

PLAY [Simple Play] ***********************************************************************************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************************************************************************
ok: [localhost]

TASK [Ping me] ***************************************************************************************************************************************************************************************
ok: [localhost]

TASK [print os] **************************************************************************************************************************************************************************************
ok: [localhost] => {
    "msg": "RedHat"
}

PLAY RECAP *******************************************************************************************************************************************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```

### ansible facts and variables
* list out facts
```
# lists all facts
ansible localhost -m setup

#list a specific fact
  # note the use of `localhost` is the inventory item `localhost`
[vagrant@rhel8 ~]$ ansible localhost -m setup  -a "filter=ansible_os_family"
localhost | SUCCESS => {
    "ansible_facts": {
        "ansible_os_family": "RedHat"
    },
    "changed": false
}
```
### locating default values and host inventory
* default config is present `/etc/ansible/ansible.cfg`
* by default, the inventory file is presented as:
```
[vagrant@rhel8 ~]$ cat /etc/ansible/ansible.cfg | egrep "inventory\s+\="
#inventory      = /etc/ansible/hosts
```

# managing the ansible config file

## configuring ansible

### config hierarchy
* config is used as prioritized:
  * `ANSIBLE_CONFIG`#most respected
  * `$CWD/ansible.cfg`
  * `$HOME/.ansible.cfg` 
  * `/etc/ansible/ansible.cfg` #least respected
  * note that the file must exist, or a lesser prioritized item will be honored

* review which current config is used (see `config file` value)
```
[vagrant@rhel8 ~]$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.6.8 (default, Jun 14 2022, 09:19:35) [GCC 8.5.0 20210514 (Red Hat 8.5.0-13)]

#create a local file
[vagrant@rhel8 ~]$ touch .ansible.cfg
[vagrant@rhel8 ~]$ ansible --version
ansible 2.9.27
  config file = /home/vagrant/.ansible.cfg
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.6.8 (default, Jun 14 2022, 09:19:35) [GCC 8.5.0 20210514 (Red Hat 8.5.0-13)]
```

## working with `ansible-config`
* view the contents of the ansible config
```
mkdir ~/test
cd ~/test
ansible --version | grep 'config file'
# as per previous section, this should be pointing to $HOME/.ansible.cfg
```

* review the contents of the cfg file only
```
ansible-config view
#remember it's an empty file!
```

* review the config at runtime with documentation
```
ansible-config list #with documentation
```

* review the actual values set
```
ansible-config dump
```

* if you modified a value in $HOME/.ansible.cfg, you can review that:
```
ansible-config dump --only-changed
```

## create custom configs
* modify the `$HOME/.ansible.cfg` file to adjust inventory
  * this calls a file called `inventory` as the inventory file (this inventory file is located in the CWD)
```
vim ~/.ansible.cfg
[defaults]
inventory = inventory
remote_user = tux

[privilege_excalation]
become = true
```

* there's an implicit entry for `ansible_connection=local`
```
[vagrant@rhel8 ~]$ ansible-config view
[defaults]
inventory = inventory
remote_user = tux

[privilege_escalation]
become = true

[vagrant@rhel8 ~]$ ansible-config dump --only-changed
DEFAULT_BECOME(/home/vagrant/.ansible.cfg) = True
DEFAULT_HOST_LIST(/home/vagrant/.ansible.cfg) = ['/home/vagrant/inventory']
DEFAULT_REMOTE_USER(/home/vagrant/.ansible.cfg) = tux

[vagrant@rhel8 ~]$ ansible localhost -m ping
[WARNING]: Unable to parse /home/vagrant/inventory as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

#note that the package install command needs priv elevation
[vagrant@rhel8 ~]$ ansible localhost -m package -a "name=zsh"
[WARNING]: Unable to parse /home/vagrant/inventory as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
localhost | CHANGED => {
    "changed": true,
    "msg": "",
    "rc": 0,
    "results": [
        "Installed: zsh-5.5.1-10.el8.x86_64"
    ]
}
```

## enforce a configuration
* use the ANSIBLE_CONFIG env var to enforce a config
```
#you can set this up as a read only variable in the bash login script (not editable by regular users)
declare -xr ANSIBLE_CONFIG=/etc/ansible/ansible.cfg
```