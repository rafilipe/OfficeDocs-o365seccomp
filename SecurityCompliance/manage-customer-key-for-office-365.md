---
title: "Controlling your data in Office 365 using Customer Key"
ms.author: krowley
author: kccross
manager: laurawi
ms.date: 8/1/2018
audience: ITPro
ms.topic: article
ms.service: O365-seccomp
localization_priority: Normal
search.appverid:
- MET150
ms.assetid: f2cd475a-e592-46cf-80a3-1bfb0fa17697
ms.collection:
- M365-security-compliance
description: "Learn how to set up Customer Key for Office 365 for Exchange Online, Skype for Business, SharePoint Online, and OneDrive for Business. With Customer Key, you control your organization's encryption keys and then configure Office 365 to use them to encrypt your data at rest in Microsoft's datacenters."
---

# Manage Customer Key for Office 365
<a name="ManageCustomerKey"> </a>

After you've set up Customer Key for Office 365, you can perform these additional management tasks.
  
- [Restore Azure Key Vault keys](manage-customer-key-for-office-365.md#RestoreAzureKeyVaultKeys)
    
- [Rolling or rotating a key in Azure Key Vault that you use with Customer Key](manage-customer-key-for-office-365.md#RollCKkey)
    
- [Manage key vault permissions](manage-customer-key-for-office-365.md#Managekeyvaultperms)
    
- [Determine the DEP assigned to a mailbox](manage-customer-key-for-office-365.md#DeterminemailboxDEP)
    
### Restore Azure Key Vault keys
<a name="RestoreAzureKeyVaultKeys"> </a>

Before performing a restore, use the recovery capabilities provided by soft delete. All keys that are used with Customer Key are required to have soft delete enabled. Soft delete acts like a recycle bin and allows recovery for up to 90 days without the need to restore. Restore should only be required in extreme or unusual circumstances, for example if the key or key vault is lost. If you must restore a key for use with Customer Key, in Azure PowerShell, run the Restore-AzureKeyVaultKey cmdlet as follows:
  
```
Restore-AzureKeyVaultKey -VaultName <vaultname> -InputFile <filename>
```

For example:
  
```
Restore-AzureKeyVaultKey -VaultName Contoso-O365EX-NA-VaultA1 -InputFile Contoso-O365EX-NA-VaultA1-Key001-Backup-20170802.backup
```

If a key with the same name already exists in the key vault, the restore operation will fail. Restore-AzureKeyVaultKey restores all key versions and all metadata for the key including the key name.
  
### Rolling or rotating a key in Azure Key Vault that you use with Customer Key
<a name="RollCKkey"> </a>

Rolling keys is not required by either Azure Key Vault or by Customer Key. In addition, keys that are protected with an HSM are virtually impossible to compromise. Even if a root key were in the possession of a malicious actor there is no feasible means of using it to decrypt data, since only Office 365 code knows how to use it. However, rolling a key is supported by Customer Key.
  
> [!CAUTION]
> Only roll an encryption key that you use with Customer Key when a clear technical reason exists or a compliance requirement dictates that you have to roll the key. In addition, do not delete any keys that are or were associated with policies. When you roll your keys, there will be content encrypted with the previous keys. For example, while active mailboxes will be re-encrypted frequently, inactive, disconnected, and disabled mailboxes may still be encrypted with the previous keys. SharePoint Online performs backup of content for restore and recovery purposes, so there may still be archived content using older keys. <br/> To ensure the safety of your data, SharePoint Online will allow no more than one Key Roll operation to be in progress at a time. If you want to roll both of the keys in a key vault, you'll need to wait for the first key roll operation to fully complete. Our recommendation is to stagger your key roll operations at different intervals, so that this is not an issue. 
  
When you roll a key, you are requesting a new version of an existing key. In order to request a new version of an existing key, you use the same cmdlet, [Add-AzureKeyVaultKey](https://docs.microsoft.com/powershell/module/AzureRM.KeyVault/Add-AzureKeyVaultKey), with the same syntax that you used to create the key in the first place.
  
For example:
  
```
Add-AzureKeyVaultKey -VaultName Contoso-O365EX-NA-VaultA1 -Name Contoso-O365EX-NA-VaultA1-Key001 -Destination HSM -KeyOps @('wrapKey','unwrapKey') -NotBefore (Get-Date -Date "12/27/2016 12:01 AM")
```

In this example, since a key named **Contoso-O365EX-NA-VaultA1-Key001** already exists in the **Contoso-O365EX-NA-VaultA1** vault, a new key version will be created. The operation adds a new key version. This operation preserves the previous key versions in the key's version history, so that data previously encrypted with that key can still be decrypted. Once you have completed rolling any key that is associated with a DEP, you must then run an additional cmdlet to ensure Customer Key begins using the new key. 
  
#### Enable Exchange Online and Skype for Business to use a new key after you roll or rotate keys in Azure Key Vault

When you roll either of the Azure Key Vault keys associated with a DEP used with Exchange Online and Skype for Business, you must run the following command to update the DEP and enable Office 365 to start using the new key.
  
To instruct Customer Key to use the new key to encrypt mailboxes in Office 365 run the Set-DataEncryptionPolicy cmdlet as follows:
  
```
Set-DataEncryptionPolicy <policyname> -Refresh 
```

Within 48 hours, the active mailboxes encrypted using this policy will become associated with the updated key. Use the steps in [Determine the DEP assigned to a mailbox](controlling-your-data-using-customer-key.md#DeterminemailboxDEP) to check the value for the DataEncryptionPolicyID property for the mailbox. The value for this property will change once the updated key has been applied. 
  
#### Enable SharePoint Online and OneDrive for Business to use a new key after you roll or rotate keys in Azure Key Vault

When you roll either of the Azure Key Vault keys associated with a DEP used with SharePoint Online and OneDrive for Business, you must run the [Update-SPODataEncryptionPolicy](https://technet.microsoft.com/library/mt843948.aspx) cmdlet to update the DEP and enable Office 365 to start using the new key. 
  
```
Update-SPODataEncryptionPolicy -Identity <SPOAdminSiteUrl> -KeyVaultName <ReplacementKeyVaultName> -KeyName <ReplacementKeyName> -KeyVersion <ReplacementKeyVersion> -KeyType <Primary | Secondary>
```

This will start the key roll operation for SharePoint Online and OneDrive for Business. This action is not immediate. To see the progress of the key roll operation, run the Get-SPODataEncryptionPolicy cmdlet as follows:
  
```
Get-SPODataEncryptionPolicy -Identity <SPOAdminSiteUrl>
```

### Manage key vault permissions
<a name="Managekeyvaultperms"> </a>

Several cmdlets are available that enable you to view and, if necessary, remove key vault permissions. You might need to remove permissions, for example, when an employee leaves the team.
  
To view key vault permissions, run the Get-AzureRmKeyVault cmdlet:
  
```
Get-AzureRmKeyVault -VaultName <vaultname>
```

For example:
  
```
Get-AzureRmKeyVault -VaultName Contoso-O365EX-NA-VaultA1
```

To remove an administrator's permissions, run the Remove-AzureRmKeyVaultAccessPolicy cmdlet:
  
```
Remove-AzureRmKeyVaultAccessPolicy -VaultName <vaultname> 
-UserPrincipalName <UPN of user>
```

For example:
  
```
Remove-AzureRmKeyVaultAccessPolicy -VaultName Contoso-O365EX-NA-VaultA1 
-UserPrincipalName alice@contoso.com
```

### Determine the DEP assigned to a mailbox
<a name="DeterminemailboxDEP"> </a>

To determine the DEP assigned to a mailbox, use the Get-MailboxStatistics cmdlet. The cmdlet returns a unique identifier (GUID).
  
```
Get-MailboxStatistics -Identity <GeneralMailboxOrMailUserIdParameter> | fl DataEncryptionPolicyID
```

Where *GeneralMailboxOrMailUserIdParameter* specifies a mailbox. For more information about the Get-MailboxStatistics cmdlet, see [Get-MailboxStatistics](https://technet.microsoft.com/library/bb124612%28v=exchg.160%29.aspx).
  
Use the GUID to find out the friendly name of the DEP to which the mailbox is assigned by running the following cmdlet.
  
```
Get-DataEncryptionPolicy <GUID>
```

Where *GUID* is the GUID returned by the Get-MailboxStatistics cmdlet in the previous step. 
  

# Related FAQs

## Can I assign a data encryption policy before migrating a mailbox to the cloud?
<a name="DiffCustomerKeyandBYOKAzureIP"> </a>

Yes. You can use the Windows PowerShell cmdlet Set-MailUser to assign a data encryption policy (DEP) to the user prior to migrating the mailbox to Office 365. When you do this, the contents of the mailbox will be encrypted using the assigned DEP as the content is migrated. This can be more efficient than assigning a DEP after the mailbox has already been migrated and then waiting for encryption to take place, which can take hours or possibly days. 

## How do I verify that encryption with Customer Key is activated and Office 365 has finished encrypting with Customer Key?
<a name="DiffCustomerKeyandBYOKAzureIP"> </a>

 **Exchange Online and Skype for Business:** You can [connect to Exchange Online using remote PowerShell](https://technet.microsoft.com/en-us/library/jj984289%28v=exchg.160%29.aspx) and then use the **[Get-MailboxStatistics]** cmdlet for each mailbox that you want to check. In the output from the Get-MailboxStatistics cmdlet, the  _IsEncrypted_ property returns a value of **true** if the mailbox is encrypted and a value of **false** if it's not. If the mailbox is encrypted, the value returned for the  _DataEncryptionPolicyID_ property is the GUID of the DEP with which the mailbox is encrypted. For more information on running this cmdlet, see [Get-MailboxStatistics](https://technet.microsoft.com/en-us/library/bb124612%28v=exchg.160%29.aspx) and using PowerShell with Exchange Online. 
  
If the mailboxes aren't encrypted after waiting 72 hours from the time you assigned the DEP, initiate a mailbox move. To do this, [connect to Exchange Online using remote PowerShell](https://technet.microsoft.com/en-us/library/jj984289%28v=exchg.160%29.aspx) and then use the New-MoveRequest cmdlet and provide the alias of the mailbox as follows: 
  
```
New-MoveRequest <alias>
```

 **SharePoint Online and OneDrive for Business:** You can [connect to SharePoint Online PowerShell](https://technet.microsoft.com/en-us/library/fp161372.aspx), and then use the **[Get-SPODataEncryptionPolicy]** cmdlet to check the status of your tenant. The ** _State_** property returns a value of **registered** if Customer Key encryption is enabled and all files in all sites have been encrypted. If encryption is still in progress, this cmdlet provides information on what percentage of sites is complete. 

 ## If I want to switch to a different set of keys, how long does it take for the new set of keys to protect my data?
<a name="DiffCustomerKeyandBYOKAzureIP"> </a>

 **Exchange Online and Skype for Business:** It can take up to 72 hours to protect a mailbox according to a new Data Encryption Policy (DEP) from the time the new DEP is assigned to the mailbox. 
  
 **SharePoint Online and OneDrive for Business:** It can take up to four hours to re-encrypt your entire tenant once a new key has been assigned. 
  