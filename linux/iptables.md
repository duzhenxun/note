
route -n

arp -a

### 查看规则
iptables -t nat -nvL

```language
Chain PREROUTING (policy ACCEPT 311 packets, 19028 bytes)
 pkts bytes target     prot opt in     out     source               destination         
  68M 3825M DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

Chain INPUT (policy ACCEPT 87 packets, 3656 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 32 packets, 2215 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DOCKER     all  --  *      *       0.0.0.0/0           !127.0.0.0/8          ADDRTYPE match dst-type LOCAL

Chain POSTROUTING (policy ACCEPT 28 packets, 1951 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  *      !docker0  172.18.0.0/16        0.0.0.0/0           
9588K  572M MASQUERADE  all  --  *      !br-dfc0e0a09d43  10.10.0.0/16         0.0.0.0/0           
    0     0 MASQUERADE  tcp  --  *      *       10.10.10.101         10.10.10.101         tcp dpt:22

Chain DOCKER (2 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  docker0 *       0.0.0.0/0            0.0.0.0/0           
21358 1281K RETURN     all  --  br-dfc0e0a09d43 *       0.0.0.0/0            0.0.0.0/0           
   64  3300 DNAT       tcp  --  !br-dfc0e0a09d43 *       0.0.0.0/0            0.0.0.0/0            tcp dpt:32769 to:10.10.10.101:22
➜ 
```


添加
~ iptables -t nat -A OUTPUT -p tcp -d 2.2.2.2 --dport 12722 -j DNAT --to-destination 10.10.10.101:22

删除 iptables -t nat -D OUTPUT 2

iptables -L -n --line-number

#### DNS查看
安装dig
yum install bind-utils
dig @114.114.114.114 baidu.com




安装dig
yum install bind-utils
dig @114.114.114.114 baidu.com
````
 ~ dig @114.114.114.114 baidu.com

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> @114.114.114.114 baidu.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 23433
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;baidu.com.                     IN      A

;; ANSWER SECTION:
baidu.com.              524     IN      A       220.181.38.148
baidu.com.              524     IN      A       39.156.69.79

;; Query time: 29 msec
;; SERVER: 114.114.114.114#53(114.114.114.114)
;; WHEN: 一 2月 03 01:41:02 CST 2020
;; MSG SIZE  rcvd: 70

````

丢包
iptables -t filter -A OUTPUT -p udp -d 114.114.114.114 -j DROP
dig @114.114.114.114 baidu.com
```shell
; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> @114.114.114.114 baidu.com
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

查看
iptables -t filter -nvL
```language
Chain INPUT (policy ACCEPT 2899 packets, 177K bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain FORWARD (policy DROP 44 packets, 2962 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 2562 packets, 529K bytes)
 pkts bytes target     prot opt in     out     source               destination         
    4   264 DROP       udp  --  *      *       0.0.0.0/0            114.114.114.114     

Chain DOCKER (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER-ISOLATION-STAGE-2 (0 references)
 pkts bytes target     prot opt in     out     source               destination         

Chain DOCKER-USER (0 references)
 pkts bytes target     prot opt in     out     source               destination 
```
删除丢包规则
iptables -t filter -D OUTPUT 1


```language
 cat /proc/net/nf_conntrack
ipv4     2 tcp      6 299 ESTABLISHED src=27.189.141.216 dst=172.17.2.236 sport=23213 dport=22 src=172.17.2.236 dst=27.189.141.216 sport=22 dport=23213 [ASSURED] mark=0 zone=0 use=2
ipv4     2 icmp     1 29 src=27.189.141.216 dst=172.17.2.236 type=8 code=0 id=23222 src=172.17.2.236 dst=27.189.141.216 type=0 code=0 id=23222 mark=0 zone=0 use=2
ipv4     2 tcp      6 431999 ESTABLISHED src=172.17.2.236 dst=100.100.30.25 sport=43640 dport=80 src=100.100.30.25 dst=172.17.2.236 sport=80 dport=43640 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 23185 ESTABLISHED src=190.237.143.89 dst=172.17.2.236 sport=50025 dport=4001 src=172.17.2.236 dst=190.237.143.89 sport=4001 dport=50025 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 41 TIME_WAIT src=10.10.10.1 dst=10.10.10.101 sport=53506 dport=443 src=10.10.10.101 dst=10.10.10.1 sport=443 dport=53506 [ASSURED] mark=0 zone=0 use=2
ipv4     2 tcp      6 55 SYN_RECV src=51.81.119.9 dst=172.17.2.236 sport=25424 dport=80 src=172.17.2.236 dst=51.81.119.9 sport=80 dport=25424 mark=0 zone=0 use=2
ipv4     2 tcp      6 41 TIME_WAIT src=66.249.71.80 dst=172.17.2.236 sport=65480 dport=443 src=172.17.2.236 dst=66.249.71.80 sport=443 dport=65480 [ASSURED] mark=0 zone=0 use=2

```

