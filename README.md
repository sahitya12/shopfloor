# Ensure the Az module is installed
if (-not (Get-Module -ListAvailable -Name Az)) {
    Install-Module -Name Az -Force -AllowClobber -Scope CurrentUser -Confirm:$false
}
Import-Module Az

# Retrieve required environment variables
$adh_group = "$($env:adh_group)"
$adh_subgroup = "$($env:adh_subgroup)"
$adh_subscription_type = "$($env:adh_subscription_type)"
$RITM = "$($env:RITM)"

$tenantId = $env:tenant_id
$clientId = $env:client_id
$clientSecret = $env:client_secret
$subscription_env_variable = "$($adh_group)_$($adh_subscription_type)_subscription_id".ToLower()
$subscription_id = (Get-Item -Path env:\$subscription_env_variable).Value

# Set Resource Group
if ($adh_group -and $adh_subscription_type -and $adh_subgroup) {
    $ResourceGroupName = "ADH_${adh_group}_${adh_subgroup}_VM_SHA"
} elseif ($adh_group -and $adh_subscription_type) {
    $ResourceGroupName = "ADH_${adh_group}_VM_SHA"
} else {
    Write-Error "Insufficient inputs to construct resource group name."
    exit 1
}

# Login to Azure using service principal
Connect-AzAccount -ServicePrincipal -Tenant $tenantId `
    -Credential (New-Object -TypeName PSCredential -ArgumentList $clientId, (ConvertTo-SecureString -String $clientSecret -AsPlainText -Force))
Set-AzContext -Subscription $subscription_id

# Read the JSON
$vmJsonPath = "$(System.DefaultWorkingDirectory)/vm.json"
if (-not (Test-Path -Path $vmJsonPath)) {
    Write-Error "vm.json not found at $vmJsonPath"
    exit 1
}

try {
    $vmList = Get-Content -Raw -Path $vmJsonPath | ConvertFrom-Json
} catch {
    Write-Error "Failed to parse vm.json: $_"
    exit 1
}

# Loop through each VM entry
foreach ($entry in $vmList) {
    $vmName = $entry.name
    $desName = $entry.diskEncryptionSet

    if (-not $vmName -or -not $desName) {
        Write-Warning "Missing VM name or DES name for entry: $($entry | ConvertTo-Json -Compress)"
        continue
    }

    Write-Host "`n--- Processing VM: $vmName ---"

    # Get DES ID
    try {
        $des = Get-AzDiskEncryptionSet -ResourceGroupName $ResourceGroupName -Name $desName
        $desId = $des.Id
        Write-Host "Resolved DES ID: $desId"
    } catch {
        Write-Error "Could not find DES '$desName' in resource group '$ResourceGroupName': $_"
        continue
    }

    # Deallocate VM
    try {
        Write-Host "Deallocating VM: $vmName"
        Stop-AzVM -ResourceGroupName $ResourceGroupName -Name $vmName -Force -NoWait
        do {
            Start-Sleep -Seconds 10
            $vmState = (Get-AzVM -ResourceGroupName $ResourceGroupName -Name $vmName -Status).Statuses[1].Code
            Write-Host "Current VM State: $vmState"
        } while ($vmState -ne "PowerState/deallocated")
    } catch {
        Write-Error "Failed to deallocate VM: $_"
        continue
    }

    # Get OS disk and update encryption
    try {
        $vm = Get-AzVM -ResourceGroupName $ResourceGroupName -Name $vmName
        $osDiskName = $vm.StorageProfile.OsDisk.Name
        $osDisk = Get-AzDisk -ResourceGroupName $ResourceGroupName -DiskName $osDiskName

        Write-Host "Updating encryption for OS Disk: $osDiskName"
        $osDisk.Encryption = @{
            Type = 'EncryptionAtRestWithCustomerKey'
            DiskEncryptionSetId = $desId
        }
        Update-AzDisk -ResourceGroupName $ResourceGroupName -DiskName $osDiskName -Disk $osDisk
        Write-Host "OS disk encryption updated successfully."
    } catch {
        Write-Error "Failed to update OS disk encryption: $_"
        continue
    }

    # Start the VM
    try {
        Write-Host "Starting VM: $vmName"
        Start-AzVM -ResourceGroupName $ResourceGroupName -Name $vmName
        Write-Host "VM started."
    } catch {
        Write-Error "Failed to start VM: $_"
    }
}



