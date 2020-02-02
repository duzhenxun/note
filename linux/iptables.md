````
route -n

arp -a

iptables -t nat -nvL

~ iptables -t nat -A OUTPUT -p tcp -d 2.2.2.2 --dport 12722 -j DNAT --to-destination 10.10.10.101:22

iptables -L -n --line-number

````