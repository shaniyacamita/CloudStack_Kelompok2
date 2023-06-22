# CloudStack_Kelompok2

## Installation Cloud Stack 4.17 on Ubuntu 22.04 Kelompok 2
#### Binar Qalbu Cimuema
#### Mohammad Darrel Tristan Budiroso
#### Raihan Jana Prasetya
#### Shaniya Camita Farin
### Home Network 192.168.10.0/24

##### Network configuration with netplan
###### Rename all existing configuration by adding .bak
maradens@dtecloud:~$ cat /etc/netplan/01-netcfg.yaml
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
         addresses: [1.1.1.1,8.8.8.8]
       interfaces: [eno1]
       dhcp4: false
       dhcp6: false
       parameters:
         stp: false
         forward-delay: 0
