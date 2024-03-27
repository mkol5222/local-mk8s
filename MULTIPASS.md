
## Multipass: Assign static IP address to VMs

### Windows: add new virtual switch for Multipass VMs (Hyper-V)

```shell
# Windows Administator PowerShell
# add dedicated Hyper-V virtual switch
New-VMSwitch -SwitchName "multipass" -SwitchType Internal
# get the interface index
Get-NetAdapter | Select-Object -Property Name, InterfaceDescription, ifIndex
# look for "vEthernet (multipass)"
Get-NetAdapter | ? Name -eq "vEthernet (multipass)" | % ifIndex
$multipassIfIndex = Get-NetAdapter | ? Name -eq "vEthernet (multipass)" | % ifIndex

# choosing address
# you might have VPN connected - check if VPN contains your subnet

# hopefully free
get-netroute | ? { $_.DestinationPrefix.StartsWith("10.38") }
# might be used on VPN
get-netroute | ? { $_.DestinationPrefix.StartsWith("192.168.0") }

# use free address for our new segment
New-NetIPAddress -IPAddress 10.38.0.254 -PrefixLength 24 -InterfaceIndex $multipassIfIndex

# check if it's there
Get-NetIPAddress -InterfaceAlias "vEthernet (multipass)" | Select-Object IPAddress, PrefixLength
```
### Use static IP address for Multipass VMs

```shell
# we specify new vSwitch network during VM launch
# we also specify MAC address to be able to assign static IP address
multipass launch -n testvm --network name=multipass,mode=manual,mac=52:54:00:f1:94:fe -m 2G -d 10G -c 2

# verify new interface eth1 - focus on MAC address
multipass exec testvm -- ip link show eth1

# assign static IP address
multipass shell testvm

# in VM testvm - make sure MAC is matching one from launch command
sudo su -
cat << EOF > /etc/netplan/10-netcfg.yaml
network:
    version: 2
    ethernets:
        extra0:
            dhcp4: no
            match:
                macaddress: "52:54:00:f1:94:fe"
            addresses: [10.38.0.99/24]
EOF
chmod o= /etc/netplan/10-netcfg.yaml
chmod o= /etc/netplan/50-cloud-init.yaml
exit # root shell
exit # vm

# back in Windows
multipass exec testvm -- sudo netplan apply

# multipass - confirm 2nd interface
multipass ls

# should be reachable from Windows
ping 10.38.0.99 -t

```

### Set global DNS server on Multipass VM

```shell
# in VM testvm
multipass shell testvm

# check current DNS server under Global section
resolvectl

# in VM testvm - set global DNS server
 sudo sed -i 's/.*DNS=.*/DNS=1.1.1.1/' /etc/systemd/resolved.conf
 sudo systemctl restart systemd-resolved
 resolvectl

```

### Multipass data location

You might have limited space on your system drive. You can move Multipass data to another drive.
https://multipass.run/docs/configure-multipass-storage#heading--windows

```shell
# look at current location
Get-ItemPropertyValue -Path "HKLM:System\CurrentControlSet\Control\Session Manager\Environment" -Name MULTIPASS_STORAGE

# look for SSH keys location
$multipassStorage = Get-ItemPropertyValue -Path "HKLM:System\CurrentControlSet\Control\Session Manager\Environment" -Name MULTIPASS_STORAGE

# look for ssh key (as admin)
Get-ChildItem -Path $multipassStorage -Recurse -Filter id_rsa

# full path
Get-ChildItem -Path $multipassStorage -Recurse -Filter id_rsa | ? FullName

# key location
$sshkey = Get-ChildItem -Path $multipassStorage -Recurse -Filter id_rsa | % FullName
```

### Multipass SSH key location

```shell
# get location of SSH key above
$sshkey="D:\ProgramData\Multipass\data\ssh-keys\id_rsa"
# try it - assume testvm is running
ssh -i $sshkey ubuntu@testvm.mshome.net
```

### Multipass - service restart

Requires admin Powershell session.

```shell
# restart Multipass service
Restart-Service multipass

# check if it's running
Get-Service multipass
```