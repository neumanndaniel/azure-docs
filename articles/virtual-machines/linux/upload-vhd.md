---
title: Upload or copy a custom Linux VM with Azure CLI 2.0 | Microsoft Docs
description: Upload or copy a customized virtual machine using the Resource Manager deployment model and the Azure CLI 2.0
services: virtual-machines-linux
documentationcenter: ''
author: cynthn
manager: jeconnoc
editor: tysonn
tags: azure-resource-manager

ms.assetid: a8c7818f-eb65-409e-aa91-ce5ae975c564
ms.service: virtual-machines-linux
ms.workload: infrastructure-services
ms.tgt_pltfrm: vm-linux
ms.devlang: azurecli
ms.topic: article
ms.date: 07/06/2017
ms.author: cynthn

---
# Create a Linux VM from custom disk with the Azure CLI 2.0

<!-- rename to create-vm-specialized -->

This article shows you how to upload a customized virtual hard disk (VHD) or copy a an existing VHD in Azure and create new Linux virtual machines (VMs) from the custom disk. You can install and configure a Linux distro to your requirements and then use that VHD to quickly create a new Azure virtual machine.

If you want to create multiple VMs from your customized disk, you should create an image from your VM or VHD. For more information, see [Create a custom image of an Azure VM using the CLI](tutorial-custom-images.md).

You have two options:
* [Upload a VHD](#option-1-upload-a-specialized-vhd)
* [Copy an existing Azure VM](#option-2-copy-an-existing-azure-vm)

## Quick commands

When creating a new VM using [az vm create](/cli/azure/vm#az_vm_create) from a customized or specialized disk you **attach** the disk (--attach-os-disk) instead of specifying a custom or marketplace image (--image). The following example creates a VM named *myVM* using the managed disk named *myManagedDisk* created from your customized VHD:

```azurecli
az vm create --resource-group myResourceGroup --location eastus --name myVM \
   --os-type linux --attach-os-disk myManagedDisk
```

## Requirements
To complete the following steps, you need:

* A Linux virtual machine that has been prepared for use in Azure. The [Prepare the VM](#prepare-the-vm) section of this article covers how to find distro specific information on installing the Azure Linux Agent (waagent) which is required for the VM to work properly in Azure and for you to be able to connect to it using SSH.
* The VHD file from an existing [Azure-endorsed Linux distribution](endorsed-distros.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json) (or see [information for non-endorsed distributions](create-upload-generic.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)) to a virtual disk in the VHD format. Multiple tools exist to create a VM and VHD:
  * Install and configure [QEMU](https://en.wikibooks.org/wiki/QEMU/Installing_QEMU) or [KVM](http://www.linux-kvm.org/page/RunningKVM), taking care to use VHD as your image format. If needed, you can [convert an image](https://en.wikibooks.org/wiki/QEMU/Images#Converting_image_formats) using **qemu-img convert**.
  * You can also use Hyper-V [on Windows 10](https://msdn.microsoft.com/virtualization/hyperv_on_windows/quick_start/walkthrough_install) or [on Windows Server 2012/2012 R2](https://technet.microsoft.com/library/hh846766.aspx).

> [!NOTE]
> The newer VHDX format is not supported in Azure. When you create a VM, specify VHD as the format. If needed, you can convert VHDX disks to VHD using [qemu-img convert](https://en.wikibooks.org/wiki/QEMU/Images#Converting_image_formats) or the [Convert-VHD](https://technet.microsoft.com/library/hh848454.aspx) PowerShell cmdlet. Further, Azure does not support uploading dynamic VHDs, so you need to convert such disks to static VHDs before uploading. You can use tools such as [Azure VHD Utilities for GO](https://github.com/Microsoft/azure-vhd-utils-for-go) to convert dynamic disks during the process of uploading to Azure.
> 
> 


* Make sure that you have the latest [Azure CLI 2.0](/cli/azure/install-az-cli2) installed and logged in to an Azure account using [az login](/cli/azure/reference-index#az_login).

In the following examples, replace example parameter names with your own values. Example parameter names included *myResourceGroup*, *mystorageaccount*, and *mydisks*.

<a id="prepimage"> </a>

## Prepare the VM

Azure supports various Linux distributions (see [Endorsed Distributions](endorsed-distros.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)). The following articles guide you through how to prepare the various Linux distributions that are supported on Azure:

* [CentOS-based Distributions](create-upload-centos.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Debian Linux](debian-create-upload-vhd.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Oracle Linux](oracle-create-upload-vhd.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Red Hat Enterprise Linux](redhat-create-upload-vhd.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [SLES & openSUSE](suse-create-upload-vhd.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Ubuntu](create-upload-ubuntu.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)
* [Other - Non-Endorsed Distributions](create-upload-generic.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json)

Also see the [Linux Installation Notes](create-upload-generic.md#general-linux-installation-notes) for more general tips on preparing Linux images for Azure.

> [!NOTE]
> The [Azure platform SLA](https://azure.microsoft.com/support/legal/sla/virtual-machines/) applies to VMs running Linux only when one of the endorsed distributions is used with the configuration details as specified under 'Supported Versions' in [Linux on Azure-Endorsed Distributions](endorsed-distros.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json).
> 
> 

## Option 1: Upload a VHD

You can upload a customized VHD that you have running on a local machine or that you exported from another cloud. To use the VHD to create a new Azure VM, you need to upload the VHD to a storage account and create a managed disk from the VHD. 

### Create a resource group

Before uploading your custom disk and creating VMs, you first need to create a resource group with [az group create](/cli/azure/group#az_group_create).

The following example creates a resource group named *myResourceGroup* in the *eastus* location:
[Azure Managed Disks overview](../windows/managed-disks-overview.md)
```azurecli
az group create \
    --name myResourceGroup \
	--location eastus
```

### Create a storage account

Create a storage account for your custom disk and VMs with [az storage account create](/cli/azure/storage/account#az_storage_account_create). 

The following example creates a storage account named *mystorageaccount* in the resource group previously created:

```azurecli
az storage account create \
    --resource-group myResourceGroup \
	--location eastus \
	--name mystorageaccount \
	--kind Storage \
	--sku Standard_LRS
```

### List storage account keys
Azure generates two 512-bit access keys for each storage account. These access keys are used when authenticating to the storage account, like carrying out write operations. Read more about [managing access to storage here](../../storage/common/storage-create-storage-account.md#manage-your-storage-account). You view the access keys with [az storage account keys list](/cli/azure/storage/account/keys#az_storage_account_keys_list).

View the access keys for the storage account you created:

```azurecli
az storage account keys list \
    --resource-group myResourceGroup \
	--account-name mystorageaccount
```

The output is similar to:

```azurecli
info:    Executing command storage account keys list
+ Getting storage account keys
data:    Name  Key                                                                                       Permissions
data:    ----  ----------------------------------------------------------------------------------------  -----------
data:    key1  d4XAvZzlGAgWdvhlWfkZ9q4k9bYZkXkuPCJ15NTsQOeDeowCDAdB80r9zA/tUINApdSGQ94H9zkszYyxpe8erw==  Full
data:    key2  Ww0T7g4UyYLaBnLYcxIOTVziGAAHvU+wpwuPvK4ZG0CDFwu/mAxS/YYvAQGHocq1w7/3HcalbnfxtFdqoXOw8g==  Full
info:    storage account keys list command OK
```
Make a note of **key1** as you will use it to interact with your storage account in the next steps.

### Create a storage container
In the same way that you create different directories to logically organize your local file system, you create containers within a storage account to organize your disks. A storage account can contain any number of containers. Create a container with [az storage container create](/cli/azure/storage/container#az_storage_container_create).

The following example creates a container named *mydisks*:

```azurecli
az storage container create \
    --account-name mystorageaccount \
    --name mydisks
```

### Upload the VHD
Now upload your custom disk with [az storage blob upload](/cli/azure/storage/blob#az_storage_blob_upload). You upload and store your custom disk as a page blob.

Specify your access key, the container you created in the previous step, and then the path to the custom disk on your local computer:

```azurecli
az storage blob upload --account-name mystorageaccount \
    --account-key key1 \
	--container-name mydisks \
	--type page \
    --file /path/to/disk/mydisk.vhd \
	--name myDisk.vhd
```
Uploading the VHD may take a while.

### Create a managed disk


Create a managed disk from the VHD using [az disk create](/cli/azure/disk#az_disk_create). The following example creates a managed disk named *myManagedDisk* from the VHD you uploaded to your named storage account and container:

```azurecli
az disk create \
    --resource-group myResourceGroup \
	--name myManagedDisk \
  --source https://mystorageaccount.blob.core.windows.net/mydisks/myDisk.vhd
```
## Option 2: Copy an existing VM

You can also create the customized VM in Azure and then copy the OS disk and attach it to a new VM to create another copy. This is fine for testing, but if you want to use an existing Azure VM as the model for multiple new VMs, you really should create an **image** instead. For more information about creating an image from an existing Azure VM, see [Create a custom image of an Azure VM using the CLI](tutorial-custom-images.md)

### Create a snapshot

This example creates a snapshot of a VM named *myVM* in resource group *myResourceGroup* and creates a snapshot named *osDiskSnapshot*.

```azure-cli
osDiskId=$(az vm show -g myResourceGroup -n myVM --query "storageProfile.osDisk.managedDisk.id" -o tsv)
az snapshot create \
    -g myResourceGroup \
	--source "$osDiskId" \
	--name osDiskSnapshot
```
###  Create the managed disk

Create a new managed disk from the snapshot.

Get the ID of the snapshot. In this example, the snapshot is named *osDiskSnapshot* and it is in the *myResourceGroup* resource group.

```azure-cli
snapshotId=$(az snapshot show --name osDiskSnapshot --resource-group myResourceGroup --query [id] -o tsv)
```

Create the managed disk. In this example, we will create a managed disk named *myManagedDisk* from our snapshot, that is 128GB in size in standard storage.

```azure-cli
az disk create \
    --resource-group myResourceGroup \
	--name myManagedDisk \
	--sku Standard_LRS \
	--size-gb 128 \
	--source $snapshotId
```

## Create the VM

Now, create your VM with [az vm create](/cli/azure/vm#az_vm_create) and attach (--attach-os-disk) the managed disk as the OS disk. The following example creates a VM named *myNewVM* using the managed disk created from your uploaded VHD:

```azurecli
az vm create \
    --resource-group myResourceGroup \
    --location eastus \
    --name myNewVM \
	--os-type linux \
    --attach-os-disk myManagedDisk
```

You should be able to SSH into the VM using the credentials from the source VM. 

## Next steps
After you have prepared and uploaded your custom virtual disk, you can read more about [using Resource Manager and templates](../../azure-resource-manager/resource-group-overview.md). You may also want to [add a data disk](add-disk.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json) to your new VMs. If you have applications running on your VMs that you need to access, be sure to [open ports and endpoints](nsg-quickstart.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json).

