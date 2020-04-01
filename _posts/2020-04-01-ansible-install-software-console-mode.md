---
layout: post
title:  "Use Ansible to install softwares in console mode"
author: Prabhu
categories: [ Ansible ]
image: assets/images/ansible.jpg
featured: true
---

There are some software which we have to install in linux operating system using console mode. That means, the installation wizard will prompt the user to input some values to proceed the installation. Ansible can be used to automate this process too.

## expect module

expect module can be used to trigger a command, read the prompt and pass the response. 

### Pre-requisites

To use expect module below packages need to be installed on the target host where the task has to be performed

* python >= 2.6
* pexpect >= 3.3

### Examples

Below example is to set password for a linux user

```
- name: Set password for user
  expect:
    command: passwd ansibleadm
    responses:
      (?i)password: "Pa$$word"
  # you don't want to show passwords in your logs
  no_log: true
```

This example show how to install a software in console mode

```
- name: Install software
  expect:
    command: sh setup.sh
    responses:
      (.*)Enter installation type:(.*): 'custom'    
      (.*)Enter installation directory:(.*): '/opt'   
      (.*)Do you want to continue\? \[y\/N\](.*): 'y'
      (.*)Do you accept the license agreement(.*): 'y'
```


#### Ansible gives us options to perform tasks that does not support silent mode. This give us more scope to perform wide range of operations !!! Cheers !!!