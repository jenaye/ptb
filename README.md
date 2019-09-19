# PTB using RPI

# Overview 

<img width="1280" alt="homepage" src="https://user-images.githubusercontent.com/15458329/65266855-29492880-db14-11e9-857b-fb1e938a1781.jpeg">

### requirements : 

- RPI 
    - openvpn
    - iptables-persistent
- VPS
    - openvpn
    - iptables-persistent
    - tools

*Note : CA server and Openvpn Server are the same ( its the VPS )*

 On both (`VPS/RPI`) u should edit `sudo nano /etc/sysctl.conf` and set net.ipv4.ip_forward to `1`
 then use `sudo sysctl -p` to verify if it worked

 *Every commmand are executed on VPS* 

# How to use 

just add the route of the target network for exemple : `ip route add 192.168.1.0/24 via 10.0.0.2`
here `10.0.0.2` are the ip of `rpi` from `tun0`
then u can try to ping machine on victim local network


# Install 


### Generate ca.crt and server.crt

``` 
wget -P ~/ https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz
cd ~
tar xvf EasyRSA-3.0.4.tgz
cd ~/EasyRSA-3.0.4/
cp vars.example vars
nano vars # change set_var EASYRSA_REQ_* by what u want and uncomment
./easyrsa init-pki
./easyrsa build-ca nopass ( just press enter )
``` 

Now we have `ca.crt`

``` 

./easyrsa gen-req server nopass
sudo cp ~/EasyRSA-3.0.4/pki/private/server.key /etc/openvpn/
./easyrsa sign-req server server # yes

```

Now copy `server.crt`, `ca.crt` to `/etc/openvpn`

### Generate dh.pem and ta.key


```
./easyrsa gen-dh
openvpn --genkey --secret ta.key
``` 

Now copy `ta.key` and `dh.pem` to /etc/openvpn


`mkdir -p ~/client-configs/keys` 

`chmod -R 700 ~/client-configs`


### Generate client1.key

./easyrsa gen-req client1 nopass

copy `client1.key`  to `/etc/openvpn`

sign it 

`./easyrsa sign-req client client1`  say yes

now we have `client1.crt` 

```

sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
sudo gzip -d /etc/openvpn/server.conf.gz

``` 

Copy this to u're `server.conf` in /etc/openvpn

```
ifconfig 10.10.10.1 10.10.10.2
proto tcp-server
port 4443
dev tun0
#ifconfig-pool-persist ipp.txt
# Cles et certificats
#tun-mtu 1400
#reneg-sec 120
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem
tls-server
#secret /etc/openvpn/ta.key
tls-auth /etc/openvpn/ta.key 0
cipher AES-256-CBC
#keepalive 10 60
keepalive 10 600
#tls-timeout 300
# Securite
user nobody
group nogroup
#chroot /etc/openvpn/jail
persist-key
persist-tun
comp-lzo
# Log
verb 5
#mute 20
status openvpn-status.log
log-append /var/log/openvpn.log
``` 

run this on both ( vps/rpi ) :  `iptables -t nat -A POSTROUTING -s 0.0.0.0/0 -j MASQUERADE`


To run openvpn service at start `sudo systemctl enable openvpn@server`  (`server` its the xxx.conf into /etc/openvpn)

```
mkdir -p ~/client-configs/files
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf
nano ~/client-configs/base.conf
```

replace by this : 

```
 
dev tun0
ifconfig 10.10.10.2 10.10.10.1
proto tcp-client
remote 163.ZZZ.YYY.XXX 4443  # change this
resolv-retry infinite
cipher AES-256-CBC

# Clés
tls-client
ca /etc/openvpn/ca.crt
cert /etc/openvpn/client1.crt
key /etc/openvpn/client1.key
tls-auth /etc/openvpn/ta.key 1
keepalive 10 600
nobind

# Securite
persist-key
persist-tun
comp-lzo

# log
verb 3


```


Create script to generate openvpn file

`nano ~/client-configs/make_config.sh`

add this into : 

```

#!/bin/bash

# First argument: Client identifier

KEY_DIR=~/client-configs/keys
OUTPUT_DIR=~/client-configs/files
BASE_CONFIG=~/client-configs/base.conf

cat ${BASE_CONFIG} \
    <(echo -e '<ca>') \
    ${KEY_DIR}/ca.crt \
    <(echo -e '</ca>\n<cert>') \
    ${KEY_DIR}/${1}.crt \
    <(echo -e '</cert>\n<key>') \
    ${KEY_DIR}/${1}.key \
    <(echo -e '</key>\n<tls-auth>') \
    ${KEY_DIR}/ta.key \
    <(echo -e '</tls-auth>') \
    > ${OUTPUT_DIR}/${1}.ovpn

``` 

add right  : `chmod 700 ~/client-configs/make_config.sh`

`sudo ./make_config.sh client1` => output `client1.ovpn` in folder `files`

Now we have config file, we just need to push it on RPI ( by scp or other ) in folder `/etc/openvpn ` and rename file `client1.ovpn` to `client1.conf`

Now u can connect using this command `sudo openvpn client1.ovpn` and u'll be connected 

Also, u can run this when rpi boot  using this command `sudo systemctl enable openvpn@client1`


# Advanced 

iptables-save -c > /etc/iptables/rules.v4


# Verify

All theses file should be in `/etc/openvpn` of the VPS : 

```
- ca.crt
- dh.pem
- xxx.conf
- server.crt
- server.key
- ta.key
``` 

if u have it, just run  `tcpdump port 4443` on VPS, then try to connect on it via RPI

u can also verify if connexion are done `netstat -laputen | grep ESTABLISHED` 


inspired by https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-ubuntu-18-04
