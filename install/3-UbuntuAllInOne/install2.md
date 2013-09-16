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

netaddrって何？

- [Bug #1097667 “stack.sh should install missing dependency python-...” : Bugs : devstack](https://bugs.launchpad.net/devstack/+bug/1097667)
- [Python で IP, MAC アドレス扱うなら netaddr が超便利！ | CUBE SUGAR STORAGE](http://momijiame.tumblr.com/post/50497347245/python-ip-mac-netaddr)

```
sudo apt-get -y install python-pip
pip install netaddr
```

[ubuntu New Install - Very slow at "Reading package lists..." in apt-get](http://ubuntuforums.org/showthread.php?t=2104709)

なんでおそいの。

- [Dpkg and apt-get reading database is really slow [fixed] : Frozentux](http://www.frozentux.net/2009/08/dpkg-and-apt-get-reading-database-is-really-slow-fixed/)
- [dpkg tip | Antti-Juhani Kaijanaho](http://antti-juhani.kaijanaho.fi/newblog/archives/521)

```
sudo dpkg --clear-avail
sudo dpkg --forget-old-unavail
dpkg: warning: obsolete '--forget-old-unavail' option; unavailable packages are automatically cleaned up

sudo apt-get -y update
```

### Temp

```
git clone git://github.com/openstack/nova.git /opt/stack/nova
```

## 参考サイト

- [Single Machine Guide - DevStack](http://devstack.org/guides/single-machine.html)
