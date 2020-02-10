Email forwarding with postfix
=============================

Setup MX Record
---------------------
Goto  the domain.tld DNS records
add ```MX 123.123.123.123, priority 10```


Install Postfix
-----------------

    apt-get install postfix
    service stop postfix
    service stop sendmail
    killall sendmail
    # make sure sendmail is not running
    psa|grep sendmail  


Config
--------
    nano /etc/postfix/main.cf

Add: 

    virtual_maps = hash:/mail/addresses

Note: Use valid ssl certs (no selfsigned) or mails could be rejected (spam).

    mkdir /mail/
    nano /mail/addresses

Add:

    domain.tld                DOMAIN
    postmaster@domain.tld     admin@digitalsook.net
    stacksetup.com            DOMAIN
    test@stacksetup.com       test@example.com
    jack@stacksetup.com       jack@example.com

Generate ddresses.db

    postmap /mail/addresses

Restart

    service postfix restart
    check syslog if errors


Spam protection
===============

Access Restriction
------------------
```
nano /etc/postfix/main.cf
```
```
# HELO restrictions:
smtpd_delay_reject = yes
smtpd_helo_required = yes
smtpd_helo_restrictions =
    permit_mynetworks,
    reject_non_fqdn_helo_hostname,
    reject_invalid_helo_hostname,
    permit

# Sender restrictions:
smtpd_sender_restrictions =
    permit_mynetworks,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain,
    permit
```

Note: Sender's domain should have a MX.
Source: https://wiki.centos.org/HowTos/postfix_restrictions


Send mail to white-listed domains only
--------------------------------------
```
nano /etc/postfix/main.cf
```
```
smtpd_recipient_restrictions = 
  check_recipient_access hash:/mail/recipient_domains, 
  reject
```
```
nano /mail/recipient_domains 
```
```
mycompany.com     OK
anotherdomain.com OK
```

generate .db file & restart

    postmap /mail/recipient_domains
    service postfix restart



Queue Management
----------------
Remove all mail from the queue:

    postsuper -d ALL

See mail queue:

    mailq




oneline shortcut
----------------

    postmap /mail/addresses && postmap /mail/client_checks && postmap /mail/recipient_domains && service postfix restart



Source
------
- http://stacksetup.com/Email/PostfixSendForward
- http://thinlight.org/2012/03/10/- postfix-only-allow-whitelisted-recipient-domains/
- https://www.cyberciti.biz/tips/howto-postfix-flush-mail-queue.html
- http://www.linuxlasse.net/linux/howtos/Blacklist_and_Whitelist_with_Postfix
- https://wiki.centos.org/HowTos/postfix_restrictions

