ruTorrent multi-user and Nginx
==============================

In this doc we assume you want to intall a full muti-user rutorrent LXC container using Arch Linux. Installing LXC and making a LXC container isn't covered here.

Install php-fpm
---------------

	yaourt -S php-fpm
	systemctl start php-fpm
	systemctl enable php-fpm

.. et c'est tout!


Install nginx
-------------

	yaourt nginx-mainline
	systemctl start nginx
	systemctl enable nginx

Check the welcome page shows up: http://10.0.0.10


### Add php support 

	nano /etc/nginx/nginx.conf

and add this into the server block

	location ~ [^/]\.php(/|$) {
		fastcgi_split_path_info ^(.+?\.php)(/.*)$;

		try_files $uri $document_root$fastcgi_script_name =404;
		fastcgi_param HTTP_PROXY "";

		fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
		fastcgi_index index.php;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		include fastcgi_params;
		}

make a info.php page..

	cd /usr/share/nginx/html
	nano info.php
	...
	<?php phpinfo(); ?>

save and test..

	nginx -t && systemctl reload nginx 

Check phpinfo is ok: http://10.0.0.10/info.php

From now on you should have a working nginx and php-fpm. Let's leave it aside for now and install rtorrent..



Install rtorrent
----------------

	cp /usr/share/doc/rtorrent/rtorrent.rc /home/user1/.rtorrent.rc

### edit the config file:

	directory.default.set = /home/user1/downloads
	session.path.set = /home/user1/session
	network.port_range.set = 12100-12198
	network.port_random.set = yes
	dht.port.set = 12199
	scgi_port = 127.0.0.1:5001

### test it:

	mkdir -p  /home/user1/downloads /home/user1/session
	chmod 755 /home/user1/session
	chown -R user1:pantouflards /home/user1

	sudo -u user1 rtorrent

### create service:

	nano/etc/systemd/system/rtorrent@.service
	...
	[Unit]
	Description=rTorrent for %I
	After=network.target

	[Service]
	Type=simple
	User=%I
	Group=%I
	KillMode=none
	WorkingDirectory=/home/%I
	# Modify the next line to the absolute path for rtorrent.lock, for example
	ExecStartPre=-/bin/rm -f /home/%I/session/rtorrent.lock
	ExecStart=/usr/bin/rtorrent -o system.daemon.set=true
	ExecStop=/bin/kill -9 $MAINPID
	Restart=always
	RestartSec=3

	[Install]
	WantedBy=multi-user.target

start the services using: 

	systemctl start rtorrent@user1
	systemctl status rtorrent@user1
	systemctl enable rtorrent@user1





### configure networking

It's likely that you'll need a range of TCP and UDP ports to be forwarded.  
These commands should be entered on the main host..

	iptables -t nat -A PREROUTING -p tcp --dport 12000:13000 -j DNAT --to-destination 10.0.0.10
	iptables -t nat -A PREROUTING -p udp --dport 12000:13000 -j DNAT --to-destination 10.0.0.10
	iptables-save > /etc/iptables/iptables.rules



install rutorrent
-----------------

We'll install rutorrent as follows:

	/var/www/rutorrent/
	                  |user1  (copy of rutorrent)
	                  |user1  (copy of rutorrent)
	                  |user2 (copy of rutorrent)

### Get the files...

	mkdir -p /var/www/rutorrent
	cd /var/www/rutorrent
	git clone https://github.com/Novik/ruTorrent
	mv rutorrent user1
	chown -R http:http /var/www/rutorrent

### configure user session..

	nano /var/www/rutorrent/user1/conf/config.php
	...
	$scgi_port = 5001;              (same as in rtorrent config)
	$XMLRPCMountPoint = "/RPCuser1";  (a different one for each user)
	...

### configure nginx vhost

organizing files this way isn't necessasry but more compatible with most workflows.. 

	nano /etc/nginx/nginx.conf
	...
	http {
	    ...
	    #gzip  on;

	    include sites-enabled/*;
	}

also clear the default website and all server blocks, we won't need it anymore.  Let's create the main vhost now..  

	mkdir /etc/nginx/sites-available /etc/nginx/sites-enabled
	touch /etc/nginx/sites-available/rutorrent.conf && ln -s /etc/nginx/sites-available/rutorrent.conf /etc/nginx/sites-enabled

	nano /etc/nginx/sites-available/rutorrent.conf
	...
	server {
		listen 80;
		server_name _;

		error_log /var/log/nginx/rutorrent.error.log error;
		access_log /var/log/nginx/rutorrent.access.log;

		root /var/www/rutorrent;

		location / {
			index index.php index.html index.htm;
		}

		location ~ [^/]\.php(/|$) {
			fastcgi_split_path_info ^(.+?\.php)(/.*)$;

			try_files $uri $document_root$fastcgi_script_name =404;
			fastcgi_param HTTP_PROXY "";

			fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
			fastcgi_index index.php;
			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
			include fastcgi_params;
			}
	}


TESTING
-------

restart nginx..

	systemctl restart php-fpm && nginx -t && systemctl reload nginx

and test it!  

http://10.0.0.10/    <<-- should give a 403  
http://10.0.0.10/user1 <<-- should display something  



### Basic security

A different basic http auth should be enough. 
apache-tools contains the htpasswd command we'll need to achieve that.

	yaourt -S  apache-tools

generate/change user password

	htpasswd -c /var/www/rutorrent/user1/.htpasswd user1

edit the vhost..

    location /user1 {
        auth_basic "Restricted";
        auth_basic_user_file /var/www/rutorrent/user1/.htpasswd;
    }






Troubleshooting
---------------

### rutorrent: Webserver user can't access external program (program name).  

	nano /var/www/rutorrent/user1/conf/config.php
    ...
    $pathToExternals = array(
                "curl"   => '/usr/bin/curl',
    ...

Check missing dependencies are installed (ffmpeg, mediainfo, python,..)



### rutorrent complaing about the ./session/ not being crossable

rutorrent requires the user's homedir to be crossable by other (755). 
If not, the app won't access the ./session/ folder. So.. 

	chmod 755 /home/user1
	chmod 755 /home/user1/session


### Ratio groups explained

https://plaza.quickbox.io/t/rutorrent-ratio-groups/449






Add new user
============

This is specific to rtorrent container configuration running on pantoufle. This should be adapted to your own setup.


In the container
----------------

### Create user in the container:

	useradd -m -u 9000 -G pantouflards user2
	mkdir -p /home/user2/downloads /home/user2/session
	cp /home/user1/.rtorrent.rc /home/user2
	chown -R user2:pantouflards /home/user2
	chmod 755 /home/user2

### Configure rtorrent

	nano /home/user2/.rtorrent.rc
	...
	directory.default.set = /home/user2/downloads
	session.path.set = /home/user2/session
	network.port_range.set = 12200-12298
	dht.port.set = 12299
	scgi_port = 127.0.0.1:5002

### test rtorrent

	sudo -u user2 rtorrent


### create user ruTorrent session

	cd /var/www/rutorrent
	cp -Rpv user1 user2

edit rutorrent config file

	nano user2/conf/config.php
	...
	$scgi_port = 5002;
	$XMLRPCMountPoint = "/RPCuser2";

### edit nginx vhost

	nano /etc/nginx/sites-enabled/rutorrent.conf
	...
    location /user2 {
        auth_basic "Restricted";
        auth_basic_user_file /var/www/rutorrent/user2/.htpasswd;
    }

### user ppassword

	htpasswd /var/www/rutorrent/user2/.htpasswd user2

### restart and test

	systemctl start rtorrent@user2
	systemctl status rtorrent@user2
	systemctl enable rtorrent@user2
	systemctl restart php-fpm && nginx -t && systemctl reload nginx



On the host
-----------

### Create user account and files

	useradd -m -u 9000 -G sftp-only user2 
	passwd user2

	mkdir /home/user2/downloads
	chown -R user2:pantouflards /home/user2
	chmod 750 /home/user2
	chmod 750 /home/user2/downloads
	chmod g+s /home/user2/downloads

set the default file mask to 750

	nano /home/user2/.bashrc
	...
	umask 027

add user to the pantouflards group for file sharing

	usermod -G sftp-only,pantouflards user2
	id user2 # check!

Use the same command to remove a user from group. Just don't mention the group you want the user to be excluded.


### Update container config

	nano /var/lib/lxc/torrent/config
	...
	lxc.mount.entry = /var/home/user2/downloads home/user2/downloads none bind 0 0

and restart the container

	lxc-stop  -n torrent (use -k if doesn't stop)
	lxc-start -n torrent
	lxc-attach -n torrent --clear-env