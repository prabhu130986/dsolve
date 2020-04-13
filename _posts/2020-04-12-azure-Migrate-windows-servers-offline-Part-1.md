---
layout: post
title:  "Migrate on-premise windows servers to Azure using offline mode - Part 1"
author: Prabhu
categories: [ Azure ]
image: assets/images/azure.jpg
featured: true
---

When migrating workload from on-premise to Azure there are several factors which need to be considered before choosing the best method for migration. What type of connectivity is there between on-premise and Azure region, bandwidth of the connectivity, type of workload, criticality of workload etc.

Here we will see a scenario as follows
* There is no connectivity between on-premise and Azure region
* Server that need to be migrated has only static contents
* Customer doesn't care about data sync as long as whatever is there in the server at the time of migration has been retained in Azure
* Size of data is in Terrabytes

For this scenario the best possible solution would be to convert the on-premise windows servers to VHD, ship them to Azure region using databox and then restore the VHD in Azure.

Let us have the complete steps in two parts. In this part one, we will cover how to convert the on-premise windows servers to VHD

## Prepare the server

In order to boot the on-premise windows servers in Azure certain steps need to be preformed on them. Else the captured VHD wont boot in Azure or we will not be able to connect them remotely even if they boot up. Below are the prep steps that need to be performed.

1. Download and install the security patch for CredSSP Remote Code Evaluation Vulnerability. This is by default disabled in all Windows OS. Download the respective patch based on the OS version and install them from https://portal.msrc.microsoft.com/en-us/security-guidance/advisory/CVE-2018-0886
2. Run the below registry add command
   ```
   REG ADD HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters\ /v AllowEncryptionOracle /t REG_DWORD /d 2
   ```
3. Download the Azure windows VM agent and install on the server. You can download it from https://go.microsoft.com/fwlink/?LinkID=394789

Once done with the above steps, proceed to capture the server as VHD.

## Capture server drives as VHD

There are several tools available to capture servers as VHD and there are few provided by Microsoft itself. We will use one such tool called disk2vhd. This is a portable tool which can be copied to the respective servers and initiate the capture. This tool can be used to capture physical servers and VMWare Vcenter VMs. Hyper-v VMs will already be in VHD or VHDX format so no need to perform this step on them.
The steps are as below: 

1. Download disk2vhd tool from https://docs.microsoft.com/en-us/sysinternals/downloads/disk2vhd and extract and trigger disk2vhd.exe
2. Tool will list down the drives which need to be captured and the destination file name for VHD to be stored. You can capture individual drives as individual VHD or select all drives and capture as one VHD
3. You can choose to store the final VHD locally in one of the drive or mount a network drive and store it there.

Once this process is done, you can see the VHD file stored in the chosen location. Follow further steps to convert the VHD to fixed type

## Convert VHD to fixed type

The captured VHD will be in dynamic VHD type. But Azure will accept VHD only that are fixed type. So the VHD need to be converted from dynamic to fixed. The steps are as below.

1. Login to a windows server with hyper-v role enabled. This can be any ordinary windows server
2. Copy the captured VHD to the server
3. Check the type of the VHD using the powershell command. The output will show the VHDType as Dynamic
   ```
   Get-VHD .\dynamic\WIN-2k12-os.VHD
   ```
4. Execute the below powershell command to convert it to Fixed type
   ```
   Convert-VHD -Path .\dynamic\WIN-2k12-os.VHD -DestinationPath .\fixed\WIN-2k12-fixed-os.vhd -VHDType Fixed
   ```
5. Check and confirm the VHD type after conversion
   ```
   Get-VHD .\fixed\WIN-2k12-fixed-os.vhd
   ```

## Ship the VHD to Azure

Azure provdies option to ship large data offline to Azure region using Data box or Data box disk. Data box supports upto 80 TB to data to be shipped where as Data box disk supports 8 TB. It also has one more option called Data box heavy which is capable of 1 Petabye of data. Based on the size, one of the above option can be chosen.

There are 4 steps involved in this.

1. Order
2. Setup
3. Connect & Copy
4. Return & Upload

Below Microsoft link has the step by step tutorial which covers all the above tasks.
https://docs.microsoft.com/en-us/azure/databox/data-box-deploy-ordered

#### So far we have covered the steps to capture the on-premise windows servers, convert them and ship them to Azure. In [Part 2]({{ site.url }}{% link _posts/2020-04-12-azure-Migrate-windows-servers-offline-Part-2.md %}) we will see how to provision Azure VMs using the VHD that are shipped !!! Cheers !!!