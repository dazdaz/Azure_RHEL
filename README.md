## Deploy and configure RHEL 7.5 Server on Azure
#### Adding data disks onto an Azure Linux VM + Re-sizing a data-disk
#### https://docs.microsoft.com/en-us/azure/virtual-machines/linux/tutorial-manage-disks
* /dev/sda OS Disk
* /dev/sdb Temporary Disk
* /dev/sdc Data Disk #1
* /dev/sdd Data Disk #2

<pre>
$ az vm create --location southeastasia --resource-group rhel75-rg --name rhel75 --image RedHat:RHEL:7.3:latest \
  --admin-username dazdaz --admin-password 'SS12345678$$' --size Standard_B1ms --data-disk-sizes-gb 5 5
sudo yum -y update ; sudo reboot
</pre>

* Temporary Disk
<pre>
$ df -h /mnt/resource
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1       3.9G  2.1G  1.7G  56% /mnt/resource

$ swapon -s --all
Filename                                Type            Size    Used    Priority
/mnt/resource/swapfile                  file    2097148 0       -1
</pre>

<pre>
$ dmesg | grep sdc
[    5.670486] sd 5:0:0:0: [sdc] 10485760 512-byte logical blocks: (5.36 GB/5.00 GiB)
[    5.694489] sd 5:0:0:0: [sdc] 4096-byte physical blocks
[    5.852078] sd 5:0:0:0: [sdc] Write Protect is off
[    5.868797] sd 5:0:0:0: [sdc] Mode Sense: 0f 00 10 00
[    5.951575] sd 5:0:0:0: [sdc] Write cache: disabled, read cache: enabled, supports DPO and FUA
[    6.065186] sd 5:0:0:0: [sdc] Attached SCSI disk
</pre>

<pre>
$ dmesg | grep sdd
[    5.713922] sd 5:0:0:1: [sdd] 10485760 512-byte logical blocks: (5.36 GB/5.00 GiB)
[    5.737661] sd 5:0:0:1: [sdd] 4096-byte physical blocks
[    5.868823] sd 5:0:0:1: [sdd] Write Protect is off
[    5.886122] sd 5:0:0:1: [sdd] Mode Sense: 0f 00 10 00
[    5.980540] sd 5:0:0:1: [sdd] Write cache: disabled, read cache: enabled, supports DPO and FUA
[    6.080823] sd 5:0:0:1: [sdd] Attached SCSI disk
</pre>

<pre>
$ (echo n; echo p; echo 1; echo ; echo ; echo w) | sudo fdisk /dev/sdc
$ sudo mkfs -t ext4 /dev/sdc1
$ sudo mkdir /datadrive && sudo mount /dev/sdc1 /datadrive
$ df -h
$ mount
$ sudo -i blkid | grep sdc1
/dev/sdc1: UUID="cda63657-f1a5-4739-b50c-8339768e8ec8" TYPE="ext4"
# Add to /etc/fstab
UUID=cda63657-f1a5-4739-b50c-8339768e8ec8   /datadrive  ext4    defaults,nofail   1  2
</pre>

<pre>
$ az vm open-port --port 80 --resource-group rhel75-rg --name rhel75
</pre>

* Install azure-cli
<pre>
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[azure-cli]\nname=Azure CLI\nbaseurl=https://packages.microsoft.com/yumrepos/azure-cli\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
sudo yum install azure-cli
</pre>

* Install Docker CE 18.03 requires a later version of container-selinux
<pre>
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
</pre>

* Configure your VM's firewall rules
<pre>
$ sudo firewall-cmd --add-port=80/tcp --permanent
$ sudo firewall-cmd --reload
$ sudo firewall-cmd --list-all
</pre>

* Set our timezone
<pre>
$ sudo timedatectl set-timezone Asia/Singapore
</pre>
