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

## ⚠️ Important Notes Before You Start

> **Read these carefully — ignoring them will cause the migration to fail.**

| # | Issue | Detail |
|---|---|---|
| 1 | **Run all phases in one PowerShell session** | `$SAS_URL` is captured in Phase 3 (source tenant). If you close the terminal before Phase 4, the variable is lost. Save it to a file as a backup (see Phase 3). |
| 2 | **SAS URL is sensitive** | Never print or log it to shared systems, CI pipelines, or chat tools. Treat it like a password. |
| 3 | **Storage account naming rules** | Must be **3–24 characters, lowercase letters and numbers only — no hyphens or underscores**. Example: `migrstginfrasea01` ✅ — `migr-stg-infra` ❌ |
| 4 | **Windows hostname limit** | Max **15 characters**. Plan your hostname in Phase 1 and verify length before Phase 8. |
| 5 | **Target Resource Group must exist** | Phase 4 will fail if `$TGT_RG` does not exist. It is created in Phase 4 before any other resources. |
| 6 | **SAS expiry vs VHD copy time** | A 128 GB disk can take 60–90 minutes to copy. The SAS token is set to **4 hours** (`14400` seconds). Do not reduce this value. |

---

## Prerequisites

- Azure PowerShell module (`Az`) installed — `Install-Module -Name Az -Scope CurrentUser`
- Contributor or Owner access on both source and target subscriptions
- Both tenant IDs and subscription IDs handy before starting
- Target Resource Group does **not** need to exist — it is created in Phase 4

---

## Phase 1 — Set Variables

> Fill in all `<placeholder>` values before running anything. Run this block first in a fresh PowerShell session and keep the session open throughout all phases.

```powershell
# ── SOURCE Variables ──
$SRC_RG           = "rg-source-infra-eus"
$SRC_VM_NAME      = "infra-buildagent-vm01"
$SRC_SUB          = "<source-subscription-id>"
$SRC_TENANT       = "<source-tenant-id>"
$SRC_LOCATION     = "eastus"
$SNAPSHOT_NAME    = "snap-buildagent-os-$(Get-Date -Format 'yyyyMMdd')"
$SAS_EXPIRY_SECONDS = 14400   # 4 hours — do not reduce; large disks take time to copy

# ── TARGET Variables ──
$TGT_RG           = "rg-target-infra-sea"
$TGT_LOCATION     = "southeastasia"
$NEW_VM_NAME      = "infra-bldagt-sea-vm01"
$TGT_SUB          = "<target-subscription-id>"
$TGT_TENANT       = "<target-tenant-id>"
$TGT_DISK_NAME    = "infra-bldagt-sea-osdisk"
$TGT_NIC_NAME     = "infra-bldagt-sea-nic"
$TGT_VNET_NAME    = "infra-bldagt-sea-vnet"
$TGT_SUBNET       = "default"
$TGT_IP_NAME      = "infra-bldagt-sea-pip"
$TGT_NSG_NAME     = "infra-bldagt-sea-nsg"
$STORAGE_ACCOUNT  = "migrstginfrasea01"   # ⚠️ 3-24 chars, lowercase alphanumeric only, no hyphens
$CONTAINER_NAME   = "migration"
$VHD_BLOB_NAME    = "buildagent-os.vhd"
$VM_SIZE          = "Standard_D4s_v3"
$WIN_HOSTNAME     = "bldagt-sea-01"       # ⚠️ Max 15 characters — verify: $WIN_HOSTNAME.Length
```

**Verify hostname length before proceeding:**

```powershell
Write-Output "Hostname length: $($WIN_HOSTNAME.Length)"
# Must be 15 or less. If over 15, shorten $WIN_HOSTNAME above and re-run Phase 1.
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

> ⚠️ Do **not** close your PowerShell session after this phase. `$SAS_URL` must remain in memory for Phase 4. As a backup, save it to a local temp file immediately.

```powershell
$sas = Grant-AzSnapshotAccess `
  -ResourceGroupName $SRC_RG `
  -SnapshotName $SNAPSHOT_NAME `
  -Access Read `
  -DurationInSecond $SAS_EXPIRY_SECONDS

$SAS_URL = $sas.AccessSAS

if ($SAS_URL) {
    Write-Output "✅ SAS URL generated successfully"

    # ── Save to temp file as backup (in case session is interrupted) ──
    $SAS_URL | Out-File -FilePath "$env:TEMP\migration-sas-url.txt" -Force
    Write-Output "💾 SAS URL saved to: $env:TEMP\migration-sas-url.txt"
    Write-Output "⚠️  Delete this file after migration is complete"
} else {
    Write-Error "❌ SAS URL generation failed. Check snapshot name and permissions."
}
```

**To restore `$SAS_URL` if session was interrupted:**

```powershell
$SAS_URL = Get-Content "$env:TEMP\migration-sas-url.txt"
```

---

## Phase 4 — Copy VHD to Target Storage (Target Tenant)

> ⚠️ Login will switch to the target tenant. Ensure `$SAS_URL` is still set in this session before running.

```powershell
# ── Verify SAS URL is still in memory ──
if (-not $SAS_URL) {
    Write-Error "❌ SAS URL is missing. Restore it: `$SAS_URL = Get-Content `"$env:TEMP\migration-sas-url.txt`""
    return
}

# ── Login to TARGET tenant ──
Connect-AzAccount -TenantId $TGT_TENANT
Set-AzContext -SubscriptionId $TGT_SUB -TenantId $TGT_TENANT

# ── Create Target Resource Group (if it doesn't exist) ──
$rgExists = Get-AzResourceGroup -Name $TGT_RG -ErrorAction SilentlyContinue
if (-not $rgExists) {
    New-AzResourceGroup -Name $TGT_RG -Location $TGT_LOCATION
    Write-Output "✅ Resource Group created: $TGT_RG"
} else {
    Write-Output "ℹ️  Resource Group already exists: $TGT_RG"
}

# ── Create Storage Account ──
# ⚠️ Name must be 3-24 chars, lowercase alphanumeric only (no hyphens/underscores)
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

Write-Output "⏳ VHD copy started — monitoring progress every 30 seconds..."

# ── Monitor Copy Progress ──
do {
  $status = Get-AzStorageBlobCopyState `
    -Container $CONTAINER_NAME `
    -Blob $VHD_BLOB_NAME `
    -Context $storageCtx

  $pct = if ($status.TotalBytes -gt 0) {
    [math]::Round(($status.BytesCopied / $status.TotalBytes) * 100, 1)
  } else { 0 }

  Write-Output "[$((Get-Date).ToString('HH:mm:ss'))] Status: $($status.Status) | $pct% | $($status.BytesCopied) / $($status.TotalBytes) bytes"
  Start-Sleep -Seconds 30
} while ($status.Status -eq "Pending")

if ($status.Status -eq "Success") {
    Write-Output "✅ VHD Copy Complete!"
} else {
    Write-Error "❌ VHD Copy failed with status: $($status.Status)"
}
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
# ⚠️ Source is set to * for initial migration access only.
# Restrict to your IP or Azure Bastion after validation.
$nsgRule = New-AzNetworkSecurityRuleConfig `
  -Name "Allow-RDP-Temp" `
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

> ⚠️ `$WIN_HOSTNAME` must be **≤ 15 characters**. Verify with `$WIN_HOSTNAME.Length` from Phase 1 before running this.

```powershell
# ── Verify hostname length before sending to VM ──
if ($WIN_HOSTNAME.Length -gt 15) {
    Write-Error "❌ Hostname '$WIN_HOSTNAME' is $($WIN_HOSTNAME.Length) characters — Windows limit is 15. Update `$WIN_HOSTNAME in Phase 1."
    return
}

Invoke-AzVMRunCommand `
  -ResourceGroupName $TGT_RG `
  -VMName $NEW_VM_NAME `
  -CommandId "RunPowerShellScript" `
  -ScriptString "Rename-Computer -NewName '$WIN_HOSTNAME' -Force -Restart"

Write-Output "⏳ VM renaming to '$WIN_HOSTNAME' and restarting..."
```

---

## Phase 9 — Verify & Connect

```powershell
# ── Wait for VM to come back after restart ──
Write-Output "⏳ Waiting 2 minutes for VM to restart..."
Start-Sleep -Seconds 120

# ── Check VM Power State ──
Get-AzVM -ResourceGroupName $TGT_RG -Name $NEW_VM_NAME -Status |
  Select-Object -ExpandProperty Statuses | Format-Table

# ── Verify Hostname ──
Invoke-AzVMRunCommand `
  -ResourceGroupName $TGT_RG `
  -VMName $NEW_VM_NAME `
  -CommandId "RunPowerShellScript" `
  -ScriptString "hostname"

# ── Get Public IP for RDP ──
$ip = Get-AzPublicIpAddress -ResourceGroupName $TGT_RG -Name $TGT_IP_NAME
Write-Output "🖥️  RDP to: $($ip.IpAddress)"
Write-Output "   Open Remote Desktop and connect to: $($ip.IpAddress):3389"
```

---

## Migration Flow Summary

| Phase | Action | Tenant |
|---|---|---|
| 1 | Set and validate all variables | — |
| 2 | Create OS disk snapshot from source VM | Source |
| 3 | Export SAS URL; save to temp file | Source |
| 4 | Create RG, storage account; copy VHD via SAS URL | Target |
| 5 | Create Managed Disk from VHD | Target |
| 6 | Create NSG, VNet, Subnet, Public IP, NIC | Target |
| 7 | Create VM from Managed Disk | Target |
| 8 | Validate hostname length; rename and restart | Target |
| 9 | Verify VM status and connect via RDP | Target |

---

## Post-Migration Checklist

- [ ] RDP access verified with correct credentials
- [ ] Hostname verified (`hostname` command output matches `$WIN_HOSTNAME`)
- [ ] Azure DevOps agent service running (`services.msc`)
- [ ] Agent registered and online in Azure DevOps agent pool
- [ ] RDP NSG rule (`Allow-RDP-Temp`) restricted to trusted IPs or replaced with Bastion
- [ ] Source snapshot revoked and deleted
- [ ] Migration storage account and VHD blob deleted
- [ ] Temp SAS URL file deleted from `$env:TEMP\migration-sas-url.txt`
- [ ] Tags applied to new VM and network resources

---

## Cleanup (After Validation)

```powershell
# ── Delete temp SAS URL file ──
Remove-Item "$env:TEMP\migration-sas-url.txt" -Force -ErrorAction SilentlyContinue
Write-Output "✅ Temp SAS file deleted"

# ── Switch back to SOURCE tenant ──
Connect-AzAccount -TenantId $SRC_TENANT
Set-AzContext -SubscriptionId $SRC_SUB -TenantId $SRC_TENANT

# ── Revoke SAS access and delete snapshot ──
Revoke-AzSnapshotAccess -ResourceGroupName $SRC_RG -SnapshotName $SNAPSHOT_NAME
Remove-AzSnapshot -ResourceGroupName $SRC_RG -SnapshotName $SNAPSHOT_NAME -Force
Write-Output "✅ Snapshot deleted: $SNAPSHOT_NAME"

# ── Switch to TARGET tenant ──
Connect-AzAccount -TenantId $TGT_TENANT
Set-AzContext -SubscriptionId $TGT_SUB -TenantId $TGT_TENANT

# ── Delete migration storage account (and VHD inside it) ──
Remove-AzStorageAccount -ResourceGroupName $TGT_RG -Name $STORAGE_ACCOUNT -Force
Write-Output "✅ Migration storage account deleted: $STORAGE_ACCOUNT"
```
