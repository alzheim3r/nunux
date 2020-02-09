OpenVPN quick start guide
=========================


## Packages to install

```sh
yaourt -S openvpn easy-rsa
```

## Generate CA and server keypair

### Generate CA cert
```sh
cd /etc/easy-rsa
export EASYRSA=$(pwd)
export EASYRSA_VARS_FILE=/etc/easy-rsa/vars
easyrsa init-pki
easyrsa build-ca nopass
```

copy CA cert to openvpn dir
```sh
cp /etc/easy-rsa/pki/ca.crt /etc/openvpn/server/
```

### Generate openvpn server keypair
```sh
easyrsa gen-req pantoufle nopass
easyrsa sign-req server pantoufle
```

copy server keypair to openvpn dir
```sh
cp /etc/easy-rsa/pki/private/pantoufle.key /etc/openvpn/server/
cp pki/issued/pantoufle.crt /etc/openvpn/server/
```

It is safe to remove reqs/
```sh
rm  /etc/easy-rsa/pki/reqs/*.req
```

### Other crypto stuff
```sh
openssl dhparam -out /etc/openvpn/server/dh.pem 2048
openvpn --genkey --secret /etc/openvpn/server/ta.key
```



## New client

```sh
easyrsa gen-req client1 nopass
easyrsa sign-req client client1
```

client keypair files are in:
- /etc/easy-rsa/pki/private/client1.key
- /etc/easy-rsa/pki/issued/client1.crt



## Revoke a client

```sh
cd /etc/easy-rsa
easyrsa revoke client1
easyrsa gen-crl
cp /etc/easy-rsa/pki/crl.pem /etc/openvpn/server/
```

make sure openvpn server uses the right CRL
```sh
nano /etc/openvpn/server/server.conf
crl-verify /etc/openvpn/server/crl.pem
```


## OpenVPN server config

### Create the config file
```sh
cd /etc/openvpn/server
cp /usr/share/openvpn/examples/server.conf .

nano server.conf
```

Make sure these lines are set as follows:
```
local 5.39.90.190
port 53
ca ca.crt
cert servername.crt
key servername.key
dh dh.pem
tls-auth ta.key 0
user nobody
group nobody
push "redirect-gateway def1 bypass-dhcp"
push "dhcp-option DNS 10.0.0.1"
push "dhcp-option DNS 1.1.1.1"
client-to-client
server 10.5.0.0 255.255.255.0
topology subnet
```

### Start the service
```sh
systemctl start openvpn-server@server
```
*where @server means server.conf*


## Client .ovpn config files

### create template config file
```sh
cd /etc/openvpn/client
nano template.conf
```
```
client
dev tun
proto udp
remote digitalsook.net 53
remote 5.39.90.190     53
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun
mute-replay-warnings
remote-cert-tls server
key-direction 1
cipher AES-256-CBC
#comp-lzo
verb 3
route 10.0.0.0 255.255.255.0
#route-nopull
redirect-gateway def1

```

### generate ovpn file
```sh
cat template.conf > client1.ovpn
echo "<ca>" >> client1.ovpn
cat /etc/openvpn/server/ca.crt >> client1.ovpn
echo "</ca>" >> client1.ovpn
echo "<cert>" >> client1.ovpn
cat /etc/easy-rsa/pki/issued/client1.crt >> client1.ovpn
echo "</cert>" >> client1.ovpn
echo "<key>" >> client1.ovpn
cat /etc/easy-rsa/pki/private/client1.key >> client1.ovpn
echo "</key>" >> client1.ovpn
echo "<tls-auth>" >> client1.ovpn
cat /etc/openvpn/server/ta.key >> client1.ovpn
echo "</tls-auth>" >> client1.ovpn
```

### generate openvpn package (alternative method)
```sh
cd /etc/openvpn/client
mkdir client1
cat template.conf | sed '/^#/ d' > client1/client1.ovpn
echo -e "ca ca.crt\ncert client1.crt\nkey client1.key\ntls-auth ta.key 1" >> client1/client1.ovpn
cp /etc/openvpn/server/ca.crt client1/
cp /etc/easy-rsa/pki/issued/client1.crt client1/
cp /etc/easy-rsa/pki/private/client1.key client1/
cp /etc/openvpn/server/ta.key client1/
zip -r client1.zip client1
rm -rf client1
```



## Give a client a static IP

### enable client specific config
```sh
nano /etc/openvpn/server/server.conf
```
```
client-config-dir ccd
```

### create client config
```sh
mkdir /etc/openvpn/client-config-dir
chmod o+rx /etc/openvpn/server
chmod o+rx /etc/openvpn/client-config-dir
touch /etc/openvpn/client-config-dir/client1
chmod o+r /etc/openvpn/client-config-dir/client1
```
set client's static IP
```sh
nano /etc/openvpn/client-config-dir/client1
```
```
ifconfig-push 10.5.0.2 255.255.255.192
```
Config filename must match the client certificate's CN


## Custom iptables rules per client

Because we use iptables, openvpn can no longer be started with no privileges. As a result, these lines must be commented out from the config file as follows..
```sh
nano /etc/openvpn/server/server.conf
```
```
;user nobody
;group nobody
```

And those lines musst be added at the end of the same config file..
```
script-security 2
client-connect    /etc/openvpn/scripts/up-client.sh
client-disconnect /etc/openvpn/scripts/down-client.sh
```

### create scripts
```sh
mkdir /etc/openvpn/server/scripts
chmod o+rx /etc/openvpn/server/scripts
touch /etc/openvpn/scripts/up-client.sh  /etc/openvpn/scripts/down-client.sh
chmod o+rx /etc/openvpn/scripts/*
```

Setup port forwarding when connecting

```sh
nano /etc/openvpn/scripts/up-client.sh
```
```sh
#!/bin/sh

echo " >> CONNECTION FROM $common_name - LOCAL IP: $ifconfig_pool_remote_ip"

case "$common_name" in
   (client1)
        echo " >> FORWARDING PORTS 10000 TO 10009"
        iptables -t nat -A PREROUTING -p tcp -m tcp --dport 10000:10009 -j DNAT --to-destination $ifconfig_pool_remote_ip
        ;;
   (*)
        echo " >> No script for $common_name"
        ;;
esac
```


Disable NAT rules when disconnecting

```sh
nano /etc/openvpn/scripts/down-client.sh
```
```sh
#!/bin/sh

echo " >> DISCONNECTION FROM $common_name - LOCAL IP: $ifconfig_pool_remote_ip"

case "$common_name" in
   (client1)
        echo " >> REMOVING NAT RULE"
        iptables -t nat -D PREROUTING -p tcp -m tcp --dport 10000:10009 -j DNAT --to-destination $ifconfig_pool_remote_ip
        ;;
   (*)
        echo " >> No script for $common_name"
        ;;
esac
```


## Bash scripts


### /etc/openvpn/client/generate_client_ovpn.sh
```sh
#!/bin/bash

cd /etc/openvpn/client || exit 2
CN=$1
MODE=$2

function make_ovpn {
        cat template.conf > $CN.ovpn
        echo "<ca>" >> $CN.ovpn
        cat /etc/openvpn/server/ca.crt >> $CN.ovpn
        echo "</ca>" >> $CN.ovpn
        echo "<cert>" >> $CN.ovpn
        cat /etc/easy-rsa/pki/issued/$CN.crt >> $CN.ovpn
        echo "</cert>" >> $CN.ovpn
        echo "<key>" >> $CN.ovpn
        cat /etc/easy-rsa/pki/private/$CN.key >> $CN.ovpn
        echo "</key>" >> $CN.ovpn
        echo "<tls-auth>" >> $CN.ovpn
        cat /etc/openvpn/server/ta.key >> $CN.ovpn
        echo "</tls-auth>" >> $CN.ovpn
}

function make_package {
        mkdir $CN
        cat template.conf > $CN/$CN.ovpn
        echo -e "ca ca.crt\ncert $CN.crt\nkey $CN.key\ntls-auth ta.key 1" >> $CN/$CN.ovpn
        cp /etc/openvpn/server/ca.crt $CN/
        cp /etc/easy-rsa/pki/issued/$CN.crt $CN/
        cp /etc/easy-rsa/pki/private/$CN.key $CN/
        cp /etc/openvpn/server/ta.key $CN/
        zip -r $CN.zip $CN
        rm -rf $CN
}

if [ -z "$CN" ]
then
        echo "  :: Error: You must provide a Common Name (CN)"
        echo "  :: Syntax: $(basename $0) CN [package]"
        exit 2
fi

if [ -f "/etc/easy-rsa/pki/issued/$CN.crt" ] && [ -f "/etc/easy-rsa/pki/private/$CN.key" ]
then
        case "$MODE" in
           (package)
                        make_package
                        echo "  :: $CN.zip ready"
                        ;;
           (*)
                        make_ovpn
                        echo "  :: $CN.ovpn ready"
                        ;;
        esac
else
        echo "  :: Error: There is no cert for given CN"
        exit 2
fi
```


### /etc/openvpn/client/new_client.sh
```sh
#!/bin/bash

cd /etc/easy-rsa || exit 2
CN=$1

if [ -z "$CN" ]
then
        echo "  :: Error: You must provide a Common Name (CN)"
        echo "  :: Syntax: $(basename $0) CN"
        exit 2
fi

if [ -f "/etc/easy-rsa/pki/ca.crt" ]
then

        if [ ! -f "/etc/easy-rsa/pki/private/$CN.key" ] && [ ! -f "/etc/easy-rsa/pki/issued/$CN.crt" ]
        then
                easyrsa gen-req $CN nopass
                easyrsa sign-req client $CN
                # rm /etc/easy-rsa/pki/reqs/$CN.req
        else
                echo "  :: Error: $CN already exist"
                exit 2
        fi
else
        echo "  :: Error: CA cert not found"
        exit 2
fi
```

### /etc/openvpn/client/revoke_client.sh
```sh
#!/bin/bash

cd /etc/easy-rsa || exit 2
CN=$1

if [ -z "$CN" ]
then
        echo "  :: Error: You must provide a Common Name (CN)"
        echo "  :: Syntax: $(basename $0) CN"
        exit 2
fi

if [ -f "/etc/easy-rsa/pki/private/$CN.key" ] && [ -f "/etc/easy-rsa/pki/issued/$CN.crt" ]
then

        easyrsa revoke $CN
        easyrsa gen-crl
        cp /etc/easy-rsa/pki/crl.pem /etc/openvpn/server/
        rm /etc/easy-rsa/pki/private/$CN.key
        rm /etc/easy-rsa/pki/issued/$CN.crt
        rm /etc/easy-rsa/pki/reqs/$CN.req
        systemctl restart openvpn-server@pantoufle

else
        echo "  :: Error: cert not found"
        exit 2
fi
```

## No internet? Don't forget NAT

on the VPN server
```sh
sysctl -w net.ipv4.ip_forward=1
iptables -P FORWARD ACCEPT
iptables --table nat -A POSTROUTING -o eth0 -j MASQUERADE
```
