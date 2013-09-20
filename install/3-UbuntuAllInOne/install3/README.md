# OpenStack Grizzly on Ubuntu(Single Node, No Quantum)

3度目の正直。  
[OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
このガイドを参考にする。

* ログ的なもおのは以下に貼り付ける。

[OpenStack Grizzly on Ubuntu logs and snippets.](https://gist.github.com/wnoguchi/b4377e74d3825610780f)

## メモ

```
apt-get -y install vim tmux curl git
```

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

* インストール

```
apt-get install -y glance
```

* Glanceサービスが動いていることを確認する

```
service glance-api status
service glance-registry status
```

* MySQLデータベースを構築する

```
mysql -u root -ppassword
CREATE DATABASE glance;
GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass';
quit;
```

* 以下は主にglance apiの認証関係の設定です。

* `/etc/glance/glance-api-paste.ini` を変更する

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
delay_auth_decision = true

# ↓

[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
delay_auth_decision = true
auth_host = 10.10.100.51
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = service_pass
```

* `/etc/glance/glance-registry-paste.ini`

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

# ↓

[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.10.100.51
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = service_pass
```

* `/etc/glance/glance-api.conf`

```
sql_connection = sqlite:////var/lib/glance/glance.sqlite
# ↓
sql_connection = mysql://glanceUser:glancePass@10.10.100.51/glance

# (snip)

[paste_deploy]
flavor = keystone
```

* glance-api と glance-registry サービスの再起動する

```
service glance-api restart; service glance-registry restart

glance-api stop/waiting
glance-api start/running, process 6399
glance-registry stop/waiting
glance-registry start/running, process 6404
```

* Glanceデータベース同期

```
glance-manage db_sync
```

* さらに再起動

```
service glance-registry restart; service glance-api restart
```

* Glanceが正常に稼働しているかテストのため cirros のイメージをネット上から取得して登録する

```
glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

Error communicating with http://192.168.100.51:9292 [Errno 113] No route to host
```

エラーになった。  
しかたがないのでwgetでダウンロードをかましてレジストリする。

```
wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location cirros-0.3.0-x86_64-disk.img

Error communicating with http://192.168.100.51:9292 [Errno 110] Connection timed out
```

またエラー。なんなの。。。

* `/etc/glance/glance-api-paste.ini`

```
# Address to bind the API server
bind_host = 0.0.0.0

# ↓

# Address to bind the API server
bind_host = 192.168.1.200
```

* さらに再起動

```
glance-manage db_sync
service glance-registry restart; service glance-api restart
glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
```

`sh keystone_endpoints_basic.sh` を見たら `EXT_HOST_IP=192.168.100.51` がハードコードされてた。。。。。。  
ぐぬぬ。。。

```
EXT_HOST_IP=192.168.100.51
# ↓
EXT_HOST_IP=192.168.1.200
```

もう一回run。

```
sh keystone_endpoints_basic.sh
```

```
```

↓以下、まだ出来てない。


* アップされたイメージがリストされていることを確認する

```
glance image-list
```

## 参考サイト

- [mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide)
- [OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
