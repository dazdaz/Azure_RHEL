## Deploy and configure RHEL 7.5 Server on Azure
#### Adding data disks onto an Azure Linux VM + Re-sizing a data-disk
#### https://docs.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-manage-disks
```
/dev/sda OS Disk
 /dev/sdb Temporary Disk
 /dev/sdc Data Disk #1
 /dev/sdd Data Disk #2
```

```
$ az group create --name myrhel-rg --location southeastasia
$ az vm create --location southeastasia --resource-group myrhel-rg --name myrhel --public-ip-address-dns-name myrhel \
--image RedHat:RHEL:7.3:latest --admin-username myuser --admin-password 'SS12345678$$' --size Standard_B1ms \
--data-disk-sizes-gb 5 5 --tags environmenttype=dev owner=harry@oceanliner.com
ssh -l myuser myrhel.southeastasia.cloudapp.azure.com
# Update to RHEL 7.5
sudo yum -y update ; sudo reboot
```

* Temporary Disk
```
$ df -h /mnt/resource
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       3.9G  2.1G  1.7G  56% /mnt/resource

$ swapon -s --all
Filename                                Type            Size    Used    Priority
/mnt/resource/swapfile                  file    2097148 0       -1
```

```
$ dmesg | grep sdc
[    5.670486] sd 5:0:0:0: [sdc] 10485760 512-byte logical blocks: (5.36 GB/5.00 GiB)
[    5.694489] sd 5:0:0:0: [sdc] 4096-byte physical blocks
[    5.852078] sd 5:0:0:0: [sdc] Write Protect is off
[    5.868797] sd 5:0:0:0: [sdc] Mode Sense: 0f 00 10 00
[    5.951575] sd 5:0:0:0: [sdc] Write cache: disabled, read cache: enabled, supports DPO and FUA
[    6.065186] sd 5:0:0:0: [sdc] Attached SCSI disk
```

```
$ dmesg | grep sdd
[    5.713922] sd 5:0:0:1: [sdd] 10485760 512-byte logical blocks: (5.36 GB/5.00 GiB)
[    5.737661] sd 5:0:0:1: [sdd] 4096-byte physical blocks
[    5.868823] sd 5:0:0:1: [sdd] Write Protect is off
[    5.886122] sd 5:0:0:1: [sdd] Mode Sense: 0f 00 10 00
[    5.980540] sd 5:0:0:1: [sdd] Write cache: disabled, read cache: enabled, supports DPO and FUA
[    6.080823] sd 5:0:0:1: [sdd] Attached SCSI disk
```

```
$ (echo n; echo p; echo 1; echo ; echo ; echo w) | sudo fdisk /dev/sdc
$ sudo mkfs -t ext4 /dev/sdc1
$ sudo mkdir /datadrive1 && sudo mount /dev/sdc1 /datadrive1
$ df -h
$ mount
$ sudo -i blkid | grep sdc1
/dev/sdc1: UUID="cda63657-f1a5-4739-b50c-8339768e8ec8" TYPE="ext4"
# Add to /etc/fstab
UUID=cda63657-f1a5-4739-b50c-8339768e8ec8   /datadrive1  ext4    defaults,nofail,barrier=0   0  2
```

## Adding a 3rd data-disk dynamically to a Linux VM, after VM deployment
* https://docs.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-manage-disks
* P20 Premium Disk = 512GiB of storage space, 2300 IOPS and 150 MB/sec which costs USD $73.220/month.
* Data Disk names must be unique per subscription, not resource group

```
$ az vm disk attach --vm-name rhel75 --resource-group rhel75-rg --disk myDataDisk3 --size-gb 512 --sku Premium_LRS --new
$ dmesg | grep sde

# Check that the disk has been provisioned successfully
$ az disk list --resource-group ubuntu1710-rg --output table
Name                                                  ResourceGroup    Location       Zones    Sku             SizeGb  ProvisioningState    OsType
----------------------------------------------------  ---------------  -------------  -------  ------------  --------  -------------------  --------
myDataDisk1                                           ubuntu1710-rg    southeastasia           Standard_LRS         5  Succeeded
myDataDisk2                                           ubuntu1710-rg    southeastasia           Standard_LRS         5  Succeeded
myDataDisk3                                           ubuntu1710-rg    southeastasia           Standard_LRS         5  Succeeded
ubuntu1710_OsDisk_1_f9c75e228c684b23b12536b3610e92a6  ubuntu1710-rg    southeastasia           Premium_LRS         30  Succeeded            Linux

$ (echo n; echo p; echo 1; echo ; echo ; echo w) | sudo fdisk /dev/sde
$ sudo mkfs -t ext4 /dev/sde1
$ sudo mkdir /datadrive3 && sudo mount -o barrier=0 /dev/sde1 /datadrive3
$ df -h

$ sudo -i blkid | grep sde1
/dev/sde1: UUID="cda63657-f1a5-4739-b50c-8339768e8ec8" TYPE="ext4"
# Add to /etc/fstab
UUID=cda63657-f1a5-4739-b50c-8339768e8ec8   /datadrive3  ext4    defaults,nofail,barrier=0   0  2
```
* Use barrier=0 mount option with ext4 on Azure Premium Disks
* Use nobarrier mount option with xfs on Azure Premium Disks
https://docs.microsoft.com/en-us/azure/virtual-machines/windows/premium-storage#premium-storage-for-linux-vms

* Open up firewall port in Azure to the VM
```
$ az vm open-port --port 80 --resource-group rhel75-rg --name rhel75
```

* Install azure-cli
```
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[azure-cli]\nname=Azure CLI\nbaseurl=https://packages.microsoft.com/yumrepos/azure-cli\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
sudo yum -y install azure-cli
```

* Install Docker CE 18.03 requires a later version of container-selinux
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum-config-manager --disable docker-ce-edge
sudo yum makecache fast
sudo yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.42-1.gitad8f0f7.el7.noarch.rpm
sudo yum install http://mirror.centos.org/centos/7/extras/x86_64/Packages/pigz-2.3.3-1.el7.centos.x86_64.rpm
sudo yum install -y docker-ce
systemctl unmask docker
systemctl unmask docker.socket
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl start docker
docker info
docker version
docker run hello-world
docker ps
```

* Configure your VM's firewall rules
```
$ sudo firewall-cmd --add-port=80/tcp --permanent
$ sudo firewall-cmd --reload
$ sudo firewall-cmd --list-all
```

* Set our timezone
```
$ sudo timedatectl set-timezone Asia/Singapore
```
