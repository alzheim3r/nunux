Crach/Dismount/Freeze when copying big file to an external USB disc
===================================================================

Observed on a raspberry pi 4B and jmicron usb-sata bridge.

This is a UAS driver problem. UAS must be disabled in usb-storage module.

Got your ID? 
------------

- module option syntax: ```quirks=Vendor_ID:Product_ID:u```
- Get Vendor_ID & Product_ID: ```lsusb -v```


If usb-storage is a module
--------------------------

	nano /etc/modprobe.d/ignore_uas.conf 
	...
	options usb-storage quirks=0x152d:0x8561:u



If usb-storage is built in the kernel
-------------------------------------

Use kernel parameter:  usb-storage.quirks=0x152d:0x8561:u


### Raspberry pi:

	nano /boot/cmdline.txt
	...
	usb-storage.quirks=0x152d:0x8561:u


Check
-----

	lsusb -v -d 0x152d:0x8561 | grep uas
	dmesg | grep uas
