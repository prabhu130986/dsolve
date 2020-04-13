---
layout: post
title:  "Define variables in ansible playbook"
author: Prabhu
categories: [ Ansible ]
image: assets/images/ansible.jpg
featured: false
---

There are different ways variables can be defined in ansible playbook. Let us see each of them with examples.

## Inside roles

Variables can be defined with the roles inside vars or defaults folder. Here is the folder structure and example. These variables can be used only by that particular role.

```
.
├── demo.yml
└── roles
    └── common
        ├── defaults
        │   └── main.yml    #  <-- default lower priority variables for this role
        ├── files
        ├── handlers
        │   └── main.yml
        ├── meta
        │   └── main.yml
        ├── README.md
        ├── tasks
        │   └── main.yml
        ├── templates
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml    #  <-- variables associated with this role
```

Example how to define

```
---
package_name: httpd
http_port: 8080
```

## While calling role

Variables can be defined while calling a role from the main playbook.

```
---
- hosts: apache
  tasks:
    - include_role:
        name: install_package
      vars:
        package_name: httpd
        http_port: 8080
```

## In main playbook

Variables can also be defined in the main playbook. This will be used by all tasks and roles in that playbook.

```
- hosts: webserver
  vars:
    http_port: 80
  vars_files:
    - variables.yml
```

Using include_vars

```
- hosts: all
  tasks:
    - name: Set global variables
      include_vars: "global_vars.yml"
```

## In group_vars

Variables defined in group_vars can be used by all tasks that gets executed on that particular hostgroup

```
---
# file: inventories/group_vars/appservers
package_name: tomcat
http_port: 9001
```

```
---
# file: inventories/group_vars/webservers
package_name: httpd
http_port: 8080
```

## In host_vars

This is similar to group_vars, variables defined under particular hostname.

```
---
# file: inventories/host_vars/server1
package_name: httpd
http_port: 8080
```

## In inventory file

Variables can directly be defined in the inventory file also. As below

```
[appserver]
server1 http_port=8001 admin_username=server1adm
server2 http_port=8002 admin_username=server2adm
```

## Using extra-vars in command-line

You can pass variables from command-line while you initiate playbook run.

```
ansible-playbook package.yml --extra-vars "package_name=httpd http_port: 8080"
```

## Prompt

If you want to wish to provide value for a variable interactively, you can use var_prompt.

```
---
- hosts: all
  vars_prompt:

    - name: admpassword
      prompt: "Please enter admin password"
```

# How to use variable in tasks

Finally, how to use that variables that are defined using one of above methods in the tasks.

```
- name: Install the Apache package
  yum:
    name: {{ package_name }}
    state: present
```


#### Start using one of the methods based on the scenario and complexity of the requirement !!! Cheers !!!