Trigger a command when a network interface shows up with UDEV
=============================================================

Get info about usb0 net interface

	udevadm test /sys/class/net/usb0

Add udev rule

	nano /etc/udev/rules.d/52-android-modem.rules

	ACTION=="add", ENV{ID_MODEL_ID}=="f00e" , ENV{ID_VENDOR_ID}=="05c6", RUN+="/usr/bin/netctl start usb0"
	ACTION=="remove", ENV{ID_MODEL_ID}=="f00e" , ENV{ID_VENDOR_ID}=="05c6", RUN+="/usr/bin/netctl stop usb0"

Use the same procedure for drives and usb keys

monitor
-------

	udevadm monitor --environment --udev

reload rules
------------

	udevadm control --reload-rules && udevadm trigger

