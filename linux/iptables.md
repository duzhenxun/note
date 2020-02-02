````
route -n

arp -a

iptables -t nat -nvL

iptables -t nat -A OUTPUT -p tcp -d 127.0.0.1 --dport 10122 -j DNAT --to-destination 10.10.10.101:22
````