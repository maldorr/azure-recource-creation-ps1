# AZURE-FULL-STACK-VM-WITH-VOLUME
## PowerShell Script - Replicating OpenStack HEAT Template

This project replicates an OpenStack full-stack deployment (Network, Security, VM, Volume, Floating IP) using native Azure PowerShell.

---

### ---------- PREREQUISITES ----------

```powershell
# Login to Azure
Connect-AzAccount

# Select your subscription (if needed)
# Set-AzContext -SubscriptionId "your-id-here"

```

### ---------- VARIABLES ----------

```powershell
$location = "switzerlandnorth"
$resourceGroupName = "RG-OpenStack-Replica"

# Network Names (Matching HEAT: my_net, my_subnet)
$vnetName       = "my_private_network"
$subnetName     = "my_subnet"
$publicIpName   = "my_floating_ip"
$nsgName        = "allow_ssh_ping"
$nicName        = "my_interface"

# Storage Name (Matching HEAT: my_data_volume)
$diskName       = "my_extra_disk"
$diskSizeGB     = 1

# VM Details (Matching HEAT: My_Full_Server)
$vmName         = "My-Full-Server"
$vmSize         = "Standard_B1s"  # Similar to m1.tiny

# Credentials
$adminUser      = "azureuser"
$adminPassword  = ConvertTo-SecureString "SecurePass!123" -AsPlainText -Force
$cred           = New-Object System.Management.Automation.PSCredential ($adminUser, $adminPassword)

```

### ---------- RESOURCE GROUP ----------

```powershell
New-AzResourceGroup -Name $resourceGroupName -Location $location

```

### ---------- VNET + SUBNET ----------

```powershell
# 1. Create Subnet Config (192.168.100.0/24)
$subnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name $subnetName `
  -AddressPrefix "192.168.100.0/24"

# 2. Create Virtual Network
$vnet = New-AzVirtualNetwork `
  -Name $vnetName `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -AddressPrefix "192.168.0.0/16" `
  -Subnet $subnetConfig

```

### ---------- SECURITY GROUP (NSG) ----------

```powershell
# Create the NSG container
$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -Name $nsgName

```

```powershell
# Rule 1: Allow SSH (Port 22) - Matches HEAT tcp 22
$nsg | Add-AzNetworkSecurityRuleConfig `
  -Name "Allow-SSH" `
  -Protocol Tcp `
  -Direction Inbound `
  -Priority 1001 `
  -SourceAddressPrefix "*" `
  -SourcePortRange "*" `
  -DestinationAddressPrefix "*" `
  -DestinationPortRange 22 `
  -Access Allow | Out-Null

# Rule 2: Allow ICMP (Ping) - Matches HEAT protocol: icmp
$nsg | Add-AzNetworkSecurityRuleConfig `
  -Name "Allow-ICMP" `
  -Protocol Icmp `
  -Direction Inbound `
  -Priority 1002 `
  -SourceAddressPrefix "*" `
  -SourcePortRange "*" `
  -DestinationAddressPrefix "*" `
  -DestinationPortRange "*" `
  -Access Allow | Set-AzNetworkSecurityGroup

```

### ---------- PUBLIC IP (FLOATING IP) ----------

```powershell
$publicIp = New-AzPublicIpAddress `
  -Name $publicIpName `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -AllocationMethod Static `
  -Sku Standard

```

### ---------- NETWORK INTERFACE (NIC) ----------

```powershell
$subnet = Get-AzVirtualNetworkSubnetConfig -Name $subnetName -VirtualNetwork $vnet

$nic = New-AzNetworkInterface `
  -Name $nicName `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -Subnet $subnet `
  -NetworkSecurityGroup $nsg `
  -PublicIpAddress $publicIp

```

### ---------- STORAGE (EXTRA VOLUME) ----------

```powershell
# Matches HEAT resource: my_data_volume (Size: 1GB)
$diskConfig = New-AzDiskConfig `
  -Location $location `
  -CreateOption Empty `
  -DiskSizeGB $diskSizeGB `
  -Sku Standard_LRS

$dataDisk = New-AzDisk `
  -ResourceGroupName $resourceGroupName `
  -DiskName $diskName `
  -Disk $diskConfig

```

### ---------- VM CONFIGURATION & CREATION ----------

```powershell
# 1. Configure VM Base
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize

# 2. Set OS (Ubuntu Linux)
$vmConfig = $vmConfig | Set-AzVMOperatingSystem `
  -Linux `
  -ComputerName $vmName `
  -Credential $cred `
  -DisablePasswordAuthentication:$false

# 3. Set Image (Canonical Ubuntu)
$vmConfig = $vmConfig | Set-AzVMSourceImage `
  -PublisherName "Canonical" `
  -Offer "0001-com-ubuntu-server-jammy" `
  -Skus "22_04-lts-gen2" `
  -Version "latest"

# 4. Add Network Interface
$vmConfig = $vmConfig | Add-AzVMNetworkInterface -Id $nic.Id

# 5. Attach the Extra Volume (Matches HEAT: volume_attachment)
$vmConfig = $vmConfig | Add-AzVMDataDisk `
  -CreateOption Attach `
  -ManagedDiskId $dataDisk.Id `
  -Lun 0

# 6. Create the VM
New-AzVM -ResourceGroupName $resourceGroupName -Location $location -VM $vmConfig

```

### ---------- OUTPUTS ----------

```powershell
$finalIp = (Get-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $resourceGroupName).IpAddress
Write-Host "✅ VM Created Successfully"
Write-Host "SSH Command: ssh $adminUser@$finalIp"
Write-Host "Volume Status: Attached at LUN 0"

```

```

```
