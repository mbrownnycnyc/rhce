* https://app.pluralsight.com/course-player?clipId=d50b1719-6b08-4026-83bc-3b4bd38b7772

cd ~/vagrant/ansible
vagrant up --parallel


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
vagrant up --parallel
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
vagrant up --parallel
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
vagrant up --parallel

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

# Managing ansible inventories

* an inventory list of nodes
* default inventory `/etc/ansible/hosts`

## inventory types

1. if you run `ansible-config dump --only-changed`, you can see DEFAULT_HOST_LIST
```
[vagrant@rhel8 ~]$ ansible-config dump --only-changed
DEFAULT_BECOME(/home/vagrant/.ansible.cfg) = True
DEFAULT_HOST_LIST(/home/vagrant/.ansible.cfg) = ['/home/vagrant/inventory']
DEFAULT_REMOTE_USER(/home/vagrant/.ansible.cfg) = tux
```

2. take a look at the existing inventory
```
[vagrant@rhel8 ~]$ ansible-inventory --host localhost
[WARNING]: Unable to parse /home/vagrant/inventory as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
{
    "ansible_connection": "local",
    "ansible_python_interpreter": "/usr/libexec/platform-python"
}
```

3. use the inventory
```
[vagrant@rhel8 ~]$ ansible localhost -m ping
[WARNING]: Unable to parse /home/vagrant/inventory as an inventory source
[WARNING]: No inventory was parsed, only implicit localhost is available
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## creating inventory file and undersatnd groups

1. create the file
```
[vagrant@rhel8 ~]$ cat ~/inventory
192.168.33.11
192.168.33.12
192.168.33.13
```

2. review implicit groups:
```
[vagrant@rhel8 ~]$ ansible --list all
  hosts (3):
    192.168.33.11
    192.168.33.12
    192.168.33.13

[vagrant@rhel8 ~]$ ansible --list ungrouped
  hosts (3):
    192.168.33.11
    192.168.33.12
    192.168.33.13
```

3. create the groups in the inventory file
```
[vagrant@rhel8 ~]$ cat ~/inventory
[rhel]
192.168.33.11
[stream]
192.168.33.12
[ubuntu]
192.168.33.13
[Redhat:children]
stream
rhel
```
* note the use of groups, and children groups (must be designated via `*:children`)

4. list out these created groups
```
[vagrant@rhel8 ~]$ ansible --list rhel
  hosts (1):
    192.168.33.11
[vagrant@rhel8 ~]$ ansible --list Redhat
  hosts (2):
    192.168.33.12
    192.168.33.11
```

## inventory output formats

1. list inventory in json
```
[vagrant@rhel8 ~]$ ansible-inventory --list
{
    "Redhat": {
        "children": [
            "rhel",
            "stream"
        ]
    },
    "_meta": {
        "hostvars": {}
    },
    "all": {
        "children": [
            "Redhat",
            "ubuntu",
            "ungrouped"
        ]
    },
    "rhel": {
        "hosts": [
            "192.168.33.11"
        ]
    },
    "stream": {
        "hosts": [
            "192.168.33.12"
        ]
    },
    "ubuntu": {
        "hosts": [
            "192.168.33.13"
        ]
    }
}
```
   
2. list ansible inventory in yaml
```
[vagrant@rhel8 ~]$ ansible-inventory --list Redhat -y
all:
  children:
    Redhat:
      children:
        rhel:
          hosts:
            192.168.33.11: {}
        stream:
          hosts:
            192.168.33.12: {}
    ubuntu:
      hosts:
        192.168.33.13: {}
    ungrouped: {}
```

## implement inventory variables
* create host_vars and group_vars directory for groups
  * for host_vars, we'll leverage the `ansible_connection` directive to set the connection type

1. create a host_vars file
```
cd ~
mkdir host_vars
echo "ansible_connection: local" > ~/host_vars/192.168.33.11
```

2. list the host variable
```
[vagrant@rhel8 ~]$ ansible-inventory --host 192.168.33.11
{
    "ansible_connection": "local"
}
```

3. use the host variable in an ansible invocation
```
[vagrant@rhel8 ~]$ ansible 192.168.33.11 -m ping
192.168.33.11 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```

## build dynamic inventories
* script to discover hosts and then create inventories

1. install nmap and scan
```
sudo yum -y install nmap
sudo nmap -Pn -p22 -n 192.168.33.0/24 --open
sudo nmap -Pn -p22 -n 192.168.33.0/24 --oG -
#use awk to only take the second string in lines that contain "22/open" and print the second space-separated string
sudo nmap -Pn -p22 -n 192.168.33.0/24 --oG - | awk '/22\/open/ { print $2 }' > ~/nmap_inventory
```

# managing nodes using ad-hoc commands

1. copy vagrant keys to controller

2. connect to systems to test and collect node public keys

3. create `tux` user with sudo rights

4. generate keys for vagrant to log in as `tux` on remote systems and distributed to remote nodes

5. adjust ansible.cfg to use private key


## place private keys on the controller system

1. locate the current keys that vagrant client uses
```
#on the host system
cd ~/vagrant/ansible
$vagrantVMs = vagrant status | sls "running" | % {($_ -split "\s+")[0]}
$vagrantsshconfigstrs = @()
$vagrantVMs | % { $vagrantsshconfigstrs += , ( ((vagrant ssh-config $_) -replace "^\s+","" ) | ? { $_.length -gt 0} ) }
```

2. because the vagrant-scp has a bug where it doesn't parse Windows filepaths correctly, and because i refuse to use WSL, here's a powershell function that creates an object out of `vagrant ssh-config` output, then copies the keys to the `rhel8` VM.
   
```
#capture the vagrant VMs' ssh config
$vagrantsshconfigs = @()
foreach ($item in $vagrantsshconfigstrs ) {
  
  #create a new object that stores all the key-values of the vagrant ssh config
  $vagrantsshconfig = new-object psobject

  foreach ($line in $item) {
    [regex]$rex="^(?<property>\w+)\s(?<value>.*)"
    $keyvalue = $rex.match($line)
    $vagrantsshconfig | add-member -notepropertyname $($keyvalue.groups["property"].value) -notepropertyvalue $($keyvalue.groups["value"].value)
  }

  $vagrantsshconfigs += , $vagrantsshconfig
}

#copy keys to targets that aren't rhel8
$rhel8host = ($vagrantsshconfigs | ? {$_.host -like "rhel8"}).hostname
$rhel8port = ($vagrantsshconfigs | ? {$_.host -like "rhel8"}).port
$rhel8key = ($vagrantsshconfigs | ? {$_.host -like "rhel8"}).identityfile

foreach ( $sshconfigs in $( $vagrantsshconfigs | ? {$_.host -notlike "rhel8"} ) ) {
  
  $keypath = $sshconfigs.identityfile

  $sb = [scriptblock]::create("scp -r -P $rhel8port -o StrictHostKeyChecking=no -i $($rhel8key) $($sshconfigs.identityfile) vagrant@$($rhel8host)://home/vagrant/$($sshconfigs.host).key")

  write-output "invoking: $sb"
  . $sb

}
```

3. test connection to other VMs from `rhel8`

```
vagrant ssh rhel8
chmod og-rwx stream.key ubuntu.key
ansible stream --private-key stream.key -u vagrant -m ping
ansible ubuntu --private-key ubuntu.key -u vagrant -m ping
```

## establish the `tux` user account

1. create a user account
```
ansible stream --private-key stream.key -u vagrant -m user -a "name=tux"
ansible ubuntu --private-key ubuntu.key -u vagrant -m user -a "name=tux"
```

2. setup a sudo file

```
#create the sudo file
echo "tux ALL=(root) NOPASSWD: ALL" > tux_sudo
#validate the sudo file
visudo -cf tux_sudo
```

3. copy the file
```
ansible stream --private-key stream.key -u vagrant -m copy -a "src=tux_sudo dest=/etc/sudoers.d/"
ansible ubuntu --private-key ubuntu.key -u vagrant -m copy -a "src=tux_sudo dest=/etc/sudoers.d/"
```

4. generating a key pair

```
ssh-keygen
#accept default name
#don't set a passphrase
```

5. copy the public key to authed keys on the target systems
```
ansible stream --private-key stream.key -u vagrant -m authorized_key -a " user=tux state=present key='{{ lookup('file','/home/vagrant/.ssh/id_rsa.pub') }}' "
ansible ubuntu --private-key ubuntu.key -u vagrant -m authorized_key -a " user=tux state=present key='{{ lookup('file','/home/vagrant/.ssh/id_rsa.pub') }}' "
```

7. add the keys to use with auth to the ~/.ansible.cfg

```
# add `private_key_file = ~/.ssh/id_rsa` to the ansible.cfg
[vagrant@rhel8 ~]$ cat ~/.ansible.cfg
[defaults]
inventory = inventory
remote_user = tux
private_key_file = ~/.ssh/id_rsa

[privilege_escalation]
become = true
```

8. test connectivity
```
[vagrant@rhel8 ~]$ ansible all -m ping
192.168.33.11 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
192.168.33.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
192.168.33.13 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
```

## using variables

* `-m` specifies the python module to use
* challenge: resolve different pacakge names for different distros
  * package manager of target systems is irrelevant thanks to the `package` module.
  * use a group_var to store a variable, and then use a jinja variable during an invocation

1. review documentation
```
ansible-doc package
#then search for EXAMPLE
```
   
2. the package `tree` is the same...
```
ansible all -m package -a "name=tree state=present"
```

3. but `vim` is different between rhel and ubuntu

```
#review the group structure in the inventory, noting the groups `Redhat` and `ubuntu`
[vagrant@rhel8 ~]$ ansible-inventory --list -vvv
ansible-inventory 2.9.27
  config file = /home/vagrant/.ansible.cfg
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.6/site-packages/ansible
  executable location = /usr/bin/ansible-inventory
  python version = 3.6.8 (default, Jun 14 2022, 09:19:35) [GCC 8.5.0 20210514 (Red Hat 8.5.0-13)]
Using /home/vagrant/.ansible.cfg as config file
host_list declined parsing /home/vagrant/inventory as it did not pass its verify_file() method
script declined parsing /home/vagrant/inventory as it did not pass its verify_file() method
auto declined parsing /home/vagrant/inventory as it did not pass its verify_file() method
Parsed /home/vagrant/inventory inventory source with ini plugin
{
    "Redhat": {
        "children": [
            "rhel",
            "stream"
        ]
    },
    "_meta": {
        "hostvars": {
            "192.168.33.11": {
                "ansible_connection": "local",
                "vim_editor": "vim-enhanced"
            },
            "192.168.33.12": {
                "vim_editor": "vim-enhanced"
            },
            "192.168.33.13": {
                "vim_editor": "vim"
            }
        }
    },
    "all": {
        "children": [
            "Redhat",
            "ubuntu",
            "ungrouped"
        ]
    },
    "rhel": {
        "hosts": [
            "192.168.33.11"
        ]
    },
    "stream": {
        "hosts": [
            "192.168.33.12"
        ]
    },
    "ubuntu": {
        "hosts": [
            "192.168.33.13"
        ]
    }
}

#create relevant `group_vars` files
mkdir ~/group_vars/
echo "vim_editor: vim-enhanced"  >> ~/group_vars/Redhat
echo "vim_editor: vim" >> ~/group_vars/ubuntu

#review how the `group_vars` applies at runtime to both systems, because of their group scope
[vagrant@rhel8 ~]$ ansible-inventory --host stream
{
    "vim_editor": "vim-enhanced"
}
[vagrant@rhel8 ~]$ ansible-inventory --host ubuntu
{
    "vim_editor": "vim"
}

# remove the package by using the jinja group_var
ansible all -m package -a "name={{ vim_editor }} state=absent"

# (re)add the package by using the jinja group_var
ansible all -m package -a "name={{ vim_editor }} state=present"
```

# Linux Administration with Ansible: Writing Ansible Playbooks

# writing in yaml

## understanding the yaml structure

1. access https://jsonformatter.org/yaml-parser

2. review yaml structure
* make sure you understand indentation

![](2023-06-30-05-15-31.png)

## choosing text editors and IDEs for yaml

1. note that you should use spaces, not tabs

2. for vim, you can use the following in `~/.vimrc`
* https://vimhelp.org/options.txt.html
* I really don't like `cursorcolumn` and `cursorline`
```
set number
set autoindent
set expandtab
set tabstop=2
set shiftwidth=2
```


## understanding playbooks

1. create a playbook, with a play, with a task
```
mkdir -p ~/ansible/simple && cd ~/ansible/simple
vim firstplay.yaml
- name: first play
  hosts: all
  become: true
  tasks:
    - name: first task
      package:
        name: tree
        state: present
```

2. install a linter, and then lint
```
sudo dnf -y install python3-devel
pip3 install ansible-lint --user
ansible-lint -v firstplay.yaml
```

3. check syntax
```
#returns nothing means no errors 
ansible-playbook --syntax-check firstplay.yaml
```

4. perform a whatif (no operations)
```
sudo yum -y remove tree
#returns nothing means no errors 
ansible-playbook -C -v firstplay.yaml
```

5. execute the playbook

```
ansible-playbook firstplay.yaml
```

## using variables and debugging playbooks

* facts are OS relevant information

1. active enable facts gathering in the playbook
```
#at the play level, add "gather_facts: true"
- name: first play
  hosts: all
  become: true
  gather_facts: true
```
   
2. add another task to the playbook
* `ansible_os_family` is a fact
```
    - name: print progress
      debug:
        msg: "this is {{ ansible_os_family }}"
```

3. note that when you execute tasks, that each task's status/output is sent to stdout:
* note the `TASK [print progress]`

```
[vagrant@rhel8 simple]$ ansible-playbook firstplay.yaml

PLAY [first play] ************************************************************************************************************************************************************************************
TASK [Gathering Facts] *******************************************************************************************************************************************************************************ok: [192.168.33.11]
ok: [192.168.33.12]
ok: [192.168.33.13]

TASK [first task] ************************************************************************************************************************************************************************************ok: [192.168.33.13]
ok: [192.168.33.12]
ok: [192.168.33.11]

TASK [print progress] ********************************************************************************************************************************************************************************ok: [192.168.33.13] => {
    "msg": "this is Debian"
}
ok: [192.168.33.12] => {
    "msg": "this is RedHat"
}
ok: [192.168.33.11] => {
    "msg": "this is RedHat"
}

PLAY RECAP *******************************************************************************************************************************************************************************************192.168.33.11              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.33.13              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## locate python/ansible modules

1. find a module

```
[vagrant@rhel8 simple]$ find /usr/lib -name package.py | egrep package
/usr/lib/python3.6/site-packages/dnf/package.py
/usr/lib/python3.6/site-packages/ansible/modules/packaging/os/package.py <-------
/usr/lib/python3.6/site-packages/ansible/plugins/action/package.py
[vagrant@rhel8 simple]$ find /usr/lib -name debug.py | egrep debug
/usr/lib/python3.6/site-packages/dnf-plugins/debug.py
/usr/lib/python3.6/site-packages/jinja2/debug.py
/usr/lib/python3.6/site-packages/ansible/modules/utilities/logic/debug.py <-------
/usr/lib/python3.6/site-packages/ansible/plugins/action/debug.py
/usr/lib/python3.6/site-packages/ansible/plugins/callback/debug.py
/usr/lib/python3.6/site-packages/ansible/plugins/strategy/debug.py
```

# scripting linux administration

* note that we've already performed provisioning of the VMs using the below playbook below in the first module.

* create a user account with the `user` module
* use the `copy` module to set content on a file
* edit a single line in the sshd_config with the `lineinfile` module, leveraging the `notify` module
* a `handler` is the target of the `notify` call

1. review the playbook
```
- name: Deploy Systems
  hosts: all
  become: true
  tasks:
    - name: Create_user
      user:
        state: present
        shell: /bin/bash
        name: tux
        password: "{{ 'Password1' | password_hash('sha512') }}"
        update_password: on_create
    - name: sudo
      copy:
        dest: /etc/sudoers.d/tux
        content: "tux ALL=(root) NOPASSWD: ALL"
    - name: edit_sshd
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PasswordAuthentication'
        line: 'PasswordAuthentication yes'
        insertafter: '^#PasswordAuthentication'
      notify: restart_sshd
  handlers:
    - name: restart_sshd
      service:
        name: sshd
        state: restarted
```

# using shell commands in playbook

* running shell commands when moduels not available:
  * `shell`
  * `command`
  * `raw`
  * `script`
  * `win_comment`

## command versus shell modules

* `command`: execution of native linux command without the need of creating a parent shell.
* `shell`: opens a shell `/bin/sh` to execute the commands.  Allows for variables and stream operators (considered less secure).

### security:
```
[vagrant@rhel8 ~]$ MYVAR='myval;ls -l /etc/hosts'
[vagrant@rhel8 ~]$ ansible rhel -m command -a "echo $MYVAR"
192.168.33.11 | CHANGED | rc=0 >>
myval;ls -l /etc/hosts

[vagrant@rhel8 ~]$ ansible rhel -m shell -a "echo $MYVAR"
192.168.33.11 | CHANGED | rc=0 >>
myval
-rw-r--r--. 1 root root 188 Mar 30 00:24 /etc/hosts
```

* note that this is risky

## working with the ansible raw module
* in cases where python isn't installed
```
ansible all -i 18.172.45.3 -b -m raw -a "dnf install -y python"
```

## executing scripts withing the script module
* execute a script on controller and have it script on inventory

```
cat << EOF | tee script.sh
#!/bin/sh
echo "hello world"
EOF
ansible all -m script -a  "/home/vagrant/script.sh"
```

## idempotentcy with creates
```
mkdir -p ~/ansible/loop; cd ~/ansible/loop
cat << EOF | tee loopdevice.yaml
- name: 'manage disk file'
  hosts: rhel
  become: true
  gather_facts: false
  tasks:
  - name: "create disk file"
    command:
      cmd: 'fallocate -l 1G /root/disk0'
      creates: '/root/disk0' #this is a meta parameter... this tells ansible that this task will CREATE this file... so if the file exists... the task won't be run again
EOF
[vagrant@rhel8 loop]$ ansible-playbook loopdevice.yaml

PLAY [manage disk file] ******************************************************************************************************************************************************************************

TASK [create disk file] ******************************************************************************************************************************************************************************
changed: [192.168.33.11]

PLAY RECAP *******************************************************************************************************************************************************************************************
192.168.33.11              : ok=1    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

[vagrant@rhel8 loop]$ ansible-playbook loopdevice.yaml

PLAY [manage disk file] ******************************************************************************************************************************************************************************

TASK [create disk file] ******************************************************************************************************************************************************************************
ok: [192.168.33.11]

PLAY RECAP *******************************************************************************************************************************************************************************************
192.168.33.11              : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## using playbook/global vars
