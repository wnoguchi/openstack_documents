# OpenStack Grizzly on Ubuntu(Single Node, No Quantum)

3度目の正直。  
[OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
このガイドを参考にする。

## インストール

### 事前準備

* rootになる

これ以降ずっとroot。

```
sudo su -
```

* Grizzyのリポジトリを追加する

```
apt-get -y install ubuntu-cloud-keyring python-software-properties software-properties-common python-keyring
echo deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/grizzly main >> /etc/apt/sources.list.d/grizzly.list
```

* システムを最新の状態にもっていく

```
apt-get -y update
apt-get -y upgrade
apt-get -y dist-upgrade
```

### ネットワーク設定

ガイドでは逆になってたので適宜読み替え。  
この適宜読み替えが仇とならなければいいが・・・。

```
#For Exposing OpenStack API over the internet
auto eth0
iface eth0 inet static
address 192.168.1.200
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 8.8.8.8

#Not internet connected(used for OpenStack management)
auto eth1
iface eth1 inet static
address 10.10.100.51
netmask 255.255.255.0
```

* ネットワーク再起動

- [Ubuntu日本語フォーラム / service /etc/init.d/networking restart がunrecognized servce ...](https://forums.ubuntulinux.jp/viewtopic.php?id=9522)

```
root@stack01:~# /etc/init.d/networking restart
 * Running /etc/init.d/networking restart is deprecated because it may not enable again some interfaces
 * Reconfiguring network interfaces...
 * Disconnecting iSCSI targets
   ...done.
 * Stopping iSCSI initiator service
   ...done.
RTNETLINK answers: No such process
 * Disconnecting iSCSI targets
   ...done.
 * Stopping iSCSI initiator service
   ...done.
RTNETLINK answers: File exists
Failed to bring up eth0.
 * Starting iSCSI initiator service iscsid
   ...done.
 * Setting up iSCSI targets
   ...done.
ssh stop/waiting
ssh start/running, process 7464
   ...done.
```

なにこれ。

- [玄柴/ホスト名とIPアドレスの設定 - PelicanWiki](http://pelican.ddo.jp/fukurou/mediawiki/index.php/%E7%8E%84%E6%9F%B4/%E3%83%9B%E3%82%B9%E3%83%88%E5%90%8D%E3%81%A8IP%E3%82%A2%E3%83%89%E3%83%AC%E3%82%B9%E3%81%AE%E8%A8%AD%E5%AE%9A)

```
oot@stack01:~# /etc/init.d/networking stop && /etc/init.d/networking start
 * Deconfiguring network interfaces...
 * Disconnecting iSCSI targets
   ...done.
 * Stopping iSCSI initiator service
   ...done.
   ...done.
Rather than invoking init scripts through /etc/init.d, use the service(8)
utility, e.g. service networking start

Since the script you are attempting to invoke has been converted to an
Upstart job, you may also use the start(8) utility, e.g. start networking
networking stop/waiting

```

あとで勉強しよう。とりあえず手っ取り早くリブート。

```
sudo reboot
```

```
root@stack01:~# ifconfig
eth0      Link encap:Ethernet  HWaddr 00:22:4d:69:07:7e
          inet addr:192.168.1.200  Bcast:192.168.1.255  Mask:255.255.255.0
          inet6 addr: fe80::222:4dff:fe69:77e/64 Scope:Link
          inet6 addr: 2001:c90:8780:734a:bc4c:eea4:42f2:c44e/64 Scope:Global
          inet6 addr: 2001:c90:8780:734a:222:4dff:fe69:77e/64 Scope:Global
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:188 errors:0 dropped:28 overruns:0 frame:0
          TX packets:127 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:21704 (21.7 KB)  TX bytes:19449 (19.4 KB)
          Interrupt:20 Memory:efa00000-efa20000

eth1      Link encap:Ethernet  HWaddr 00:22:4d:69:07:7d
          inet addr:10.10.100.51  Bcast:10.10.100.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
          Interrupt:18 Memory:ef900000-ef920000

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:130 errors:0 dropped:0 overruns:0 frame:0
          TX packets:130 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:8406 (8.4 KB)  TX bytes:8406 (8.4 KB)

virbr0    Link encap:Ethernet  HWaddr de:09:e4:ae:2c:99
          inet addr:192.168.122.1  Bcast:192.168.122.255  Mask:255.255.255.0
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

### MySQL & RabbitMQ

```
apt-get install -y mysql-server python-mysqldb
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
apt-get install -y rabbitmq-server
```

* NTSサービスのインストール

```
apt-get install -y ntp
```

### その他

```
apt-get install -y vlan bridge-utils
```

```
sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
```

* 永続化する

```
sysctl net.ipv4.ip_forward=1
```

## 参考サイト

- [mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide)
- [OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
