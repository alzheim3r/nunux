Upnp port forwarding request
=============================

Only works if Upnp is enabled on the router.

# Nat port 12001

    upnpc -a 192.168.1.17 12001 12001 TCP
    upnpc -a 192.168.1.17 12001 12001 UDP

Where *192.168.1.17* is the IP addres given by the router.


# Remove torrent port 

    upnpc -d 12001 TCP
    upnpc -d 12001 UDP
