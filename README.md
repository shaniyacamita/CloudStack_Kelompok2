# CloudStack_Kelompok2
This guide provides step-by-step instructions to install CloudStack 4.17 on Ubuntu 22.04.

## Installation Cloud Stack 4.17 on Ubuntu 22.04 Kelompok 2
#### Binar Qalbu Cimuema
#### Mohammad Darrel Tristan Budiroso
#### Raihan Jana Prasetya
#### Shaniya Camita Farin
### Home Network 192.168.10.0/24

## Install ssh server and others tool if not yet present
```
sudo apt-get install openntpd openssh-server sudo vim htop tar
sudo apt-get install intel-microcode
sudo passwd root
```
## Install ssh server and others tool if not yet present
### Buka file /etc/ssh/sshd_config:
```
sudo nano /etc/ssh/sshd_config
```
### Temukan baris PermitRootLogin dan ubah nilainya menjadi yes.
### Restart layanan SSH:
```
sudo service ssh restart
```

## Network configuration with netplan
### Rename all existing configuration by adding .bak extension
```
cat /etc/netplan/01-netcfg.yaml
```
### Edit isi konfigurasi netplan
```
sudo nano /etc/netplan/01-netcfg.yaml
```
### Gantikan isi file dengan konfigurasi berikut:
```
network:
  version: 2
  renderer: networkd
  ethernets:
    eno1:
      dhcp4: false
      dhcp6: false
      optional: true
    eno2:
      dhcp4: false
      dhcp6: false
      optional: true
  bridges:
    cloudbr0:
      addresses: [192.168.10.100/24]
      routes:
       - to: default
         via: 192.168.10.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
      interfaces: [eno1]
      dhcp4: false
      dhcp6: false
      parameters:
        stp: false
        forward-delay: 0
```
### Terapkan perubahan konfigurasi jaringan:
```
sudo netplan generate
sudo netplan apply
sudo reboot
```
## Update your system and install useful tools
```
sudo apt update
sudo apt install htop lynx duf bridge-utils -y
```
## Configure LVM
### Extend the LVM volume group with additional disks:
```
sudo vgextend ubuntu-vg /dev/sda
sudo vgextend ubuntu-vg /dev/sdb
```
### Extend the logical volume size:
```
sudo lvextend -L +100G /dev/ubuntu-vg/ubuntu-lv
```
### Resize the file system:
```
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```
## CloudStack Installation
Silakan ikuti petunjuk yang diberikan dalam posting blog berikut:
[Apache CloudStack on Ubuntu with x86_64 KVM](https://rohityadav.cloud/blog/cloudstack-kvm/)

## CloudStack Management Server Setup
```
mkdir -p /etc/apt/keyrings
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
```
```
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.17 / > /etc/apt/sources.list.d/cloudstack.list
```
```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

#CloudStack usage and billing (optional)
```
apt-get install cloudstack-usage 
```

#Configure database
```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
```
[mysqld]
server-id = 1
sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
innodb_rollback_on_timeout=1
innodb_lock_wait_timeout=600
max_connections=1000
log-bin=mysql-bin
binlog-format = 'ROW'
```
```
systemctl restart mysql
```

#deploy database as root and then create cloud user with password cloud too
```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:password -i 192.168.10.22
```

#Storage Setup
```
apt-get install nfs-kernel-server quota
```
```
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```

#Configure NFS server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```

## Setup KVM
#Setup KVM host and cloudstack agent
```
apt-get install qemu-kvm cloudstack-agent
```
```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf
```
```
nano /etc/default/libvirtd
```

#On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.
#uncomment LIBVIRTD_ARGS="--listen"

#Configure default libvirtd config:
```
echo 'listen_tls=0' >> /etc/libvirt/libvirtd.conf
echo 'listen_tcp=1' >> /etc/libvirt/libvirtd.conf
echo 'tcp_port = "16509"' >> /etc/libvirt/libvirtd.conf
echo 'mdns_adv = 0' >> /etc/libvirt/libvirtd.conf
echo 'auth_tcp = "none"' >> /etc/libvirt/libvirtd.conf
```
```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```

#On certain hosts where you may be running docker and other services, you may need to add the following in /etc/sysctl.conf and then run sysctl -p:
```
nano /etc/sysctl.conf
```
```
net.bridge.bridge-nf-call-arptables = 0
net.bridge.bridge-nf-call-iptables = 0
sysctl -p
```

#generate host id
```
apt-get install uuid
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

#configure firewall (OPTIONAL)
# configure iptables
```
NETWORK=192.168.1.0/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT
```
```
apt-get install iptables-persistent
```
#yes yes

# Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

#Launch Management Server
#Start your cloud:
```
cloudstack-setup-management
```
```
systemctl status cloudstack-management
```
```
tail -f /var/log/cloudstack/management/management-server.log
```

#After management server is UP, proceed to http://192.168.10.22(i.e. the cloudbr0-IP):8080/client and log in using the default credentials - username admin and password password.

#Enable XRDP
#https://www.digitalocean.com/community/tutorials/how-to-enable-remote-desktop-protocol-using-xrdp-on-ubuntu-22-04
```
apt update
apt install xfce4 xfce4-goodies -y
apt install xrdp -y
```
#configure to allow tcp ipv4 listen to 3389. It's a bug only listen to tcp6
#port=tcp://:3389
```
nano /etc/xrdp/xrdp.ini
```
```
systemctl restart xrdp
```
```
systemctl status xrdp
```
