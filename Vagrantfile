# -*- mode: ruby -*-
# vim: set ft=ruby :
disk_controller = 'IDE' # MacOS. This setting is OS dependent. Details
MACHINES = {
:zfs => {
:box_name => "centos/7",
:box_version => "2004.01",
:disks => {
:sata1 => {
:dfile => './sata1.vdi',
:size => 512,
:port => 1
},
:sata2 => {
:dfile => './sata2.vdi',
:size => 512, # Megabytes
:port => 2
},
:sata3 => {
:dfile => './sata3.vdi',
:size => 512,
:port => 3
},
:sata4 => {
:dfile => './sata4.vdi',
:size => 512,
:port => 4
},
:sata5 => {
:dfile => './sata5.vdi',
:size => 512,
:port => 5
},
:sata6 => {
:dfile => './sata6.vdi',
:size => 512,
:port => 6
},
:sata7 => {
:dfile => './sata7.vdi',
:size => 512,
:port => 7
},
:sata8 => {
:dfile => './sata8.vdi',
:size => 512,
:port => 8
},
}
},
}
Vagrant.configure("2") do |config|
MACHINES.each do |boxname, boxconfig|
config.vm.define boxname do |box|
box.vm.box = boxconfig[:box_name]
box.vm.box_version = boxconfig[:box_version]
box.vm.host_name = "zfs"
box.vm.provider :virtualbox do |vb|
vb.customize ["modifyvm", :id, "--memory", "1024"]
needsController = false
boxconfig[:disks].each do |dname, dconf|
unless File.exist?(dconf[:dfile])
vb.customize ['createhd', '--filename', dconf[:dfile],
'--variant', 'Fixed', '--size', dconf[:size]]
needsController = true
end
end
if needsController == true
vb.customize ["storagectl", :id, "--name", "SATA",
"--add", "sata" ]
boxconfig[:disks].each do |dname, dconf|
vb.customize ['storageattach', :id, '--storagectl',
'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd',
'--medium', dconf[:dfile]]
end
end
end
box.vm.provision "shell", inline: <<-SHELL
#install zfs repo
yum install -y http://download.zfsonlinux.org/epel/zfs-release.el7_8.noarch.rpm
#import gpg key
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zfsonlinux
#install DKMS style packages for correct work ZFS
yum install -y epel-release kernel-devel zfs
#change ZFS repo
yum-config-manager --disable zfs
yum-config-manager --enable zfs-kmod
yum install -y zfs
#Add kernel module zfs
modprobe zfs
#install wget
yum install -y wget
zpool create otus1 mirror /dev/sdb /dev/sdc
zpool create otus2 mirror /dev/sdd /dev/sde
zpool create otus3 mirror /dev/sdf /dev/sdg
zpool create otus4 mirror /dev/sdh /dev/sdi
zfs set compression=lzjb otus1
zfs set compression=lz4 otus2
zfs set compression=gzip-9 otus3
zfs set compression=zle otus4
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
wget -O archive.tar.gz --no-check-certificate "https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download"
tar xzvf archive.tar.gz
zpool import -d zpoolexport/ otus
wget -O otus_task2.file --no-check-certificate "https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download"
zfs receive otus/test@today < otus_task2.file
SHELL
end
end
end