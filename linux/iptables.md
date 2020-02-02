````
route -n

arp -a

iptables -t nat -nvL

添加
~ iptables -t nat -A OUTPUT -p tcp -d 2.2.2.2 --dport 12722 -j DNAT --to-destination 10.10.10.101:22

删除 iptables -t nat -D OUTPUT 2

iptables -L -n --line-number

安装dig
yum install bind-utils
dig @114.114.114.114 baidu.com

````
````
`