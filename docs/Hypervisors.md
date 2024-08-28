# Hypervisors

## Hyper-V 



# Hyper-V server configuration
- FYB standard requires that the latest windows server os is used [ Windows Sever 2022 ]
- [Extra MS DATA For Hyper-V](https://learn.microsoft.com/en-us/biztalk/technical-guides/system-resource-costs-on-hyper-v)
- **Disk requirements**
   - **OS install:** 
     - 2 x 256 GB SSD NVME / SATA configure in RAID 1
   - **VM Storage:** [ minimum one of the following must be used, Drive can be more then listed these are  minimum requirements ] :
     - 4 x 500GB SATA SSDs in RAID 10
     - 8 x 500GB SATA HDDs in RAID 10
     - 2 x 1TB   NVME SSDs  Storage Spaces Mirror / NVME Array Controller
 - **Network Requirements** [ this will depend on the use case]
     - minimum  4 x 1Gb  interfaces teamed 
     - 2 x 10Gb interfaces teamed  
     - 2 x 40Gb interfaces teamed [ this is used if the server will host storage services]
## Server Information
```powershell
invoke-restmethod https://raw.githubusercontent.com/Flipyourbit/winfetch_FYB/main/src/Get-sysdetails.ps1 | invoke-expression
```
## Hardware
### General
>dsregcmd /status check add and domain joined status
#### UEFI/BIOS 
| Option    | Status  |
|------------|---------------|
| | |
|Secure Boot:  | Enabled |
|AMD-v or Intel VT-x :  | Enabled |

#### Install drivers VIA CLI 

```powershell
 PNPUtil.exe /add-driver %drivername%  /install

```

### Dell
#### Vendor Storage
 ##### Enable Fast Path H730 
 > NOTE: when  using SSDs the controller cache settings:
 >>   - No Read Ahead
 >>   - Write-through
 >>  <br> 

#### Install OMS and Chipset driver
> Place Holder 
---
### HP 
---
### Lenovo
### Desktop Hardware

## Virtual Switches
### Design 


<pre>
     \ | NIC Name   \
vNICs <| [Team Type] }  VM Switch Name
     / | NIC Name   /
</pre>

<pre>
     \ | Mellanox connectX3 40Gbps port 1 \
vNICs <|                       [Emb Team ] } uplink-80Gbps
     / | Mellanox connectX3 40Gbps port 2 /
</pre>

<pre>
     \ | Intel x520 10Gbps \
vNICs <|        [Emb Team ] }  uplink-20Gbps
     / | Intel x520 10Gbps /
</pre>


### VM Switch Settings
> Prerequisites Features
>> - Datacenter Bridging 
>> - Hyper-V
>> - SRIOV enable in UEFI and  On the card 

```powershell
# Create VM switch 
 New-vmswitch  -name UPLINK-RDMA -allowmanagementos:0 enableiov:1 -enableembeddedteaming:1 -confirm:0 -netadaptername "nic#", "nic#" 
  ```
 ```powershell
# Add MGMT network adapater
 Add-vmnetworkadapter -name Mgmt-VLAN60 -ManagementOS -switchname uplink-rdma -confirm:0 -passthru | Set-vmnetworkadaptervlan -access -vlanid 60
  ```
 ```powershell
# Add static ip  to mgmt interface 
 New-Netipaddress -ipaddress 10.60.60.x -defaultgateway 10.60.60.1 -prefixlength 24 -interfaceindex (get-netadapter | where-object name -like "*Mgmt-vlan60*").interfaceindex
  ```
 ```powershell
# Set DNS client IP address 
 Set-DnsClientServerAddress -InterfaceIndex (get-netadapter | where-object name -like "*Mgmt-vlan60*").interfaceindex -ServerAddresses ("10.60.61.12","10.60.61.14")
```
### Install FOD 
```powershell 
 Add-WindowsCapability -Online -Name ServerCore.AppCompatibility~~~~0.0.1.0
```

## PCIE and GPU-PV

**GPU Pass through**
```powershell 
# Steps required for GPU passthrough
  Set-VM -Name VMName -AutomaticStopAction TurnOff
  Set-VM -GuestControlledCacheTypes $true -VMName VMName
  Set-VM -LowMemoryMappedIoSpace 3Gb -VMName VMName
  Set-VM -HighMemoryMappedIoSpace 33280Mb -VMName VMName
  Dismount-VMHostAssignableDevice -force -LocationPath  $locationpath   # may need to use force depends on the device 
  Add-VMAssignableDevice -LocationPath $locationPath -VMName VMName


#
set-vm -GuestControlledCacheTypes $true -VMName $_.Name -LowMemoryMappedIoSpace 3Gb -HighMemoryMappedIoSpace 33280Mb }


```

**GPU Para Virtual**
```powershell
# just notes at this point, on how to configure this for dev work
#----

# pulls GPUs that can be partioned
  Get-VMHostPartitionableGpu 
# set VM partitions for a GPU 
  Set-VMHostPartitionableGpu 

```

## Security  :lock:
### Win Firewall
```powershell
# Configure remote desktop firewall access to a certain network
      Get-NetFirewallRule remote*user*mod*  | % { set-netfirewallrule -Name $_.name -RemoteAddress "10.60.66.129-10.60.66.199","10.60.60.0/24","10.60.66.226"   }
```
### Securing Hyper-v Hosts
[ref-link](https://learn.microsoft.com/en-us/windows-server/virtualization/hyper-v/plan/plan-hyper-v-security-in-windows-server)

| Option    | Status  | Ref |
|------------|---------------|-------|
| | |
|Credential Guard: | Enabled | [Link](https://learn.microsoft.com/en-us/windows/security/identity-protection/credential-guard/credential-guard) |
|code integrity policies:  | Enabled |[Link](https://learn.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/windows-defender-application-control-deployment-guide)  |

#### Apply Security Standards


---
## Storage
### Storage Naming convention 
 All storage prefix will begin with the Type IE "ZFS", HRAID, DRAID, NFS, SMB,MSSDS
 >[Disk Config Type][Storage Prefix][Storage Type][Media][Raid level][unique ID]
- NVME and Optane storage  [Platinum]
  
 ### NAND based storage 
 | Media         | Interface Type/protocol/Storage Type| Prefix | Abv Prefix | Raid type or equivalent |
 |     ----      |  ------  | ------ |  ---  | ---- | 
 | NAND / Optane |  NVME / PCIe / U.2 ..etc   |  [Platinum]  | [Plat] | all except R0 | 
 |  Optane       | P Memory       |  [Platinum]     | [Plat] | all except R0 | 
 | NAND / Optane | SAS / SATA     | [Gold] | [GLD]  | all except R0 | 
 

 ### Spinning Rust
 | Type          | Interface Type/protocol/Storage Type| Prefix | Abv Prefix | Raid type or equivalent |
 |     ----      |  ------  | ------ |  ---  | ---- | 
 | HDD 15k to 7200k | SAS-12G, SATA-6G     | [Silver] | [SLVR] | R10, Mirror |
 | HDD 15k to 7200k | SAS-12G, SATA-6G     | [Bronze] | [BRNZ] | R5, R6, Raidz1 and 2  |
 | HDD 15k to 7200k | SAS-6G, SATA-3G      | [Silver] | [SLVR] | R10, Mirror |
 | HDD 15k to 7200k | SAS-6G, SATA-3G      | [Silver] | [SLVR] | R10, Mirror, Raidz1 with SSD meta vDev  |
 | HDD 15k to 7200k | SAS-6G, SATA-3G      | [Bronze] | [BRNZ] | R5, R6, Raidz1 and 2  |
 | HDD 5400k | Any SAS/SAS/USB/eSATA      | [Copper] | [Cppr]   |  Any   |

### Use Case 
| Storage Class | Use Case |
|---|----|
|Platinum | latency sensitive workload and Heavy  Use Virtual machine OS Storage | 
|GOLD     |  Heavy  Use Virtual machine OS Storage  and Mass storage |
|Silver   |  Medium   Use Virtual machine OS Storage and Mass Storage |
|Bronze   | Backups and light VM use for certain use cases |  
|Copper   | Archival Storage Backs ups or cold storage, Low use files | 

>NOTE
>> Hybrid approaches may fall into different classes.


---
### Storage Configuration [ SAN Storage ]
```powershell

  # intended to be blank

```

---
### Storage Configuration  [ SDS ]

```powershell
# New storage pool SSD
 New-storagepool -FriendlyName 'SSD-Storage-pool' -StorageSubSystemUniqueId (Get-StorageSubSystem).UniqueId -PhysicalDisks (Get-PhysicalDisk | where friendlyname -like "ata*")
 ```

 ```powershell
# New storage pool  NVME
 New-storagepool -FriendlyName 'NVME-Storage-pool' -StorageSubSystemUniqueId (Get-StorageSubSystem).UniqueId -PhysicalDisks (Get-PhysicalDisk | where friendlyname -like "wdbr*")
  ```
 ```powershell
# New Virtual Disk
 New-VirtualDisk -StoragePoolUniqueId $poolvar.UniqueId -FriendlyName 'SSD-VD-Mir' -ResiliencySettingName mirror -UseMaximumSize -Interleave 32KB
  ```
 ```powershell
# New Virtual Disk  and Volume
 Get-Disk | where-object friendlyname -like "ssd-vd-mir-001" | Initialize-Disk -PartitionStyle GPT -PassThru | New-Partition -UseMaximumSize -AssignDriveLetter| Format-Volume -FileSystem NTFS -AllocationUnitSize 64KB -NewFileSystemLabel 'SSD_Hyper-V_Strg-01'
```

## Add Drivers from  CLI 


pnputil /add-driver device.inf /install


----
# EOF