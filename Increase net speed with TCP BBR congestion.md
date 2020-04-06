Increase Linux Internet speed with TCP BBR congestion control
=============================================================

Source
------

* https://bbs.archlinux.org/viewtopic.php?id=223879
* https://www.cyberciti.biz/cloud-computing/increase-your-linux-server-internet-speed-with-tcp-bbr-congestion-control/

commands
--------

    modprobe tcp_bbr
    sysctl net.core.default_qdisc=fq
    sysctl net.ipv4.tcp_congestion_control=bbr

check
-----

    sysctl net.ipv4.tcp_available_congestion_control
    sysctl net.ipv4.tcp_congestion_control
    sysctl net.core.default_qdisc

save config
-----------

    echo "tcp_bbr" > /etc/modules-load.d/modules.conf
    echo "net.core.default_qdisc=fq" > /etc/sysctl.d/bbr.conf
    echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.d/bbr.conf