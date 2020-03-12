---
layout: post
title:  "Ansible Adhoc Commands"
author: Prabhu
categories: [ Ansible ]
image: assets/images/ansible.jpg
featured: true
---

Ansible adhoc commands are great to use when you want to perform a rare task or test a task. Below are some modules which we can test using adhoc commands

## Syntax

Below is the syntax to use adhoc commands

```
ansible [pattern] -m [module] -a "[module options]"
```

Where pattern is the host group on which you have to perform the task. module and module options are tasks which you have to perform.

### Ping Module

This command uses ping module to verify connectivity to the target server

```
ansible localhost -m ping
```

The output for this command will be 

```
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Command Module

This command is used to reboot a hostgroup using command module

```
ansible webservers -a "/sbin/reboot"
```

Here you can notice that we have not used -m option. The default module which ansible uses is command module. So in this case "/sbin/reboot" command is triggered via command module without calling it explicitly.

### Copy Module

This command is used to copy files to target machines 

```
ansible appservers -m copy -a "src=/tmp/config dest=/opt/config"
```

### File Module

These commands uses file modle to manage files and create folder

```
ansible webservers -m file -a "dest=/opt/startup.sh mode=744"
```

```
ansible webservers -m file -a "dest=/opt/httpd mode=755 owner=httpd group=httpd state=directory"
```

### Yum Module

This command uses yum module to install a package

```
ansible webservers -m yum -a "name=apache state=present"
```

### Service Module

This command uses service module to manage services

```
ansible webservers -m service -a "name=httpd state=restarted"
```


#### This way you can use any module in the adhoc command with right options !!! Cheers !!!