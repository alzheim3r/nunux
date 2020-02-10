Using a network namespace for VPN torrenting
=====================================

Using a VPN for torrenting is a good idea but what if you don't want all your internet trafic to go through the VPN? Here we'll explain how to use a vpn for all bittorent traffic while using your *'normal'* internet for other stuff.


## Enable Internet connection sharing

	sysctl -w net.ipv4.ip_forward=1
	iptables -P FORWARD ACCEPT
	iptables --table nat -A POSTROUTING -o eth0 -j MASQUERADE

## Create the torrent namespace

	ip netns add torrent

### Check

	ip netns list

## Create virtual network interfaces

They'll work like a pipe between the global and the torrent namespace.

- **veth0** will be on the global namespace side, 
- **veth1** will be on the torrent namespace side.
  
```sh
	ip link add veth0 type veth peer name veth1
	ip link set veth1 netns torrent
```

### Check:

should show **veth0** and not **veth1**

	ifconfig  

should show **veth1**

	ip netns exec torrent ifconfig -a  



## Wakeup and setup the net inerfaces

	ifconfig veth0 up
	ip netns exec torrent ifconfig veth1 up
	ip addr add 10.10.10.1/24 dev veth0
	ip netns exec torrent ip addr add 10.10.10.2/24 dev veth1
	ip netns exec torrent ip route add default via 10.10.10.1


### Check

	ip netns exec torrent ifconfig
	ip netns exec torrent ping 10.10.10.1
	ip netns exec torrent ping 8.8.8.8
	ip netns exec torrent ping google.com


##  Start openvpn client in the new namespace

	ip netns exec torrent openvpn --daemon --config myvpn.ovpn


### Check

	ip netns exec torrent curl curlmyip.org



## Launch Transmission

	ip netns exec torrent su username -c "source /etc/profile;source /home/username/.bashrc;export QT_STYLE_OVERRIDE="breeze";export KDE_SESSION_VERSION=5;export KDE_FULL_SESSION=true;transmission-qt"

The ``su username -c`` is necessary otherwize the application will be launched as root.



## Double check Transmission IP

Open This magnet link: ```magnet:?xt=urn:btih:2fa71a2dbb7d53a39373a9c4e2c9d89aaa7a6db1&dn=checkMyTorrentIp.png&tr=http%3A%2F%2Fcheckmytorrentip.net%2Ftorrentip%2Fannounce.php```

 The tracker should show an error showing the client IP address.




## Shell in the namespace

	sudo ip netns exec torrent bash 
	sudo ip netns exec torrent su username -c bash



## Delete the namespace

	sudo pkill openvpn

	sudo ip netns exec torrent ifconfig veth1 down
	sudo ip netns exec torrent ip link delete veth1

	# should not exist
	sudo ifconfig veth0 down
	sudo ip link delete veth0

	sudo ip netns del torrent



## Source

http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/