---
title: Add a VM image to Azure Stack | Microsoft Docs
description: Learn how to add or remove a VM image to Azure Stack.
services: azure-stack
documentationcenter: ''
author: mattbriggs
manager: femila
editor: ''

ms.service: azure-stack
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: PowerShell
ms.topic: conceptual
ms.date: 07/23/2019
ms.author: mabrigg
ms.reviewer: kivenkat
ms.lastreviewed: 06/08/2018

---
# Add a VM image to Azure Stack

*Applies to: Azure Stack integrated systems and Azure Stack Development Kit*

In Azure Stack, you can add a virtual machine (VM) image to the marketplace to make  available to your users. Images are added by using Azure Resource Manager templates for Azure Stack. You can also add VM images to the Azure Marketplace UI as a Marketplace item using the administrator portal or Windows PowerShell. Use either an image from the global Azure Marketplace or your own custom VM image.

## Add a VM image through the portal

> [!NOTE]  
> With this method, you must create the Marketplace item separately.

Images must be able to be referenced by a blob storage URI. Prepare a Windows or Linux operating system image in VHD format (not VHDX), and then upload the image to a storage account in Azure or Azure Stack. If your image is already uploaded to the blob storage in Azure or Azure Stack, you can skip step 1.

1. [Upload a Windows VM image to Azure for Resource Manager deployments](https://azure.microsoft.com/documentation/articles/virtual-machines-windows-upload-image/) or, for a Linux image, follow the instructions described in [Deploy Linux VMs on Azure Stack](azure-stack-linux.md). Before you upload the image, it's important to consider the following factors:

   - Azure Stack only supports generation one (1) VM in the fixed disk VHD format. The fixed-format structures the logical disk linearly within the file, so that disk offset X is stored at blob offset X. A small footer at the end of the blob describes the properties of the VHD. To confirm if your disk is fixed, use the [Get-VHD](https://docs.microsoft.com/powershell/module/hyper-v/get-vhd?view=win10-ps) PowerShell command.  

     > [!IMPORTANT]  
     >  Azure Stack doesn't support dynamic disk VHDs. Resizing a dynamic disk that's attached to a VM will leave the VM in a failed state. To mitigate this issue, delete the VM without deleting the VM's disk, a VHD blob in a storage account. Then, convert the VHD from a dynamic disk to a fixed disk and re-create the VM.

   - It's more efficient to upload an image to Azure Stack blob storage than to Azure blob storage because it takes less time to push the image to the Azure Stack image repository.

   - When you upload the [Windows VM image](https://azure.microsoft.com/documentation/articles/virtual-machines-windows-upload-image/), make sure to switch the **Login to Azure** step with the [Configure the Azure Stack operator's PowerShell environment](azure-stack-powershell-configure-admin.md) step.  

   - Make a note of the blob storage URI where you upload the image. The blob storage URI has the following format:
     *&lt;storageAccount&gt;/&lt;blobContainer&gt;/&lt;targetVHDName&gt;*.vhd.

   - To make the blob anonymously accessible, go to the storage account blob container where the VM image VHD was uploaded. Select **Blob**, and then select **Access policy**. Optionally, you can generate a shared access signature for the container, and include it as part of the blob URI. This step makes sure the blob is available to be used. If the blob isn't anonymously accessible, the VM image will be created in a failed state.

     ![Go to storage account blobs](./media/azure-stack-add-vm-image/image1.png)

     ![Set blob access to public](./media/azure-stack-add-vm-image/image2.png)

2. Sign in to Azure Stack as operator. In the menu, select **All services** > **Images** under **Compute** > **Add**.

3. Under **Create image**, enter the Name, Subscription, Resource Group, Location, OS disk, OS type, storage blob URI, Account type, and Host caching. Then, select **Create** to begin creating the VM image.

   ![Begin to create the image](./media/azure-stack-add-vm-image/image4.png)

   When the image is successfully created, the VM image status changes to **Succeeded**.

4. To make the VM image more readily available for user consumption in the UI, it's a good idea to [create a Marketplace item](azure-stack-create-and-publish-marketplace-item.md).

## Remove a VM image through the portal

1. Open the administrator portal at [https://adminportal.local.azurestack.external](https://adminportal.local.azurestack.external).

2. Select **Marketplace management**, and then select the VM you would like to delete.

3. Click **Delete**.

## Add a VM image to the Marketplace by using PowerShell

> [!Note]  
> When you add an image it'll only be available for Azure Resource Manager-based templates and PowerShell deployments. To make an image available to your users as a marketplace item, publish the marketplace item using the steps in this article: [Create and publish a Marketplace item](azure-stack-create-and-publish-marketplace-item.md)

1. [Install PowerShell for Azure Stack](azure-stack-powershell-install.md).  

2. Sign in to Azure Stack as an operator. For instructions, see [Sign in to Azure Stack as an operator](azure-stack-powershell-configure-admin.md).

3. Open PowerShell with an elevated prompt, and run:

   ```powershell
    Add-AzsPlatformimage -publisher "<publisher>" `
      -offer "<offer>" `
      -sku "<sku>" `
      -version "<#.#.#>" `
      -OSType "<ostype>" `
      -OSUri "<osuri>"
   ```

   The **Add-AzsPlatformimage** cmdlet specifies values used by the Azure Resource Manager templates to reference the VM image. The values include:
   - **publisher**  
     For example: `Canonical`  
     The publisher name segment of the VM image that users use when they deploy the image. An example is **Microsoft**. Don't include a space or other special characters in this field.  
   - **offer**  
     For example: `UbuntuServer`  
     The offer name segment of the VM image that users use when they deploy the VM image. An example is **WindowsServer**. Don't include a space or other special characters in this field.  
   - **sku**  
     For example: `14.04.3-LTS`  
     The SKU name segment of the VM Image that users use when they deploy the VM image. An example is **Datacenter2016**. Don't include a space or other special characters in this field.  
   - **version**  
     For example: `1.0.0`  
     The version of the VM Image that users use when they deploy the VM image. This version is in the format *\#.\#.\#*. An example is **1.0.0**. Don't include a space or other special characters in this field.  
   - **osType**  
     For example: `Linux`  
     The osType of the image must be either **Windows** or **Linux**.  
   - **OSUri**  
     For example: `https://storageaccount.blob.core.windows.net/vhds/Ubuntu1404.vhd`  
     You can specify a blob storage URI for an `osDisk`.  

     For more info, see the PowerShell reference for the [Add-AzsPlatformimage](https://docs.microsoft.com/powershell/module/azs.compute.admin/add-azsplatformimage) cmdlet and the [New-DataDiskObject](https://docs.microsoft.com/powershell/module/Azs.Compute.Admin/New-DataDiskObject) cmdlet.

## Add a custom VM image to the Marketplace by using PowerShell
 
1. [Install PowerShell for Azure Stack](azure-stack-powershell-install.md).

   ```powershell
    # Create the Azure Stack operator's Azure Resource Manager environment by using the following cmdlet:
    Add-AzureRMEnvironment -Name "AzureStackAdmin" -ArmEndpoint "https://adminmanagement.local.azurestack.external" `
      -AzureKeyVaultDnsSuffix adminvault.local.azurestack.external `
      -AzureKeyVaultServiceEndpointResourceId https://adminvault.local.azurestack.external

    $TenantID = Get-AzsDirectoryTenantId `
      -AADTenantName "<myDirectoryTenantName>.onmicrosoft.com" `
      -EnvironmentName AzureStackAdmin

    Add-AzureRmAccount `
      -EnvironmentName "AzureStackAdmin" `
      -TenantId $TenantID
   ```

2. If using **Active Directory Federation Services (AD FS)**, use the following cmdlet:

   ```powershell
   # For Azure Stack Development Kit, this value is set to https://adminmanagement.local.azurestack.external. To get this value for Azure Stack integrated systems, contact your service provider.
   $ArmEndpoint = "<Resource Manager endpoint for your environment>"

   # For Azure Stack Development Kit, this value is set to https://graph.local.azurestack.external/. To get this value for Azure Stack integrated systems, contact your service provider.
   $GraphAudience = "<GraphAudience endpoint for your environment>"

   # Create the Azure Stack operator's Azure Resource Manager environment by using the following cmdlet:
    Add-AzureRMEnvironment -Name "AzureStackAdmin" -ArmEndpoint "https://adminmanagement.local.azurestack.external" `
      -AzureKeyVaultDnsSuffix adminvault.local.azurestack.external `
      -AzureKeyVaultServiceEndpointResourceId https://adminvault.local.azurestack.external
    ```

3. Sign in to Azure Stack as an operator. For instructions, see [Sign in to Azure Stack as an operator](azure-stack-powershell-configure-admin.md).

4. Create a storage account in global Azure or Azure Stack to store your custom VM image. For instructions see [Quickstart: Upload, download, and list blobs using the Azure portal](https://docs.microsoft.com/azure/storage/blobs/storage-quickstart-blobs-portal).

5. Prepare a Windows or Linux operating system image in VHD format (not VHDX), upload the image to your storage account, and get the URI where the VM image can be retrieved by PowerShell.  

   ```powershell
    Add-AzureRmAccount `
      -EnvironmentName "AzureStackAdmin" `
      -TenantId $TenantID
   ```

6. (Optionally) You can upload an array of data disks as part of the VM image. Create your data disks using the New-DataDiskObject cmdlet. Open PowerShell from an elevated prompt, and run:

   ```powershell
    New-DataDiskObject -Lun 2 `
    -Uri "https://storageaccount.blob.core.windows.net/vhds/Datadisk.vhd"
   ```

7. Open PowerShell with an elevated prompt, and run:

   ```powershell
    Add-AzsPlatformimage -publisher "<publisher>" -offer "<offer>" -sku "<sku>" -version "<#.#.#>" -OSType "<ostype>" -OSUri "<osuri>"
   ```

    For more info about the Add-AzsPlatformimage cmdlet and New-DataDiskObject cmdlet, see the Microsoft PowerShell [Azure Stack Operator module documentation](https://docs.microsoft.com/powershell/module/).

## Remove a VM image by using PowerShell

When you no longer need the VM image that you uploaded, you can delete it from the Marketplace by using the following cmdlet:

1. [Install PowerShell for Azure Stack](azure-stack-powershell-install.md).

2. Sign in to Azure Stack as an operator.

3. Open PowerShell with an elevated prompt, and run:

   ```powershell  
   Remove-AzsPlatformImage `
    -publisher "<publisher>" `
    -offer "<offer>" `
    -sku "<sku>" `
    -version "<version>" `
   ```
   The **Remove-AzsPlatformImage** cmdlet specifies values used by the Azure Resource Manager templates to reference the VM image. The values include:
   - **publisher**  
     For example: `Canonical`  
     The publisher name segment of the VM image that users use when they deploy the image. An example is **Microsoft**. Don't include a space or other special characters in this field.  
   - **offer**  
     For example: `UbuntuServer`  
     The offer name segment of the VM image that users use when they deploy the VM image. An example is **WindowsServer**. Don't include a space or other special characters in this field.  
   - **sku**  
     For example: `14.04.3-LTS`  
     The SKU name segment of the VM Image that users use when they deploy the VM image. An example is **Datacenter2016**. Don't include a space or other special characters in this field.  
   - **version**  
     For example: `1.0.0`  
     The version of the VM Image that users use when they deploy the VM image. This version is in the format *\#.\#.\#*. An example is **1.0.0**. Don't include a space or other special characters in this field.  
    
     For more info about the Remove-AzsPlatformImage cmdlet, see the Microsoft PowerShell [Azure Stack Operator module documentation](https://docs.microsoft.com/powershell/module/).

## Next steps

[Provision a virtual machine](../user/azure-stack-create-vm-template.md)
