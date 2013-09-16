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

### Temp

```
git clone git://github.com/openstack/nova.git /opt/stack/nova
```

## 参考サイト

- [Single Machine Guide - DevStack](http://devstack.org/guides/single-machine.html)
