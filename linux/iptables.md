
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
