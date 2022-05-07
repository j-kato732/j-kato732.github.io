# CentOS7以降のファイアウォールはfirewalld

CentOS6までは高機能なパケットフィルタとしてiptablesを使用していたが、
CentOS7からは新しく「firewalld」が実装された。

以降デフォルトでfirewalldが使われているので、
firewalldを使ったfirewallの操作についてまとめていく。

# ゾーンという概念

firewalldにはゾーンという概念が存在する。

ゾーンとは、インターフェースをセクション単位ではなく性質単位で設定するための方式である。
簡単にいうと「通信の設定を簡単に区分けするための区分け」である。
具体的には、firewalldのインターフェース（接触面）をグループ化し、性質ごとのアクセス制御が設定できる。
例えば、WebサーバとのインターフェースはゾーンA、LANとのインターフェースはゾーンBを適用するなどが可能。

一度作成したゾーンは複数のインターフェースに割り当てられるため、同じ性質のインタフェースを個別に設定しなくて済む。
例えば、Webサーバとのインターフェースが複数ある場合、それら全てにゾーンAを一括適用が可能。何度も同じ設定をする必要がないので、管理が簡単。

firewalldには、新規の状態で９種類のゾーンが準備されており、それらはNIC（ネットワークアダプタ）に適用することで機能する。
ゾーンを自作して増やすことも可能。

ゾーンを利用した区分けの方式をゾーンベースモデルと呼ぶ。

# 9種類のゾーン
| ゾーン | 説明 |
|---|---|
| drop | 全てのパケットを破棄する<br>内部から外部へのパケットは許可されるが、返信されてきたパケットも破棄するので実質的に通信不可となる。 |
| block | 外部からのパケットは基本的に破棄される。<br>内部からの通信パケットの返信は許可される。 |
| public | デフォルトで「ssh」と「dhcpv6-client」のみ許可されている |
| external | デフォルトで「sshのみ許可される。ipマスカレードが有効になる |
| dmz | デフォルトで「ssh」のみ許可されている |
| work | デフォルトで「dhcpv6-client」と「ipp-client」、「ssh」が許可される |
| home | デフォルトで「dhcpv6-client」と「ipp-client」と「mdns」と「samba-client」、「ssh」が許可される |
| internal | デフォルトで「dhcpv6-client」と「ipp-client」と「mdns」と「samba-client」、「ssh」が許可される |
| trusted | 全てのパケットが許可される |

※
dhcpv6（Dynamic Host Configuration Protocol for IP version 6）: コンピュータがネットワーク接続する際に必要な情報を自動的に割り当てる通信プロトコルのIPv6版  
ipp（Internet Printing Protocol）: TCP/IPネットワークを利用して、遠隔地にあるプリンタとコンピュータの間で印刷データなどのやりとりを行うためのプロトコル  
mdns（multicast DNS）: ローカルネットワーク内でホスト名からIPアドレスを割り出すために用いられるプロトコル  
samba: LinuxやBSDなどのUNIX系OSをWindowsNetwarkに参加されるためのソフト（ftpのようにWinサーバとやりとりが可能になる）
ipマスカレード: 2つのネットワーク間（WAN/LAN）でIPアドレスやTCP/UDPのポート番号を変換して、限られたIPアドレス資源を有効に使うための技術

# firewalldを操作する
## サービスの起動確認
```
$ systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2022-03-13 07:36:12 UTC; 5 days ago
     Docs: man:firewalld(1)
 Main PID: 787 (firewalld)
    Tasks: 2 (limit: 23696)
   Memory: 36.9M
   CGroup: /system.slice/firewalld.service
           └─787 /usr/libexec/platform-python -s /usr/sbin/firewalld --nofork --nopid

 3月 13 07:36:09 localhost.localdomain systemd[1]: Starting firewalld - dynamic firewall daemon...
 3月 13 07:36:12 localhost.localdomain systemd[1]: Started firewalld - dynamic firewall daemon.
 3月 13 07:36:13 localhost.localdomain firewalld[787]: WARNING: AllowZoneDrifting is enabled. This is considered an insecure configuration option. It will be removed in a future release. Please consider >
```

or 

```
$ firewall-cmd --state
running
```

## 起動されていなかったら
```
$ systemctl enable firewalld
$ systemctl start firewalld
```

## デフォルトの設定を確認
```
$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
eth0にpublicゾーンが指定されている

## ゾーンの設定を確認する
対象のゾーンの確認
```
$ firewall-cmd --zone=public --list-all
```

全てのゾーンの設定を確認する
```
$ firewall-cmd --list-all-zones
``` 

## 設定の変更
デフォルトのゾーンの変更
```
$ firewall-cmd --set-default-zonez=block
```

インターフェースのゾーンの変更
```
$ firewall-cmd --change-interface=enp0s3
```

恒久的にインターフェースのゾーンを変更する場合
```
$ vim /etc/sysconfig/network-scripts/ifcfg-enp0s3
```
Zoneの記述ないなeth０のやつみてみたけど

## サービスの確認

利用可能サービスを確認する
`/usr/lib/firewalld/services`内のxmlファイルでも確認できる
```
$ firewall-cmd --get-services
RH-Satellite-6 RH-Satellite-6-capsule amanda-client amanda-k5-client amqp amqps apcupsd audit bacula bacula-client bb bgp bitcoin bitcoin-rpc bitcoin-testnet bitcoin-testnet-rpc bittorrent-lsd ceph ceph-mon cfengine cockpit collectd condor-collector ctdb dhcp dhcpv6 dhcpv6-client distcc dns dns-over-tls docker-registry docker-swarm dropbox-lansync elasticsearch etcd-client etcd-server finger foreman foreman-proxy freeipa-4 freeipa-ldap freeipa-ldaps freeipa-replication freeipa-trust ftp galera ganglia-client ganglia-master git grafana gre high-availability http https imap imaps ipp ipp-client ipsec irc ircs iscsi-target isns jenkins kadmin kdeconnect kerberos kibana klogin kpasswd kprop kshell kube-apiserver ldap ldaps libvirt libvirt-tls lightning-network llmnr managesieve matrix mdns memcache minidlna mongodb mosh mountd mqtt mqtt-tls ms-wbt mssql murmur mysql nbd nfs nfs3 nmea-0183 nrpe ntp nut openvpn ovirt-imageio ovirt-storageconsole ovirt-vmconsole plex pmcd pmproxy pmwebapi pmwebapis pop3 pop3s postgresql privoxy prometheus proxy-dhcp ptp pulseaudio puppetmaster quassel radius rdp redis redis-sentinel rpc-bind rquotad rsh rsyncd rtsp salt-master samba samba-client samba-dc sane sip sips slp smtp smtp-submission smtps snmp snmptrap spideroak-lansync spotify-sync squid ssdp ssh steam-streaming svdrp svn syncthing syncthing-gui synergy syslog syslog-tls telnet tentacle tftp tftp-client tile38 tinc tor-socks transmission-client upnp-client vdsm vnc-server wbem-http wbem-https wsman wsmans xdmcp xmpp-bosh xmpp-client xmpp-local xmpp-server zabbix-agent zabbix-server
```

icmpに使わない機能を追加する
`/usr/lib/firewalld/icmptypes`内のxmlファイルでも確認できる
```
$ firewall-cmd --get-icmptypes
address-unreachable bad-header beyond-scope communication-prohibited destination-unreachable echo-reply echo-request failed-policy fragmentation-needed host-precedence-violation host-prohibited host-redirect host-unknown host-unreachable ip-header-bad neighbour-advertisement neighbour-solicitation network-prohibited network-redirect network-unknown network-unreachable no-route packet-too-big parameter-problem port-unreachable precedence-cutoff protocol-unreachable redirect reject-route required-option-missing router-advertisement router-solicitation source-quench source-route-failed time-exceeded timestamp-reply timestamp-request tos-host-redirect tos-host-unreachable tos-network-redirect tos-network-unreachable ttl-zero-during-reassembly ttl-zero-during-transit unknown-header-type unknown-option
```

## サービスを追加する
サービスはport番号の別名
sshは22、httpなら80、dhcpv6-clientなら546が許可される
```
$ firewall-cmd --permanent --add-service=https
```

```
# firewall-cmd --permanent --add-service=https
success
# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
# firewall-cmd --reload
success
# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: cockpit dhcpv6-client https ssh
  ports:
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
サービスの削除
```
$ firewall-cmd --remove-service=http
```

# ポートの開放
8080番を解放する場合
```
$ firewall-cmd --add-port=8080/tcp
```

解放されたか確認してみる  
ports:に追加されていればOK
```
$ firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0
  sources:
  services: http https ssh
  ports: 8080/tcp
  protocols:
  forward: no
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```


上記のコマンドの場合、firewalldが再起動した場合設定が失われるため  
恒久的に設定を変更する場合は`--permanent`オプションを追加する
```
$ firewall-cmd --add-port=8080/tcp --permanent
```
