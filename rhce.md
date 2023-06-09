* https://app.pluralsight.com/course-player?clipId=89c8a865-1f08-40f6-92d9-0e905e152405


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


## place private keys for inventory hosts on the controller system

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
1. Create a file with global vars, and also leverages create

```
mkdir ~/ansible/loop
cat << EOF | tee loopdevice.yaml
- name: 'Manage Disk File'
  hosts: all
  gather_facts: false
  vars:
    disk_file: '/root/disk0'
    loop_dev: '/dev/loop100'
  tasks:
    - name: 'Create raw disk file'
      command:
        cmd: "fallocate -l 1G {{ disk_file }}"
        creates: "{{ disk_file }}"
    - name: 'Create loop device'
      command:
        cmd: "losetup {{ loop_dev }} {{ disk_file }}"
        creates: "{{ loop_dev }}"
    - name: 'Create XFS FS'
      filesystem:
        fstype: xfs
        dev: "{{ loop_dev }}"
EOF
```

2. running with creates:

```
[vagrant@rhel8 loop]$ ansible-playbook loopdevice.yaml

PLAY [Manage Disk File] ******************************************************************************************************************************************************************************

TASK [Create raw disk file] **************************************************************************************************************************************************************************
ok: [192.168.33.11]
changed: [192.168.33.12]
changed: [192.168.33.13]

TASK [Create loop device] ****************************************************************************************************************************************************************************
changed: [192.168.33.13]
changed: [192.168.33.11]
changed: [192.168.33.12]

TASK [Create XFS FS] *********************************************************************************************************************************************************************************
changed: [192.168.33.11]
changed: [192.168.33.13]
changed: [192.168.33.12]

PLAY RECAP *******************************************************************************************************************************************************************************************
192.168.33.11              : ok=3    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.33.12              : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.33.13              : ok=3    changed=3    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

[vagrant@rhel8 loop]$ ansible-playbook loopdevice.yaml

PLAY [Manage Disk File] ******************************************************************************************************************************************************************************

TASK [Create raw disk file] **************************************************************************************************************************************************************************
ok: [192.168.33.13]
ok: [192.168.33.11]
ok: [192.168.33.12]

TASK [Create loop device] ****************************************************************************************************************************************************************************
ok: [192.168.33.13]
ok: [192.168.33.11]
ok: [192.168.33.12]

TASK [Create XFS FS] *********************************************************************************************************************************************************************************
ok: [192.168.33.13]
ok: [192.168.33.11]
ok: [192.168.33.12]

PLAY RECAP *******************************************************************************************************************************************************************************************
192.168.33.11              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.33.12              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
192.168.33.13              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Windows commands with win_command
1. install `ansible.windows`

```
ansible-galaxy collection install ansible.windows
```

2. create a playbook that executes a `whoami`
```
- name: execute whoami
  win_command: whoami
```

# working with the big three

## what are the big three
* three items used to manage an OS instance
  * packages
  * files
  * services

## review the inventory

### listing inventory

```
[vagrant@rhel8 ~]$ cat inventory
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

2. validating inventory on each host
```
[vagrant@rhel8 ~]$ ansible all -m debug -a 'var=groups.keys()'
192.168.33.13 | SUCCESS => {
    "groups.keys()": "dict_keys(['all', 'ungrouped', 'rhel', 'stream', 'ubuntu', 'Redhat'])"
}
192.168.33.12 | SUCCESS => {
    "groups.keys()": "dict_keys(['all', 'ungrouped', 'rhel', 'stream', 'ubuntu', 'Redhat'])"
}
192.168.33.11 | SUCCESS => {
    "groups.keys()": "dict_keys(['all', 'ungrouped', 'rhel', 'stream', 'ubuntu', 'Redhat'])"

```


### listing membership

```
[vagrant@rhel8 ~]$ ansible-inventory --graph
@all:
  |--@Redhat:
  |  |--@rhel:
  |  |  |--192.168.33.11
  |  |--@stream:
  |  |  |--192.168.33.12
  |--@ubuntu:
  |  |--192.168.33.13
  |--@ungrouped:
```

## managing statis service and packages

1. install httpd

* version 1:
```
cat << EOF | tee apache.yaml
- name: 'Manage Apache Webserver Deployment'
  hosts: Redhat
  become: true
  gather_facts: false
  tasks:
    - name: 'Install the Apache Webserver'
      package:
        name: "httpd"
        state: 'present'
    - name: 'Ensure web server is running and enabled'
      service:
        name: "httpd"
        state: 'started'
        enabled: true
EOF
```

2. add content to web page by adding a new task
```
cat << EOF | tee apache.yaml
- name: 'Manage Apache Webserver Deployment'
  hosts: Redhat
  become: true
  gather_facts: false
  tasks:
    - name: 'Install the Apache Webserver'
      package:
        name: "httpd"
        state: 'present'
    - name: 'Ensure web server is running and enabled'
      service:
        name: "httpd"
        state: 'started'
        enabled: true
    - name: 'Copy web content'
      copy:
        dest: '/var/www/html/index.html'
        content:  'this is a simple webserver'
EOF
```

3. add content using the `|`
```
cat << EOF | tee apache.yaml
- name: 'Manage Apache Webserver Deployment'
  hosts: Redhat
  become: true
  gather_facts: false
  tasks:
    - name: 'Install the Apache Webserver'
      package:
        name: "httpd"
        state: 'present'
    - name: 'Ensure web server is running and enabled'
      service:
        name: "httpd"
        state: 'started'
        enabled: true
    - name: 'Copy web content'
      copy:
        dest: '/var/www/html/index.html'
        content: |
          this is a simple webserver
          <h1>welcome</h1>
EOF
```

4. copy a file with copy module's "src" parameter
```
cat << EOF | tee apache.yaml
- name: 'Manage Apache Webserver Deployment'
  hosts: Redhat
  become: true
  gather_facts: false
  tasks:
    - name: 'Install the Apache Webserver'
      package:
        name: "httpd"
        state: 'present'
    - name: 'Ensure web server is running and enabled'
      service:
        name: "httpd"
        state: 'started'
        enabled: true
    - name: 'Copy web content'
      copy:
        dest: '/var/www/html/index.html'
        src: index.html
EOF
```

5. copy a directory with copy module's "src" parameter
```
cat << EOF | tee apache.yaml
- name: 'Manage Apache Webserver Deployment'
  hosts: Redhat
  become: true
  gather_facts: false
  tasks:
    - name: 'Install the Apache Webserver'
      package:
        name: "httpd"
        state: 'present'
    - name: 'Ensure web server is running and enabled'
      service:
        name: "httpd"
        state: 'started'
        enabled: true
    - name: 'Copy web content'
      copy:
        dest: '/var/www/html/'
        src: 'web/'
EOF
```

## copy across multiple system types focusing on group_vars

* group_vars tell ansible to use different values for variables per group.
* * the `group_vars` directory exists 

1. establish the structure for `group_vars`
```
mkdir ~/group_vars

cat << EOF | tee ~/group_vars/Redhat 
apache_pkg: httpd
apache_src: httpd
EOF

cat << EOF | tee ~/group_vars/ubuntu
apache_pkg: apache2
apache_src: apache2
EOF
```

2. deploy using the group_var
```
cat << EOF | tee ~/apache.yaml
- name: 'Manage Apache Webserver Deployment'
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: 'Install the Apache Webserver'
      package:
        name: "{{ apache_pkg }}"
        state: 'present'
    - name: 'Ensure web server is running and enabled'
      service:
        name: "{{ apache_svc }}"
        state: 'started'
        enabled: true
    - name: 'Copy web content'
      copy:
        dest: '/var/www/html/'
        src:  'web/'
EOF
```

## managing chrony time server

1. create a simplified chrony config that will be distributed
```
mkdir ~/ansible/chrony
grep -Ev '^($|#)' /etc/chrony.conf > ~/chrony/chrony.conf
```

2. set up group_vars
```
echo 'chrony_conf: /etc/chrony.conf' >> ~/group_vars/Redhat
echo 'chrony_svc: chronyd' >> ~/group_vars/Redhat
echo 'chrony_conf: /etc/chrony/chrony.conf' >> ~/group_vars/ubuntu
echo 'chrony_svc: chrony' >> ~/group_vars/ubuntu
```

## using Handlers
* handlers will only run when called upon by another element within the task (when they are notified)
  * example: restart chrony once a conf is placed

```
tasks:
  - name: "chrony conf"
    src: 'chrony.conf'
    dest: "{{ chrony_conf }}"
  notify: restart_chrony

handlers:
- name: 'restart_chrony'
  service:
    name: "{{ chrony_svc }}"
    state: 'restarted'
```

## create the chrony playbook

* this is a good example of `group_vars`, use of `handlers`, and the following modules: `package`, `service`, `copy`

1. create the yaml
* note that the copy module's src is an absolute path used at runtime
  * the searched paths are on the ansible Controller and are relevative to the directory where the playbook is located:
    * ~/chrony/files/ansible/chrony/chrony.conf <-- this is the usual convention
    * ~/chrony/ansible/chrony/chrony.conf
```
mkdir ~/chrony/
cat << EOF | tee ~/chrony/chrony.yaml
- name: 'Manage the chrony timeserver'
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: 'Ensure that chrony time server is installed'
      package:
        name: 'chrony'
        state: 'present'
    - name: 'Ensure chrony time server is enabled and running'
      service:
        name: "{{ chrony_svc }}"
        state: 'started'
        enabled: true
    - name: 'Copy standard config file for chrony time server'
      copy:
        dest: "{{ chrony_conf }}"
        src: '~/ansible/chrony/chrony.conf'
      notify: 'restart_chrony'
  handlers:
    - name: 'restart_chrony'
      service:
        name: "{{ chrony_svc }}"
        state: 'restarted'
EOF
```

2. execute

```
# check the deployment without affecting anything
ansible-playbook ~/chrony/chrony.yaml -C

# then actually execute
ansible-playbook ~/chrony/chrony.yaml
```

# managing users in ansible

## creating users

1. create a simple playbook using the `user` module

```
mkdir ~/ansible/user; cd ~/ansible/user
cat << EOF | tee user.yaml
- name: manage users
  hosts: all
  become: true
  gather_facts: flase
  tasks:
    - name: manage user
      user:
        name: ansible
        state: present
        shell: /bin/bash
EOF
ansible-playbook user.yaml
```

2. there are also variables that can be used... and you can provide a default value using the jinja2 templating function `default()`:

```
cat << EOF | tee user.yaml
- name: manage users
  hosts: all
  become: true
  gather_facts: flase
  tasks:
    - name: manage user
      user:
        name: "{{ user_account | default('ansible') }}"
        state: present
        shell: /bin/bash
EOF

#you can invoke this and provide the extra vars with:
ansible-playbook --extra-vars user_account=mary user.yaml
```

3. leveraging the `when` parameter and understanding the `user` module's `state` and `remove` parameters:
* `remove` will delete homedir as well
```
cat << EOF | tee user.yaml
- name: manage users
  hosts: all
  become: true
  gather_facts: flase
  tasks:
    - name: create user
      user:
        name: "{{ user_account | default('ansible') }}"
        state: present
        shell: /bin/bash
      when: user_create == 'yes'
    - name: delete user
      user:
        name: "{{ user_account | default('ansible') }}"
        state: absent
        remove: true
      when: user_create == 'no'
EOF
```

4. understand verbosity:

```
ansible-playbook --extra-vars user_account=mary --extra-vars user_create=yes user.yaml -vvv
#provides a lot of info
```

5. remove users:
```
#remove mary
ansible-playbook --extra-vars user_account=mary --extra-vars user_create=no user.yaml

#remove ansible, leveraging the filter that calls default()
ansible-playbook --extra-vars user_create=no user.yaml

#create ansible, leveraging the filter that calls default()
ansible-playbook --extra-vars user_create=yes user.yaml
```

## managing user passwords

* learn more about the `user` module's `password` and `update_password` parameters

1. create playbook

```
cd ~/ansible/user
cat << EOF | tee user.yaml
- name: manage users
  hosts: all
  become: true
  gather_facts: flase
  tasks:
    - name: create user
      user:
        name: "{{ user_account | default('ansible') }}"
        state: present
        shell: /bin/bash
        password: "{{ 'Password1' | password_hash('sha512') }}"
        update_password: on_create
      when: user_create == 'yes'
    - name: delete user
      user:
        name: "{{ user_account | default('ansible') }}"
        state: absent
        remove: true
      when: user_create == 'no'
EOF
```

## managing user auth keys

1. create a playbook and review the `user` module's parameters: `generate_ssh_key`, `ssh_key_type`, `ssh_key_file`

* note the use of `hosts: 'rhel'`
  * this will generate the key pair within `~/.ssh/` of vagrant
```
cd ~/ansible/user
cat << EOF | tee localuser.yaml
- name: 'Manage Local Account'
  hosts: 'rhel'
  become: true
  gather_facts: false
  tasks:
    - name: 'Manage User Account'
      user:
        name: '{{ user_account }}'
        state: 'present'
        generate_ssh_key: true
        ssh_key_type: 'ecdsa'
        ssh_key_file: '.ssh/id_ecdsa'
EOF

ansible-playbook localuser.yaml --extra-var user_account=$USER

#validate that the keys were generated
ll /home/vagrant/.ssh/id_ecdsa.pub
```

## authenticate with new key pair

1. use the `authorized_key` module to copy across a key

```
#insert a new task in user.yaml
- name: ssh auth to remote acct
  authorized_key:
    user: "{{ user_account | default('ansible') }}"
    state: present
    manage_dir: true
    key: "{{ lookup('file', '/home/vagrant/.ssh/id_ecdsa.pub') }}
  when: user_create == 'yes'
```

2. run
```
ansible-playbook user.yaml --extra-vars user_create=yes 
```

3. test auth with the key:
```
ssh -i ~/.ssh/id_ecdsa ansible@192.168.33.13
```

## using blocks and pushing a sudoers file to targets

* you can use blocks to designate conditionals across many tasks

1. generate the playbook

* note that the plays are targetting different hosts

```
cd ~/ansible/user
cat << EOF | tee user.yaml
- name: 'Manage Local Account'
  hosts: 'rhel'
  become: true
  gather_facts: false
  tasks:
    - name: 'Manage User Account'
      user:
        name: 'vagrant'
        state: 'present'
        generate_ssh_key: true
        ssh_key_type: 'ecdsa'
        ssh_key_file: '.ssh/id_ecdsa'

- name: 'Create and Manage Remote Ansible User'
  hosts: all
  become: true
  gather_facts: false
  tasks:
    - name: 'Create User Account, SSH Auth and Sudoers Entry'
      block:
        - name: 'Create Ansible User'
          user:
            name: 'ansible'
            state: 'present'
            shell: '/bin/bash'
            password: "{{ 'Password1' | password_hash('sha512') }}"
            update_password: 'on_create'       
        - name: 'Allow SSH Authentication via key for vagrant account to new remote account'
          authorized_key:
            user: 'ansible'
            state: 'present'
            manage_dir: true
            key: "{{ lookup('file', '/home/vagrant/.ssh/id_ecdsa.pub') }}"
        - name: 'Copy Sudoers file' 
          copy:
            dest: '/etc/sudoers.d/ansible'
            content: 'ansible ALL=(root) NOPASSWD: ALL'
      when: user_create == 'yes'        
    - name: 'Delete User Account'
      user:
        name: 'ansible'
        state: 'absent'
        remove: true
      when: user_create == 'no'
EOF
```

2. validate:

```
ansible-playbook --extra-vars user_create=yes user.yaml -C
```

3. run

```
ansible-playbook --extra-vars user_create=yes user.yaml -C
```

# ancillary ansible playbooks

## archiving files

1. archiving files
```
- name compress archvive of /etc
  archive:
    path: /etc
    dest: "/tmp/etc-{{ ansible_hostname }}.tgz"
```

## importing tasks
* dynamic (include): can define vars
```
- name: manage server backup
  hosts: all
  become: true
  gather_facts: true
  tasks:
  - include_tasks: backup.yaml
```

* static (import): just runs tasks (no import of vars)

## lab: create parent play, import children, understand scopes

1. Create env

```
#on rhel8 VM
cd ~/ansible
mkdir extra
```

2. create backup.yaml

* create just the task
```
cat << EOF | tee backup.yaml
- name: 'Backup /etc directory on system'
  archive:
    path: '/etc/'
    dest: "/tmp/etc-{{ ansible_hostname }}.tgz"
EOF
```

3. create archive.yaml with include_tasks
* create the (parent) playbook
```
cat << EOF | tee archive.yaml
- name: 'Backup and schedule backups'
  hosts: 'all'
  become: true
  gather_facts: true
  tasks:
  - include_tasks: backup.yaml
EOF
```

4. run the playbook and validate success
```
ansible-playbook archive.yaml

[vagrant@rhel8 extra]$ ansible-playbook archive.yaml
<snip>

TASK [include_tasks] *********************************************************************************************************************************************************************************
included: /home/vagrant/ansible/extra/backup.yaml for 192.168.33.13, 192.168.33.12, 192.168.33.11
<snip>
[vagrant@rhel8 extra]$ ls /tmp/etc*
/tmp/etc-rhel8.tgz
```

* NOTE: `import_tasks` simply adds the tasks to the playbook (similarly to just copy and pasting the tasks into the parent playbook)

5. create `schedule.yaml`
```
cat << EOF | tee schedule.yaml
- name: 'Scheduled backup of /etc'
  ansible.builtin.cron:
    name: 'backup /etc'
    weekday: '5'
    minute: '0'
    hour: '2'
    user: 'root'
    job: "tar -czf /tmp/etc-{{ ansible_hostname }}.tgz /etc"
    cron_file: etc_backup
EOF
```

6. include_tasks the `schedule.yaml` into the playbook archive.yaml
```
vim archive.yaml
#add the following line to the end
  - include_tasks: schedule.yaml
```

7. validate cron schedule
```
[vagrant@rhel8 extra]$ sudo cat /etc/cron.d/etc_backup
#Ansible: backup /etc
0 2 * * 5 root tar -czf /tmp/etc-rhel8.tgz /etc
```

## managing VDO storage
* this example will create multiple tasks and group them together in a playbook

### what is VDO?
* virtual data optimizer: available on RHEL and centos
  * VDO is a logical abstraction (API) of file systems

### letsss gooooo

* note that you must create a block device of 8GB as /dev/sdb, update the kernel, and reboot system.  I don't cover this, but will later.

1. install VDO to targets
```
cat << EOF | tee install.yaml
- name: install
  package:
    name:
      - vdo
      - kmod-kvdo
    state: latest  
EOF
```

2. start the vdo service
```
cat << EOF | tee service.yaml
- name: start
  service:
    name: vdo
    state: started
    enabled: true
EOF
```

3. create the vdo device
```
cat << EOF | tee createvdo.yaml
- name: create 
  vdo:
    name: vdo1
    state: present
    device: /dev/sdb
    logicalsize: 20G
```

4. create a file system in VDO
```
cat << EOF | tee fs.yaml
- name: format
  filesystem:
    type: xfs
    dev: /dev/mapper/vdo1
EOF
```

5. create the mountpoint dir
```
cat << EOF | tee mountpoint.yaml
- name: mountpoint
  file:
    path: /vdo1
    state: directory
EOF
```

6. mount the file system
```
cat << EOF | tee mount.yaml
- name: mount
  mount:
    path: /vdo1
    fstype: xfs
    state: mounted
    src: /dev/mapper/vdo1
    opts: defaults,x-systemd.requires=vdo.service
EOF
```

7. create playbook with include_tasks
```
cat << EOF | tee vdo.yaml
- name: vdo
  hosts: Redhat
  become: true
  gather_facts: false
  tasks:
  - include_tasks: install.yaml
  - include_tasks: service.yaml
  - include_tasks: createvdo.yaml
  - include_tasks: fs.yaml
  - include_tasks: mountpoint.yaml
  - include_tasks: mount.yaml
EOF
```

8. validate mounts
```
mount -t XFS
```

## importing playbooks

1. create a playbook that calls other playbooks

```
cat << EOF | tee p1.yaml
- import_playbook: archive.yaml
- import_playbook: vdo.yaml
EOF
```

2. execute
```
ansible-playbook p1.yaml
```

# jinja2 templating with ansible

* review template concepts
* variables
* filters
* if conditional
* for loops

## intro to templating

* a template is a basis
* data is loaded to a template
* jinja2 templating is used by ansible for templating
  * jinja2 templating is agnostic to context

### jinja2 template delimiters

1. loops and conditionals
```
{% for vhost in apache_vhosts %}
```

2. variables
```
{{ vhost.servername }}
```

3. comments
```
{# this is a comment #}
```

## ansible template module

* template module will generate dynamic configurations and then copy them to the target inventory hosts.
* templating occurs on the ansible controller, not on the target host.
* note that it is similar to the copy module, but it uses templating.

1. example of using templating module

* note that when no path is applied, templates can be stored either in the playbook directory, or a subdirectory of the playbook called `template`.
```
- name: template mariadb config file
  template:
    src: mariadb_conf.j2
    dest: /etc/my.cnf
    mode: 0664
    owner: root
    group: root
    force: yes
    backup: yes
```

2. ansible can run a validation on the ansible controller before copying the file to the dest:

```
- name: validate new sudoers with visudo, then copy
  src: sudoers
  dest: /etc/sudoers
  validate: /usr/sbin/visudo -cf %s
  backup: yes
```

3. you can use a loop to generate and copy multiple templates:

```
- name: copy multiple templated files
  template:
    src: "{{ item.src }}"
    dest: "{{ item.dst }}"
  loop:
    - { src: 'templates/myapp_cfg.j2}, dest: '/home/joe/myapp.cfg' }
    - { src: 'templates/index.html.j2}, dest: '/var/www/index.html' }
    - { src: 'templates/config_xml.j2}, dest: '/tmp/config.xml' }
```

## assigning vars within the inventory file

* you can assign vars in a few ways within the inventory file

1. which inventory file is being used?  Refer to [config hierarchy](#config-hierarchy)
```
[vagrant@rhel8 ansible]$ ansible-config dump | grep HOST_LIST
DEFAULT_HOST_LIST(/home/vagrant/.ansible.cfg) = ['/home/vagrant/inventory']
```

2. you can assign variables to hosts, such as `max_connections`

```
vim /home/vagrant/inventory
[rhel]
192.168.33.11

[stream]
192.168.33.12

[ubuntu]
192.168.33.13 max_connections=10000

[Redhat:children]
stream
rhel
```

3. you can assign variables to be available to hosts in groups:

* note that you can assign a host to multiple groups.
  
```
[rhel]
192.168.33.11 max_connections=10000

[stream]
192.168.33.12 max_connections=5000

[ubuntu]
192.168.33.13 max_connections=5000

[Redhat:children]
stream
rhel

[frontend:children]
rhel

[backend:children]
ubuntu
stream

[frontend:vars]
host_role='frontend'

[backend:vars]
host_role='backend'

[all:vars]
section_header='[global_config]'
```

4. review the inventory structure:
```
[vagrant@rhel8 ansible]$ ansible-inventory --graph
@all:
  |--@Redhat:
  |  |--@rhel:
  |  |  |--192.168.33.11
  |  |--@stream:
  |  |  |--192.168.33.12
  |--@backend:
  |  |--@ubuntu:
  |  |  |--192.168.33.13
  |--@frontend:
  |  |--@rhel:
  |  |  |--192.168.33.11
  |  |--@stream:
  |  |  |--192.168.33.12
  |--@ungrouped:
```

5. inventory host vars can also be configured via files within the `host_vars` and `group_vars` subdirectory (see [inventory vars](#implement-inventory-variables) section) that are at the same level as the inventory file

     a. you can view hostvars:
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
             "hostvars": {
                 "192.168.33.11": {
                     "ansible_connection": "local",
                     "apache_pkg": "httpd",
                     "apache_src": "httpd",
                     "chrony_conf": "/etc/chrony.conf",
                     "chrony_svc": "chronyd",
                     "host_role": "frontend",
                     "section_header": "[global_config]"
                 },
                 "192.168.33.12": {
                     "apache_pkg": "httpd",
                     "apache_src": "httpd",
                     "chrony_conf": "/etc/chrony.conf",
                     "chrony_svc": "chronyd",
                     "host_role": "frontend",
                     "section_header": "[global_config]"
                 },
                 "192.168.33.13": {
                     "apache_pkg": "apache2",
                     "apache_src": "apache2",
                     "chrony_conf": "/etc/chrony/chrony.conf",
                     "chrony_svc": "chrony",
                     "host_role": "backend",
                     "max_connections": 10000,
                     "section_header": "[global_config]"
                 }
             }
         },
     <snip>
     ```

## var substitution

1. connect to the rhel8 VM and create the lab structure

```
vagrant ssh rhel8
cd ~/ansible
mkdir -p ./module2/templates
```
   
2. create a template in the templates subdir
```
cd ~/ansible/module2
cat << EOF | tee ./templates/app_config.j2
#config for app that will render straight through
{# this comment does not get rendered in the destination file #}
{{ section_header }}
host_role={{ host_role }}
max_connections={{ max_connections }}
EOF
```

3. create the playbook that calls the template module

```
cd ~/ansible/module2
cat << EOF | tee ./template1.yml
- hosts: all
  tasks:
  - name: template generator for app.conf
    template:
      src: app_config.j2
      dest: /tmp/app.conf
EOF
```

4. run the template generator playbook
```
ansible-playbook template1.yml
```

5. validate the templating worked as expected
* note the host_role and max_connections matches the vars that were set
```
# on rhel8
[vagrant@rhel8 module2]$ cat /tmp/app.conf
#config for app that will render straight through
[global_config]
host_role= frontend
max_connections= 10000

# on ubuntu
vagrant@ubuntu:~$ cat /tmp/app.conf
#config for app that will render straight through
[global_config]
host_role= backend
max_connections= 5000
```

* note that ansible facts are also accessible, such as...
```
{{ ansible_facts.eth1.ipv4.address }}
```


