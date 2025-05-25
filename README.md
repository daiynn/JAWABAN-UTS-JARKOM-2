# JAWABAN-UTS-JARKOM-2
#1. DHCP SERVER

#set ip enp0s3
sudo nano /etc/netplan/00-installer-config.yaml

ubah ke:
network: 
  version: 2 
  ethernets: 
    enp0s3: 
      addresses: [192.168.10.1/24] 
      gateway4: 192.168.10.1 
      nameservers: 
        addresses: [8.8.8.8] 
    enp0s8: 
      dhcp4: no 

lalu simpan dengan :
sudo netplan apply

#instal DHCP server
sudo apt update 
sudo apt install isc-dhcp-server 

file konfigurasi: 
sudo nano /etc/dhcp/dhcpd.conf 
edit ke: 
default-lease-time 600; 
max-lease-time 7200; 
authoritative; 
subnet 192.168.10.0 netmask 255.255.255.0 { 
 range 192.168.10.100 192.168.10.200; 
 option routers 192.168.10.1; 
 option domain-name-servers 192.168.10.1; 
 option domain-name "server.local"; 
} 

Edit interfacenya : 
sudo nano /etc/default/isc-dhcp-server 
ganti ke: 
INTERFACESv4="enp0s3"

setelah itu restart semua:
sudo systemctl restart isc-dhcp-server 
sudo systemctl enable isc-dhcp-server 

kemudian tes di vm client:
karna saya memakai 2 adapter kita pastikan enp0s8 dalam keadaan UP: 
sudo ip link set enp0s8 up 
Kemudian jalankan DHCP client: 
sudo dhclient enp0s8 
Cek IP-nya: 
ip a 



#2. DNS server
instal bind9
sudo apt update
sudo apt install bind9 -y 

edit file netplan
sudo nano /etc/netplan/00-installer-config.yaml 

configurasinya:
network: 
  version: 2 
  ethernets: 
    enp0s3: 
      dhcp4: true 
    enp0s8: 
      addresses: 
        - 192.168.56.1/24 
      dhcp4: no 
      optional: true 

Lalu apply: 
sudo chmod 600 /etc/netplan/*.yaml 
sudo netplan apply 
sudo ip link set enp0s8 up 

konfigurasi bind
sudo nano /etc/bind/named.conf.local 

tambahkan:
zone "derian.der" { 
     type master; 
     file "/etc/bind/db.derian.der"; 
}; 
zone "56.168.192.in-addr.arpa" { 
     type master; 
     file "/etc/bind/db.192"; 
}; 

Salin dan edit file zona 
Buat file zona untuk derian.der 
sudo cp /etc/bind/db.local /etc/bind/db.derian.der 
sudo nano /etc/bind/db.derian.der 

isinya:
; 
; BIND data file for derian.der 
; 
$TTL    604800 
@       
IN      SOA     derian.der. root.derian.der. ( 
2         ; Serial 
604800         ; Refresh 
86400         ; Retry 
2419200         ; Expire 
604800 )       ; Negative Cache TTL 
; 
@       IN      NS      derian.der. 
@       IN      A       192.168.56.1 
www     IN      A       192.168.56.1 

buat file reverse zone
sudo cp /etc/bind/db.127 /etc/bind/db.192 
sudo nano /etc/bind/db.192 
Edit: 
; 
; BIND reverse data file for 192.168.56.0/24 
; 
$TTL    604800 
@       IN      SOA     derian.der. root.derian.der. ( 
2         ; Serial 
604800         
; Refresh 
86400         ; Retry 
2419200         ; Expire 
604800 )       ; Negative Cache TTL 
; 
@       IN      NS      derian.der. 
1       IN      PTR     derian.der.

Cek konfigurasi dan restart BIND 
sudo named-checkconf 
sudo named-checkzone derian.der /etc/bind/db.derian.der 
sudo named-checkzone 56.168.192.in-addr.arpa /etc/bind/db.192 
sudo systemctl restart bind9 


#buka vm client
Atur DNS client
Edit file resolv.conf: 
sudo nano /etc/resolv.conf 
Isi: 
nameserver 192.168.56.1 

kemudian uji coba
ping derian.der 
nslookup derian.der 
dig derian.der 


#3. firewall iptables

bloki ping dari luar subnet 
sudo iptables -A INPUT -p icmp -j DROP 

Izinkan port 22 (SSH), 80 (HTTP), 443 (HTTPS) 
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT 
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT 
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT 

Tampilkan aturan 
sudo iptables -L -v 


