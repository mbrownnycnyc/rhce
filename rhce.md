# https://app.pluralsight.com/explore/certifications/topics/linux?trackId=43182a5e-279a-4ef6-808c-c3f62d01ed57&examPrepId=0e4f07d3-5443-4d9c-9331-2a886c87bdb6

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
```

5. run vagrant up, which will download all three distros, and create VMs.
* this will take a while (an hour or so) and may require initial interaction.
```
cd ~/vagrant/ansible
vagrant up
# you may need to change the vm.box for centos to `"bento/centos-stream-8"`
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
  * 