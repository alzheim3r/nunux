Using a network namespace for VPN torrenting
=====================================

## Source
http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/


## Setup a bridge

First setup a network bridge.This totorial assumes that the bridge **br0** is already up and running. **#RTFM**


## Create the namespace: 'torrent'
	ip netns add torrent

### check
	ip netns list


## Create virtual network interfaces

They'll work like a pipe between the global and the torrent namespace:

- **veth0** will be on the global namespace side, 
- **veth1** will be on the torrent namespace side.


	ip link add veth0 type veth peer name veth1
	ip link set veth1 netns torrent


### Control:

should show the bridge **br0** and **veth0** 

	ifconfig  

should show **veth1**

	ip netns exec torrent ifconfig -a  



## Wakeup and setup the net inerfaces

	ifconfig veth0 up
	ip netns exec torrent ifconfig veth1 up


And add **veth0** to the bridge

	brctl addif br0 veth0

### Control
	brctl show


### Give the torrent namespace an IP address

Here we're using DHCP to get an IP:

	ip netns exec torrent dhcpcd veth1

To assign a fixed IP, use: 

	ip netns exec torrent ip addr add 192.168.0.12/24 broadcast 192.168.0.255 dev veth1
	ip netns exec torrent ip route add default via 192.168.0.1



### Note: 
On Ubuntu, dhclient takes a long time for some misterious reason. Choosing a manual IP config works too but veth1 will take about 1 minute to respond.

### Control

	ip netns exec torrent ifconfig         # Should report a valid IP
	ip netns exec torrent ping 192.168.0.1 # Ping the router
	ip netns exec torrent ping 8.8.8.8     # google DNS
	ip netns exec torrent ping google.com


##  Start openvpn in the new namespace
Openvpn config not covered here.** #RTFM**

### Note:
Remove * --daemon*  for testing.

	ip netns exec torrent openvpn --daemon --config /home/cif/openvpn/seedbox-cif.conf


### Control:

	ip netns exec torrent  wget http://ipinfo.io/ip -qO -  # should show openVPN's IP


## DONE!

Launch programs in the new netspace using the examples below: 

	sudo ip netns exec torrent bash 
	sudo ip netns exec torrent su cif -c bash
	sudo -n ip netns exec torrent sudo -n -u cif -g cif transmission-gtk 


### Double check the bittorrent's client IP
Follow [This magnet link](magnet:?xt=urn:btih:2fa71a2dbb7d53a39373a9c4e2c9d89aaa7a6db1&dn=checkMyTorrentIp.png&tr=http%3A%2F%2Fcheckmytorrentip.net%2Ftorrentip%2Fannounce.php "This magnet link"). 

The tracker should show an error showing the client IP address.




## Delete the namespace

	sudo pkill openvpn

	sudo ip netns exec torrent ifconfig veth1 dhcpcd -x
	sudo ip netns exec torrent ifconfig veth1 down
	sudo ip netns exec torrent ip link delete veth1

	# should not exist
	sudo brctl delif br0 veth0
	sudo ifconfig veth0 down
	sudo ip link delete veth0

	sudo ip netns del torrent
