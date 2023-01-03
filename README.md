#  Практические навыки работы с ZFS

Задание:
```text
    1.Определить алгоритм с наилучшим сжатием.
    2. Определить настройки pool’a.
    3. Найти сообщение от преподавателей.
```


Выполнение:

Развернём виртуальную машину с помощью преднастроенного Vagrantfile:

```ruby
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
```
Данный Vagrantfile выполняет все настройки описанные ниже. Вывод произведён и скопирован дополнительно. 


1.	Определить алгоритм с наилучшим сжатием.

Создаём четыре пула из восьми дисков в режиме RAID 1 и добавим разные алгоритмы сжатия в каждую файловую систему:

```bash
zpool create otus1 mirror /dev/sdb /dev/sdc
zpool create otus2 mirror /dev/sdd /dev/sde
zpool create otus3 mirror /dev/sdf /dev/sdg
zpool create otus4 mirror /dev/sdh /dev/sdi
zfs set compression=lzjb otus1
zfs set compression=lz4 otus2
zfs set compression=gzip-9 otus3
zfs set compression=zle otus4
```

Скачаем один и тот же текстовый файл во все пулы:

```bash
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
```

Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:

```text
[root@zfs ~]# zfs list
NAME             USED  AVAIL     REFER  MOUNTPOINT
otus            4.96M   347M       25K  /otus
otus/hometask2  1.88M   347M     1.88M  /otus/hometask2
otus/test       2.85M   347M     2.83M  /otus/test
otus1           21.6M   330M     21.5M  /otus1
otus2           17.7M   334M     17.6M  /otus2
otus3           10.8M   341M     10.7M  /otus3
otus4           39.2M   313M     39.1M  /otus4

```

```text
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus             compressratio         1.23x                  -
otus/hometask2   compressratio         1.00x                  -
otus/test        compressratio         1.32x                  -
otus/test@today  compressratio         1.32x                  -
otus1            compressratio         1.81x                  -
otus2            compressratio         2.22x                  -
otus3            compressratio         3.64x                  -
otus4            compressratio         1.00x                  -
```

Таким образом, у нас получается, что алгоритм gzip-9 самый эффективный
по сжатию.

2. 	Определить настройки pool’a.


Скачиваем архив в домашний каталог:

```bash
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
wget -O archive.tar.gz --no-check-certificate "https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download"
```

Разархивируем его:

```bash
tar xzvf archive.tar.gz
```

Сделаем импорт данного пула к нам в ОС:

```bash
zpool import -d zpoolexport/ otus
```

Команда zpool status выдаст нам информацию о составе импортированного
пула:
```text
[root@zfs ~]# zpool status
  pool: otus
 state: ONLINE
  scan: none requested
config:

	NAME                                 STATE     READ WRITE CKSUM
	otus                                 ONLINE       0     0     0
	  mirror-0                           ONLINE       0     0     0
	    /home/vagrant/zpoolexport/filea  ONLINE       0     0     0
	    /home/vagrant/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors
```
3. 	Найти сообщение от преподавателей.

Скачаем файл, указанный в задании:

```bash
wget -O otus_task2.file --no-check-certificate "https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download"
```
Восстановим файловую систему из снапшота: 

```bash
zfs receive otus/test@today < otus_task2.file
```
Далее, ищем в каталоге /otus/test файл с именем “secret_message”:

```text
[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
```
Смотрим содержимое найденного файла:

```text
[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://github.com/sindresorhus/awesome
```

