---
layout: post
title:  "Ansible - How to connect to Windows host"
author: Prabhu
categories: [ Ansible ]
image: assets/images/ansible.jpg
featured: false
---

Ansible is not only for automating Linux hosts but also Windows hosts. In order to connect to a windows target host some additional steps need to be performed. You can follow the below steps to configure windows host connection

## Install WinRM 

Windows Remote Management - WinRM need to be configured on the target windows hosts so that Ansible control machine can establish connection with it.

Ansible has given a powershell script which can be used to perform this. This script basically created a self-signed certificate for securing the connection and enable the WinRM ports on the host machine. You can execute the below command on the windows host.

```
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)

powershell.exe -ExecutionPolicy ByPass -File $file
```

## Install Pywinrm

Pywinrm package need to be installed on Ansible control machine as this is not enabled by default. Use this command to install the package.

```
pip install pywinrm
```

## Update inventory

Host inventory need to be updated with the below options to for enabling connection.

```
[win-host]
10.128.0.10 

[win-host:vars]
ansible_user=admin
ansible_password=password
ansible_connection=winrm
ansible_winrm_server_cert_validation=ignore
```

## Test Connection

Finally you can test the connection using ansible adhoc command and win_ping module

```
ansible win-host -i hosts -m win_ping

10.128.0.10 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

```


#### When compared to Linux hosts the tasks which can be performed on Windows hosts are limited. But still as modules are getting added the scope will increse too !!! Cheers !!!