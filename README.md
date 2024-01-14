# ЗАДАНИЕ: Vagrant стенд для NFS

### Цель домашнего задания:
#### Научиться самостоятельно развернуть сервис NFS и подключить к нему клиента
### Основная часть:
`vagrant up` должен поднимать 2 настроенных виртуальных машины (сервер
NFS и клиента) без дополнительных ручных действий; 
- на сервере NFS должна быть подготовлена и экспортирована директория;
- в экспортированной директории должна быть поддиректория с именем __upload__
с правами на запись в неё;
- экспортированная директория должна автоматически монтироваться на клиенте при
старте виртуальной машины (systemd, autofs или fstab - любым способом);
- монтирование и работа NFS на клиенте должна быть организована с
использованием NFSv3 по протоколу UDP;
- firewall должен быть включен и настроен как на клиенте, так и на сервере.
### *Для самостоятельной реализации:
- настроить аутентификацию через KERBEROS с использованием NFSv4.
### Требуется предварительно установленный и работоспособный [Hashicorp
### Vagrant](https://www.vagrantup.com/downloads) и [Oracle VirtualBox]
### (https://www.virtualbox.org/wiki/Linux_Downloads). Также имеет смысл предварительно
### загрузить образ CentOS 7 2004.01 из Vagrant Cloud командой ```vagrant box add
### centos/7 --provider virtualbox --box version 2004.01 --clean```, т.к. предполагается, что
### дальнейшие действия будут производиться на таких образах.
### Все дальнейшие действия были проверены при использовании CentOS 7.9.2009 в
### качестве хостовой ОС, Vagrant 2.2.18, VirtualBox v6.1.26 и образа CentOS 7 2004.01 из
### Vagrant Cloud. Серьёзные отступления от этой конфигурации могут потребовать
### адаптации с вашей стороны.
### любой работоспособный VPN.
### Создаём тестовые виртуальные машины

### Создаем Vagrantfile по предложенному шаблону
```ruby
# -*- mode: ruby -*- 
# vi: set ft=ruby : vsa
 
Vagrant.configure(2) do |config| 
    config.vm.box = "centos/7" 
    config.vm.box_version = "2004.01" 
    config.vm.provider "virtualbox" do |v| 
      v.memory = 256 
      v.cpus = 1 
end 
    config.vm.define "nfss" do |nfss| 
    nfss.vm.network "private_network", ip: "192.168.56.10",  virtualbox__intnet: "net1" 
    nfss.vm.hostname = "nfss" 
    nfss.vm.provision "shell", path: "nfss_script.sh"
end 
    config.vm.define "nfsc" do |nfsc| 
    nfsc.vm.network "private_network", ip: "192.168.56.11",  virtualbox__intnet: "net1" 
    nfsc.vm.hostname = "nfsc" 
    nfsc.vm.provision "shell", path: "nfsc_script.sh"
  end 
end 
```
### Настраиваем сервер NFS
### Создаем скрипт для сервера с именем nfss_script.sh:
```ruby
#!/bin/bash

sudo yum install nfs-utils -y
sudo systemctl enable firewalld --now
sudo firewall-cmd --add-service="nfs3" --add-service="rpc-bind" --add-service="mountd" --permanent
sudo firewall-cmd --reload
sudo systemctl enable nfs --now
sudo mkdir -p /srv/share/upload
sudo chown -R nfsnobody:nfsnobody /srv/share
sudo chmod 0777 /srv/share/upload
sudo cat << EOF > /etc/exports
/srv/share 192.168.56.10/24(rw,sync,root_squash)
EOF
sudo exportfs -r
sudo touch /srv/share/upload/chek_file
```
### В данном скрипте мы лишь доустанавливаем недостающие утилиты для отладки к уже установленному серверу NFS
> sudo yum install nfs-utils -y
### Далее мы запускаем и настраиваем firewall с уже имеющимися правилами в которых SSH уже разрешен, но при настройке с нуля это стоит уитывать
> sudo systemctl enable firewalld --now \
> sudo firewall-cmd --add-service="nfs3" --add-service="rpc-bind" --add-service="mountd" --permanent \
> sudo firewall-cmd --reload
### включаем сервер NFS (для конфигурации NFSv3 over UDP он не требует дополнительной настройки, однако вы можете ознакомиться с умолчаниями в файле __/etc/nfs.conf__)
> systemctl enable nfs --now
### Затем создаём и настраиваем директорию, которая будет экспортирована в будущем с правами на чтение и запись
> sudo mkdir -p /srv/share/upload \
> sudo chown -R nfsnobody:nfsnobody /srv/share \
> sudo chmod 0777 /srv/share/upload
### Следующим шагом в скрипте создаём в файле '/etc/exports' структуру, которая позволит экспортировать ранее созданную директорию
> sudo cat << EOF > /etc/exports \
> /srv/share 192.168.56.10/24(rw,sync,root_squash) \
> EOF
### экспортируем ранее созданную директорию
> exportfs -r
### И сразу создадим файл в директории
> sudo touch /srv/share/upload/chek_file
### Настраиваем клиент NFS
### Создаем скрипт для клиента с именем nfsс_script.sh:
```ruby
#!/bin/bash

sudo yum install nfs-utils -y
sudo systemctl enable firewalld --now
sudo echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
sudo systemctl daemon-reload
sudo systemctl restart remote-fs.target
sudo touch /mount/upload/client_file
```
### В данном скрипте мы лишь доустанавливаем вспомогательные утилиты
> sudo yum install nfs-utils -y
### добавляем в __/etc/fstab__ строку_ для автомонтирования шары NFS
> sudo echo "192.168.56.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab
### Для перезагрузки измененного файла выполним
> systemctl daemon-reload
### Для того, чтобы автоматически перемонтировать любые новые записи, выполним
> sudo systemctl restart remote-fs.target
### Также создадим файл в расшаренной директории
> sudo touch /mnt/upload/client_file
### ПО окончании отработки скриптов мы получаем полностью подготовленные сервер и клиент:
### Для проверки работоспособности можно войти на клиент
> vagrant up nfsc
### Повышаем привелегии
> sudo -i
### переходим в директорию которая подмонтирована
> cd /mnt/upload
### И вводим 
> mount | grep mnt
### В ответе получим
```ruby
systemd-1 on /mnt type autofs (rw,relatime,fd=46,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=26821)
192.168.56.10:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.56.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.56.10)
```
### И проверяем созданные файлы
> ls -l
### Вывод команды следующий
```ruby
-rw-r--r--. 1 root      root      0 Jan 14 21:03 chek_file
-rw-r--r--. 1 nfsnobody nfsnobody 0 Jan 14 21:05 client_file
```
### Задание выполнено
