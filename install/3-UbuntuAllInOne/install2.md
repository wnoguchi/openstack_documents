# UbuntuにAllInOne構成でOpenStackをインストール(Grizzly) 2

以前は失敗したので仕切りなおし。  
Preseedで同じ構成の物理サーバー何回も作りなおすことできるようになったから気兼ねなく試せそう。

## インストール

### 環境構成

- eth0: 192.168.1.200/24
- eth1: （線だけはつながってる）
- GW: 192.168.1.1

### パッケージインストール

```
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get -y install git
```

### ユーザーstackを追加してパス無しsudoできるようにする

```
sudo adduser stack
sudo visudo
```

```
stack ALL=(ALL) NOPASSWD: ALL
```

```
logout
```

以降はユーザーstackでログインして作業。

### DevStackのチェックアウト

```
git clone git://github.com/openstack-dev/devstack.git
cd devstack
git checkout stable/grizzly
```

### localrc

```
cp samples/localrc .
```

### 実験しやすくする（バージョン管理する）

```
cd ..
git clone https://gist.github.com/c31e7e1933d34dcf979e.git
mv c31e7e1933d34dcf979e/ config
ln -s /home/stack/config/localrc /home/stack/devstack/localrc
```

```
pending.
```

### 実行

```
./stack.sh
Traceback (most recent call last):
  File "<string>", line 2, in <module>
ImportError: No module named netaddr
Traceback (most recent call last):
  File "<string>", line 2, in <module>
ImportError: No module named netaddr
Using mysql database backend

################################################################################
ENTER A SERVICE_TOKEN TO USE FOR THE SERVICE ADMIN TOKEN.
################################################################################
This value will be written to your localrc file so you don't have to enter it 
again.  Use only alphanumeric characters.
If you leave this blank, a random default value will be used.
Enter a password now:

(snip)

2013-09-16 23:23:13 E: Could not get lock /var/lib/dpkg/lock - open (11: Resource temporarily unavailable)
2013-09-16 23:23:13 E: Unable to lock the administration directory (/var/lib/dpkg/), is another process using it?
2013-09-16 23:23:13 +++ failed
2013-09-16 23:23:13 +++ local r=100
2013-09-16 23:23:13 ++++ jobs -p
2013-09-16 23:23:13 +++ kill
2013-09-16 23:23:13 +++ set +o xtrace
2013-09-16 23:23:13 stack.sh failed: full log in /opt/stack/logs/stack.sh.log.2013-09-16-232302
```

**netaddrって何？**

- [Bug #1097667 “stack.sh should install missing dependency python-...” : Bugs : devstack](https://bugs.launchpad.net/devstack/+bug/1097667)
- [Python で IP, MAC アドレス扱うなら netaddr が超便利！ | CUBE SUGAR STORAGE](http://momijiame.tumblr.com/post/50497347245/python-ip-mac-netaddr)

```
sudo apt-get -y install python-pip
sudo pip install netaddr
```

```
./stack.sh
ENTER EXTRA なんちゃら
（ENTERをキーイン ）
```

以下のキーワードで検索

```
checking http://169.254.169.254/ devstack all-in-one
```

プロミスキャスモード？

- [OpenStack Folsom Quantum Devstack Installation Tutorial](http://networkstatic.net/openstack-folsom-quantum-devstack-installation-tutorial/)
- [Ubuntuのブリッジ接続設定 - ぽたうんのメモ帳](http://d.hatena.ne.jp/portown/20110211/1297354625)
- [知らないと地味にハマるOpen stackインストール時の注意点](http://www.slideshare.net/d-shen/open-stack)

```
http://169.254.169.254/2009-04-04/instance-id
```

- [Devstack Grizzly networking help needed | OpenStack | Dev](http://www.gossamer-threads.com/lists/openstack/dev/29795)

```
auto eth0
iface eth0 inet static 
      address 192.168.1.200 
      netmask 255.255.255.0 
      network 192.168.1.0 
      broadcast 192.168.1.255 
      gateway 192.168.1.1 
      dns-nameservers 8.8.8.8 8.8.4.4 

#Secondary network not connected to anything(cable not plugged in) 
auto eth1 
      iface eth1 inet manual 
      up ifconfig $IFACE 0.0.0.0 up 
      up ip link set $IFACE promisc on 
      down ip link set $IFACE promisc off 
      down ifconfig $IFACE down 

```

```
# While ``stack.sh`` is happy to run without ``localrc``, devlife is better when
# there are a few minimal variables set:
FLOATING_RANGE=192.168.1.224/27
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=eth0

# If the ``*_PASSWORD`` variables are not set here you will be prompted to enter
# values for them by ``stack.sh`` and they will be added to ``localrc``.
ADMIN_PASSWORD=nomoresecrete
MYSQL_PASSWORD=stackdb
RABBIT_PASSWORD=stackqueue
SERVICE_PASSWORD=$ADMIN_PASSWORD
```

### Temp

```
git clone git://github.com/openstack/nova.git /opt/stack/nova
```

ムリポ。。。ワカランぽ。。。  
挫折。。。

## 参考サイト

- [Single Machine Guide - DevStack](http://devstack.org/guides/single-machine.html)
