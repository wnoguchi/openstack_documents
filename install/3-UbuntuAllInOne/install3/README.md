# OpenStack Grizzly on Ubuntu(Single Node, No Quantum)

3度目の正直。  
[OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
このガイドを参考にする。

* ログ的なもおのは以下に貼り付ける。

[OpenStack Grizzly on Ubuntu logs and snippets.](https://gist.github.com/wnoguchi/b4377e74d3825610780f)

## ノードの準備

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

* システム再起動

### MySQL & RabbitMQ

```
apt-get install -y mysql-server python-mysqldb
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mysql/my.cnf
service mysql restart
apt-get install -y rabbitmq-server
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

## Keystone

```
apt-get install -y keystone
```

* Keystone動いていることを確認する。

```
service keystone status

keystone start/running, process 3580
```

* Keystoneデータベース作成。

```
mysql -u root -ppassword
CREATE DATABASE keystone;
GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass';
quit;
```

* `/etc/keystone/keystone.conf`

```
connection = mysql://keystoneUser:keystonePass@10.10.100.51/keystone
```

* Identity Serviceを再起動。DB同期。

```
service keystone restart
keystone-manage db_sync
```

```
git clone https://gist.github.com/b4377e74d3825610780f.git
cd b4377e74d3825610780f/

#Modify the HOST_IP and HOST_IP_EXT variables before executing the scripts
sh keystone_basic.sh
sh keystone_endpoints_basic.sh
```

* のちのち読み込むクレデンシャルファイル `creds` を作成する

```
# creds
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin_pass
export OS_AUTH_URL="http://192.168.1.200:5000/v2.0/"
```

* ロード

```
source creds
```

* Keystoneのテスト

```
root@stack01:~/b4377e74d3825610780f# keystone user-list
+----------------------------------+---------+---------+--------------------+
|                id                |   name  | enabled |       email        |
+----------------------------------+---------+---------+--------------------+
| 652e949b65d144aa9e703017835b4d2c |  admin  |   True  |  admin@domain.com  |
| ecec2eb6154f4eaeaddf6258c15cd63a |  cinder |   True  | cinder@domain.com  |
| f30108077fc749c7a92ecbb6e3a92bac |  glance |   True  | glance@domain.com  |
| ba61c5b2e22f4768801b6255a2db7b30 |   nova  |   True  |  nova@domain.com   |
| 4590f63aa4574afa9d7e3c70f8abf12f | quantum |   True  | quantum@domain.com |
+----------------------------------+---------+---------+--------------------+
```

## Glance



## 参考サイト

- [mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide)
- [OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
