Upnp port forwarding request
=============================

No need to access the router's webUI setup port forwarding to your linux PC! However the router needs to have Upnp enabled for this to work.

# Nat port 12001

    upnpc -a 192.168.1.17 12001 12001 TCP
    upnpc -a 192.168.1.17 12001 12001 UDP

Where *192.168.1.17* is the IP addres given by the router.


# Remove torrent port 

    upnpc -d 12001 TCP
    upnpc -d 12001 UDP
