````
route -n

arp -a

iptables -t nat -nvL

iptables -t nat -A OUTPUT -p tcp -d 10.10.10.101 --dport 10122 -j DNAT 
````