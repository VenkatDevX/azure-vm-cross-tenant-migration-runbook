# Azure VM Cross-Tenant Migration Runbook

> **Repo:** `azure-devops-vm-migration-runbook`
> **Description:** Step-by-step PowerShell runbook for migrating Azure VMs across tenants using snapshot-based disk transfer — covering snapshot creation, SAS URL export, VHD copy, managed disk creation, and VM provisioning.

---

## Overview

This runbook documents the end-to-end process for migrating a Windows VM from one Azure tenant to another using a snapshot-based approach. It is intended for DevOps and Cloud engineers working in multi-tenant Azure environments.

| Item | Value |
|---|---|
| **Source Location** | East US |
| **Target Location** | Southeast Asia |
| **VM Size** | Standard_D4s_v3 |
| **OS** | Windows Server |
| **Migration Method** | Snapshot → SAS URL → VHD Copy → Managed Disk → VM |

---

## Prerequisites

- Azure PowerShell module (`Az`) installed
- Contributor or Owner access on both source and target subscriptions
- Storage account available (or will be created) in target tenant
- Target VNet/Subnet planned or to be created during migration

---

## Phase 1 — Set Variables

```powershell
# ── SOURCE Variables ──
$SRC_RG        = "rg-source-infra-eus"
$SRC_VM_NAME   = "infra-buildagent-vm01"
$SRC_SUB       = "<source-subscription-id>"
$SRC_TENANT    = "<source-tenant-id>"
$SRC_LOCATION  = "eastus"
$SNAPSHOT_NAME = "snap-buildagent-os-$(Get-Date -Format 'yyyyMMdd')"
$SAS_EXPIRY_HOURS = 3600   # seconds

# ── TARGET Variables ──
$TGT_RG        = "rg-target-infra-sea"
$TGT_LOCATION  = "southeastasia"
$NEW_VM_NAME   = "infra-buildagent-sea-vm01"
$TGT_SUB       = "<target-subscription-id>"
$TGT_TENANT    = "<target-tenant-id>"
$TGT_DISK_NAME = "infra-buildagent-sea-vm01-osdisk"
$TGT_NIC_NAME  = "infra-buildagent-sea-vm01-nic"
$TGT_VNET_NAME = "infra-buildagent-sea-vnet"
$TGT_SUBNET    = "default"
$TGT_IP_NAME   = "infra-buildagent-sea-vm01-pip"
$TGT_NSG_NAME  = "infra-buildagent-sea-nsg"
$STORAGE_ACCOUNT = "migrstginfrasea01"
$CONTAINER_NAME  = "migration"
$VHD_BLOB_NAME   = "buildagent-os.vhd"
$VM_SIZE          = "Standard_D4s_v3"
```

---

## Phase 2 — Create Snapshot (Source Tenant)

```powershell
# ── Login to SOURCE tenant ──
Connect-AzAccount -TenantId $SRC_TENANT
Set-AzContext -SubscriptionId $SRC_SUB -TenantId $SRC_TENANT

# ── Get Source VM OS Disk ──
$srcVM      = Get-AzVM -ResourceGroupName $SRC_RG -Name $SRC_VM_NAME
$osDiskName = $srcVM.StorageProfile.OsDisk.Name
Write-Output "OS Disk: $osDiskName"

# ── Create Snapshot ──
$diskConfig = New-AzSnapshotConfig `
  -SourceUri (Get-AzDisk -ResourceGroupName $SRC_RG -DiskName $osDiskName).Id `
  -Location $SRC_LOCATION `
  -CreateOption Copy

New-AzSnapshot `
  -ResourceGroupName $SRC_RG `
  -SnapshotName $SNAPSHOT_NAME `
  -Snapshot $diskConfig

Write-Output "✅ Snapshot created: $SNAPSHOT_NAME"
```

---

## Phase 3 — Export Snapshot via SAS URL (Source Tenant)

```powershell
$sas = Grant-AzSnapshotAccess `
  -ResourceGroupName $SRC_RG `
  -SnapshotName $SNAPSHOT_NAME `
  -Access Read `
  -DurationInSecond $SAS_EXPIRY_HOURS

$SAS_URL = $sas.AccessSAS
Write-Output "✅ SAS URL generated"
Write-Output $SAS_URL
```

> ⚠️ **Note:** The SAS URL expires after `$SAS_EXPIRY_HOURS` seconds. Copy the VHD before it expires. The URL is sensitive — do not log it to any shared system.

---

## Phase 4 — Copy VHD to Target Storage (Target Tenant)

```powershell
# ── Login to TARGET tenant ──
Connect-AzAccount -TenantId $TGT_TENANT
Set-AzContext -SubscriptionId $TGT_SUB -TenantId $TGT_TENANT

# ── Create Storage Account ──
New-AzStorageAccount `
  -ResourceGroupName $TGT_RG `
  -Name $STORAGE_ACCOUNT `
  -Location $TGT_LOCATION `
  -SkuName Standard_LRS `
  -Kind StorageV2

$storageKey = (Get-AzStorageAccountKey `
  -ResourceGroupName $TGT_RG `
  -AccountName $STORAGE_ACCOUNT)[0].Value

$storageCtx = New-AzStorageContext `
  -StorageAccountName $STORAGE_ACCOUNT `
  -StorageAccountKey $storageKey

# ── Create Container ──
New-AzStorageContainer `
  -Name $CONTAINER_NAME `
  -Context $storageCtx `
  -Permission Off

# ── Copy VHD from SAS URL ──
Start-AzStorageBlobCopy `
  -AbsoluteUri $SAS_URL `
  -DestContainer $CONTAINER_NAME `
  -DestBlob $VHD_BLOB_NAME `
  -DestContext $storageCtx

Write-Output "⏳ VHD copy started..."

# ── Monitor Copy Progress ──
do {
  $status = Get-AzStorageBlobCopyState `
    -Container $CONTAINER_NAME `
    -Blob $VHD_BLOB_NAME `
    -Context $storageCtx
  Write-Output "Status: $($status.Status) | Copied: $($status.BytesCopied) / $($status.TotalBytes)"
  Start-Sleep -Seconds 30
} while ($status.Status -eq "Pending")

Write-Output "✅ VHD Copy Complete!"
```

---

## Phase 5 — Create Managed Disk from VHD (Target Tenant)

```powershell
$vhdUri = "https://$STORAGE_ACCOUNT.blob.core.windows.net/$CONTAINER_NAME/$VHD_BLOB_NAME"

$diskConfig = New-AzDiskConfig `
  -Location $TGT_LOCATION `
  -CreateOption Import `
  -SourceUri $vhdUri `
  -StorageAccountId (Get-AzStorageAccount -ResourceGroupName $TGT_RG -Name $STORAGE_ACCOUNT).Id `
  -OsType Windows `
  -DiskSizeGB 128

New-AzDisk `
  -ResourceGroupName $TGT_RG `
  -DiskName $TGT_DISK_NAME `
  -Disk $diskConfig

Write-Output "✅ Managed Disk created: $TGT_DISK_NAME"
```

---

## Phase 6 — Create Network Resources (Target Tenant)

```powershell
# ── NSG with RDP Rule ──
$nsgRule = New-AzNetworkSecurityRuleConfig `
  -Name "Allow-RDP" `
  -Protocol Tcp `
  -Direction Inbound `
  -Priority 1000 `
  -SourceAddressPrefix "*" `
  -SourcePortRange "*" `
  -DestinationAddressPrefix "*" `
  -DestinationPortRange 3389 `
  -Access Allow

$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName $TGT_RG `
  -Location $TGT_LOCATION `
  -Name $TGT_NSG_NAME `
  -SecurityRules $nsgRule

# ── VNet + Subnet ──
$subnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name $TGT_SUBNET `
  -AddressPrefix "10.0.0.0/24" `
  -NetworkSecurityGroup $nsg

$vnet = New-AzVirtualNetwork `
  -ResourceGroupName $TGT_RG `
  -Location $TGT_LOCATION `
  -Name $TGT_VNET_NAME `
  -AddressPrefix "10.0.0.0/16" `
  -Subnet $subnetConfig

# ── Public IP ──
$publicIP = New-AzPublicIpAddress `
  -ResourceGroupName $TGT_RG `
  -Location $TGT_LOCATION `
  -Name $TGT_IP_NAME `
  -AllocationMethod Static `
  -Sku Standard

# ── NIC ──
$nic = New-AzNetworkInterface `
  -ResourceGroupName $TGT_RG `
  -Location $TGT_LOCATION `
  -Name $TGT_NIC_NAME `
  -SubnetId $vnet.Subnets[0].Id `
  -PublicIpAddressId $publicIP.Id `
  -NetworkSecurityGroupId $nsg.Id

Write-Output "✅ Network resources created"
```

> ⚠️ **Security Note:** The RDP rule (`3389`) is open to `*` for initial access. Restrict to specific IPs or a Bastion host after migration is validated.

---

## Phase 7 — Create VM from Managed Disk (Target Tenant)

```powershell
$osDisk = Get-AzDisk -ResourceGroupName $TGT_RG -DiskName $TGT_DISK_NAME

$vmConfig = New-AzVMConfig -VMName $NEW_VM_NAME -VMSize $VM_SIZE

$vmConfig = Set-AzVMOSDisk `
  -VM $vmConfig `
  -ManagedDiskId $osDisk.Id `
  -CreateOption Attach `
  -Windows

$vmConfig = Add-AzVMNetworkInterface `
  -VM $vmConfig `
  -Id $nic.Id

New-AzVM `
  -ResourceGroupName $TGT_RG `
  -Location $TGT_LOCATION `
  -VM $vmConfig `
  -DisableBginfoExtension

Write-Output "✅ VM Created: $NEW_VM_NAME"
```

---

## Phase 8 — Rename Windows Hostname

```powershell
Invoke-AzVMRunCommand `
  -ResourceGroupName $TGT_RG `
  -VMName $NEW_VM_NAME `
  -CommandId "RunPowerShellScript" `
  -ScriptString "Rename-Computer -NewName 'infra-bldagt-sea' -Force -Restart"

Write-Output "⏳ VM renaming and restarting..."
```

---

## Phase 9 — Verify & Connect

```powershell
# ── Wait for VM to come back ──
Start-Sleep -Seconds 120

# ── Check VM Status ──
Get-AzVM -ResourceGroupName $TGT_RG -Name $NEW_VM_NAME -Status |
  Select-Object -ExpandProperty Statuses | Format-Table

# ── Verify Hostname ──
Invoke-AzVMRunCommand `
  -ResourceGroupName $TGT_RG `
  -VMName $NEW_VM_NAME `
  -CommandId "RunPowerShellScript" `
  -ScriptString "hostname"

# ── Get Public IP for RDP ──
Get-AzPublicIpAddress -ResourceGroupName $TGT_RG |
  Select-Object Name, IpAddress | Format-Table -AutoSize
```

---

## Migration Flow Summary

| Phase | Action | Tenant |
|---|---|---|
| 1 | Set all variables | — |
| 2 | Create OS disk snapshot from source VM | Source |
| 3 | Export SAS URL from snapshot | Source |
| 4 | Create storage account, copy VHD via SAS URL | Target |
| 5 | Create Managed Disk from VHD | Target |
| 6 | Create NSG, VNet, Subnet, Public IP, NIC | Target |
| 7 | Create VM from Managed Disk | Target |
| 8 | Rename Windows hostname and restart | Target |
| 9 | Verify VM status and connect via RDP | Target |

---

## Post-Migration Checklist

- [ ] RDP access verified with correct credentials
- [ ] Hostname matches expected value
- [ ] Azure DevOps agent service running (`services.msc`)
- [ ] Agent registered and online in Azure DevOps agent pool
- [ ] RDP NSG rule restricted to trusted IPs or Bastion
- [ ] Source snapshot deleted after validation
- [ ] Migration storage account and VHD blob cleaned up
- [ ] Tags applied to new VM resources

---

## Cleanup (After Validation)

```powershell
# ── Revoke SAS and delete snapshot (source tenant) ──
Revoke-AzSnapshotAccess -ResourceGroupName $SRC_RG -SnapshotName $SNAPSHOT_NAME
Remove-AzSnapshot -ResourceGroupName $SRC_RG -SnapshotName $SNAPSHOT_NAME -Force

# ── Delete migration storage and VHD (target tenant) ──
Remove-AzStorageAccount -ResourceGroupName $TGT_RG -Name $STORAGE_ACCOUNT -Force
```
