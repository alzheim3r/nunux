fail2ban
========

unban ip
--------

	iptables -S 
	# look for f2b-sshd
	fail2ban-client set sshd unbanip 37.165.28.214
	fail2ban-client set php-404 unbanip 37.165.28.214


whitelist ip
------------

	nano /etc/fail2ban/jail.local
	
	[DEFAULT]
	ignoreip  = 127.0.0.1/8 10.0.0.0/8 37.165.230.232/32


logs
----

	lnav /var/log/fail2ban.log


custom filter
-------------

dir: /etc/fail2ban/filter.d/  

Try to match this kind of log:

	46.246.123.79 - - [30/Jun/2019:12:21:35 +0200] "GET /shop/index.php/admin/ HTTP/1.1" 404 8257 "-" "Mozilla/5.0 (Windows; U; Windows NT 2.0) Gecko/20091201 Firefox/3.5.6 GTB5"

	nano custom_php-404.conf

	[Definition]
	failregex=<HOST> - - \[.*\] \"GET \/.*\.php.*HTTP\/\d+\.\d+\" 404.*
	ignoreregex=

* ```\d+``` = any digit
* ```.*```  = any one or multiple characters
* Escape everything!


### Test the filter

Check if the regex works:

	fail2ban-regex --print-all-matched /var/log/website.access.log /etc/fail2ban/filter.d/custom_php-404.conf

For multiple logfiles:

	for f in /var/log/nginx/*.access.log ; do fail2ban-regex -v --print-all-matched $f /etc/fail2ban/filter.d/custom_php-404.conf ; done

### Setup the jail

dir: /etc/fail2ban/jail.d/  

	nano php-404.local

	[php-404]
	enabled   = true
	filter    = custom_php-404
	logpath = /var/log/nginx/pantoufle.ml.access.log
	          /var/log/nginx/5.39.90.190.access.log
	          /var/log/nginx/digitalsook.net.access.log
	          /var/log/nginx/ns3277635.ip-5-39-90.eu.access.log
	          /var/log/nginx/porn.pantoufle.ml.access.log
	          /var/log/nginx/sonic.access.log
	          /var/log/nginx/isso.access.log
	          /var/log/nginx/seahub.access.log
	          /var/log/nginx/ru.access.log
	maxretry  = 1
	findtime  = 1800
	bantime   = 3600
	port      = http,https

### Test and restart

	fail2ban-client reload && systemctl restart fail2ban

Check fail2ban.log !


fail2ban commands
-----------------

	fail2ban-client status
	fail2ban-client status php-404
	fail2ban-client reload
	systemctl restart fail2ban
	iptables -S
