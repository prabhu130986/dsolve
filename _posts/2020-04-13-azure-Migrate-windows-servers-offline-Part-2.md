---
layout: post
title:  "Migrate on-premise windows servers to Azure using offline mode - Part 2"
author: Prabhu
categories: [ Ansible ]
image: assets/images/azure.jpg
featured: true
---

In this [Part 1]({{ site.url }}{% link _posts/2020-04-12-azure-Migrate-windows-servers-offline-Part-1.md %}) post we saw how to capture and convert on-premise windows servers to VHD and send it to Azure region using offline mode. In this post let us see how to convert the VHD to Azure managed disk and provision VM using that.

Let us consider that the following tasks have been performed and verified
* On-premise windows servers are captured as VHD
* Dynamic VHD type is converted to Fixed VHD type
* VHD is shipped to Azure region and uploaded into a storage account

Now let us proceed further.

## Create a Managed disk from the fixed VHD

Let us use Azure powershell modules to perform the tasks. First let us define the storage account related information in the variables. Let us consider we are going to create Premium managed disk.

```
$storageType = 'Premium_LRS'
$sourceVHDURI = 'https://************.blob.core.windows.net/vhd/WIN-2k12-fixed-os.vhd'
$storageAccountId = '/subscriptions/********-****-****-****-*********/resourceGroups/migrate-rg/providers/Microsoft.Storage/storageAccounts/vhdosdisks'
```

Using the below commands create disk configuration and create managed disk.

```
$diskConfig = New-AzDiskConfig -AccountType $storageType -Location $location -CreateOption Import -StorageAccountId $storageAccountId -SourceUri $sourceVHDURI

$osDisk = New-AzDisk -Disk $diskConfig -ResourceGroupName $resourceGroupName -DiskName $diskName
```

This will create a new managed disk in the specified resource group.

## Create Network interface for VM

Network interface for a VM has to be created before provisioning a VM. Let us assume we assign a Public IP for that VM. Use the below command to create a public ip.

```
$pip = New-AzPublicIpAddress `
   -Name $ipName -ResourceGroupName $resourceGroupName `
   -Location $location `
   -AllocationMethod Dynamic
```

Let us assume a virtual network and subnet are already created. Now using that information let us create a network interface using the below commands.

```
$vn = Get-AzVirtualNetwork -Name $vnetname

$sn = Get-AzVirtualNetworkSubnetConfig -Name migration-subnet -VirtualNetwork $vn

$nic = New-AzNetworkInterface -Name $nicName `
   -ResourceGroupName $resourceGroupName `
   -Location $location -SubnetId $sn.Id `
   -PublicIpAddressId $pip.Id
```

## Provision VM using the managed disk

Let us define the VM size now.

```
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize "Standard_A2"
```

Now let us add the network interface that was created earlier to the VM.

```
$vm = Add-AzVMNetworkInterface -VM $vmConfig -Id $nic.Id  
```

Let us set the OS disk configuration settings using the below command

```
$vm = Set-AzVMOSDisk -VM $vm -ManagedDiskId $osDisk.Id -StorageAccountType Standard_LRS `
    -DiskSizeInGB $diskSize -CreateOption Attach -Windows
```

Here we are passing the os disk id for the disk which was created earlier.

Now provision VM using the below command.

```
New-AzVM -VM $vm -ResourceGroupName $resourceGroupName -Location $location
```

Now you will be able to see the VM is provisioned in the given resource group and location. Since we installed the Azure Windows VM agent in the server before capturing VHD, Azure automatically detects the VM after provisioning. 

Now you can login to the VM using the credential you used to login in on-premise.

#### The steps in this post can be used individually to create a VM using Azure powershell modules or part of a migration process.  !!! Cheers !!!