Trigger a command when a network interface shows up with UDEV
=============================================================

So you want to connect your linux PC to the internet using your mobile and a USB cable?  No problem! Most of the times, a USB net interface pops up in your linux as soon as USB thetering is enabled on your android. Assuming this iface is named 'usb0', the only thing you need to do is a good old ```dhcpcd usb0``` and you're good to go.

Now what if you're one of those wierd Arch-user-nerds not using NetworkManager but still want to automate this process anyway?  

## Create netctl profile

	nano /etc/netctl/usb0
	...
	Description='DHCP connection'
	Interface=usb0
	Connection=ethernet
	IP=dhcp
	DNS=('1.1.1.1')

## Get info about usb0 net interface

	udevadm test /sys/class/net/usb0

## Add udev rule

	nano /etc/udev/rules.d/52-android-modem.rules
    ...
	ACTION=="add", ENV{ID_MODEL_ID}=="f00e" , ENV{ID_VENDOR_ID}=="05c6", RUN+="/usr/bin/netctl start usb0"
	ACTION=="remove", ENV{ID_MODEL_ID}=="f00e" , ENV{ID_VENDOR_ID}=="05c6", RUN+="/usr/bin/netctl stop usb0"

## monitor

	udevadm monitor --environment --udev

## reload rules

	udevadm control --reload-rules && udevadm trigger

Same procedure can be adapted to any other USB device you may connect to your linux host (ie. usb drives)

eNJOy ;)