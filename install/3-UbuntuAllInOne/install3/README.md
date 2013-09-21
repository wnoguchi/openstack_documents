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

* 以下は主にglance apiの認証トークンまわりの設定です。

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
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | None                                 |
| container_format | bare                                 |
| created_at       | 2013-09-21T01:16:13.056555           |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | 07350478-6eab-4376-8360-b1ceda60ba0f |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | myFirstImage                         |
| owner            | None                                 |
| protected        | False                                |
| size             | 9761280                              |
| status           | active                               |
| updated_at       | 2013-09-21T01:16:13.398988           |
+------------------+--------------------------------------+
```

### トラブルシューティング

```
glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img

Error communicating with http://192.168.100.51:9292 [Errno 113] No route to host
```

`keystone_endpoints_basic.sh` を見たら `EXT_HOST_IP=192.168.100.51` がハードコードされてた。。。  
ぐぬぬ。。。

```
EXT_HOST_IP=192.168.100.51
# ↓
EXT_HOST_IP=192.168.1.200
```

として

```
sh keystone_endpoints_basic.sh
```

解決。

* アップされたイメージがリストされていることを確認する

```
glance image-list

+--------------------------------------+--------------+-------------+------------------+---------+--------+
| ID                                   | Name         | Disk Format | Container Format | Size    | Status |
+--------------------------------------+--------------+-------------+------------------+---------+--------+
| 07350478-6eab-4376-8360-b1ceda60ba0f | myFirstImage | qcow2       | bare             | 9761280 | active |
+--------------------------------------+--------------+-------------+------------------+---------+--------+
```

## Quantum

* Quantum関係のパッケージをインストール

```
apt-get install -y quantum-server quantum-plugin-linuxbridge quantum-plugin-linuxbridge-agent dnsmasq quantum-dhcp-agent quantum-l3-agent
```

* MySQLデータベースを構築する

```
mysql -u root -ppassword
CREATE DATABASE quantum;
GRANT ALL ON quantum.* TO 'quantumUser'@'%' IDENTIFIED BY 'quantumPass';
quit;
```

* すべてのQuantumコンポーネントが動作していることを確認する

```
cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i status; done

quantum-dhcp-agent start/running, process 12325
quantum-l3-agent start/running, process 12364
quantum-metadata-agent start/running, process 12287
quantum-plugin-linuxbridge-agent start/running, process 12405
quantum-server start/running, process 12448
```

* `/etc/quantum/quantum.conf` を編集

```
core_plugin = quantum.plugins.openvswitch.ovs_quantum_plugin.OVSQuantumPluginV2

# ↓

core_plugin = quantum.plugins.linuxbridge.lb_quantum_plugin.LinuxBridgePluginV2
```

* `/etc/quantum/api-paste.ini` を編集

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
admin_user = quantum
admin_password = service_pass
```

* LinuxBridgeプラグインの設定ファイル `/etc/quantum/plugins/linuxbridge/linuxbridge_conf.ini` を編集する

```
# under [DATABASE] section
sql_connection = mysql://quantumUser:quantumPass@10.10.100.51/quantum
# under [LINUX_BRIDGE] section
physical_interface_mappings = physnet1:eth0
# under [VLANS] section
tenant_network_type = vlan
network_vlan_ranges = physnet1:1000:2999
```

* L3 agent設定ファイル `/etc/quantum/l3_agent.ini` を編集する

```
interface_driver = quantum.agent.linux.interface.OVSInterfaceDriver

# ↓

interface_driver = quantum.agent.linux.interface.BridgeInterfaceDriver
```

* `/etc/quantum/quantum.conf` を更新

```
[keystone_authtoken]
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = %SERVICE_TENANT_NAME%
admin_user = %SERVICE_USER%
admin_password = %SERVICE_PASSWORD%
signing_dir = /var/lib/quantum/keystone-signing

# ↓

[keystone_authtoken]
auth_host = 10.10.100.51
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = quantum
admin_password = service_pass
signing_dir = /var/lib/quantum/keystone-signing
```

* `/etc/quantum/dhcp_agent.ini` を編集する

```
interface_driver = quantum.agent.linux.interface.OVSInterfaceDriver

# ↓

interface_driver = quantum.agent.linux.interface.BridgeInterfaceDriver
```

* `/etc/quantum/metadata_agent.ini` を更新する

```
[DEFAULT]
# Show debugging output in log (sets DEBUG log level output)
# debug = True

# The Quantum user information for accessing the Quantum API.
#auth_url = http://localhost:35357/v2.0
#auth_region = RegionOne
#admin_tenant_name = %SERVICE_TENANT_NAME%
#admin_user = %SERVICE_USER%
#admin_password = %SERVICE_PASSWORD%
auth_url = http://10.10.100.51:35357/v2.0
auth_region = RegionOne
admin_tenant_name = service
admin_user = quantum
admin_password = service_pass


# IP address used by Nova metadata server
# nova_metadata_ip = 127.0.0.1
nova_metadata_ip = 10.10.100.51


# TCP Port used by Nova metadata server
# nova_metadata_port = 8775
nova_metadata_port = 8775

# When proxying metadata requests, Quantum signs the Instance-ID header with a
# shared secret to prevent spoofing.  You may select any string for a secret,
# but it must match here and in the configuration used by the Nova Metadata
# Server. NOTE: Nova uses a different key: quantum_metadata_proxy_shared_secret
# metadata_proxy_shared_secret =
metadata_proxy_shared_secret = helloOpenStack
```

* Quantumのサービス群を再起動する

```
cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

quantum-dhcp-agent stop/waiting
quantum-dhcp-agent start/running, process 12527
quantum-l3-agent stop/waiting
quantum-l3-agent start/running, process 12540
quantum-metadata-agent stop/waiting
quantum-metadata-agent start/running, process 12549
quantum-plugin-linuxbridge-agent stop/waiting
quantum-plugin-linuxbridge-agent start/running, process 12558
quantum-server stop/waiting
quantum-server start/running, process 12569



service dnsmasq restart

 * Restarting DNS forwarder and DHCP server dnsmasq        [ OK ]
```

* Note: 'dnsmasq' fails to restart if already a service is running on port 53. In that case, kill that service before 'dnsmasq' restart

## Nova

コンピューティングコンポーネントのセットアップ。

### KVM

* サーバーが仮想化に対応しているか確認

```
apt-get -y install cpu-checker
kvm-ok

KVM acceleration can be used
```

* KVMインストール、設定

```
apt-get install -y kvm libvirt-bin pm-utils
```

* `/etc/libvirt/qemu.conf` を編集

```
cgroup_device_acl = [
"/dev/null", "/dev/full", "/dev/zero",
"/dev/random", "/dev/urandom",
"/dev/ptmx", "/dev/kvm", "/dev/kqemu",
"/dev/rtc", "/dev/hpet","/dev/net/tun"
]
```

* デフォルトの仮想ブリッジの削除

```
# virsh net-destroy default
Network default destroyed


# virsh net-undefine default
Network default has been undefined
```

* `/etc/libvirt/libvirtd.conf` を編集してライブマイグレーションを有効にする

```
listen_tls = 0
listen_tcp = 1
auth_tcp = "none"
```

* `/etc/init/libvirt-bin.conf` libvirtd_opts 変数を編集

```
env libvirtd_opts="-d"

# ↓

env libvirtd_opts="-d -l"
```

* libvirtサービスを再起動

```
service libvirt-bin restart
```

### Nova-*

Novaを構成するサブコンポーネントはたくさん。

```
apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor nova-compute-kvm
```

* Novaのサービスが起動していることを確認する

```
cd /etc/init.d/; for i in $( ls nova-* ); do service $i status; cd; done

nova-api start/running, process 16248
nova-cert start/running, process 16288
nova-compute start/running, process 16547
nova-conductor start/running, process 16345
nova-consoleauth start/running, process 16385
nova-novncproxy start/running, process 16425
nova-scheduler start/running, process 16465
```

* MySQLのDB用意

```
mysql -u root -ppassword
CREATE DATABASE nova;
GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass';
quit;
```

* `/etc/nova/api-paste.ini` の authtokenセクションを編集する

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = %SERVICE_TENANT_NAME%
admin_user = %SERVICE_USER%
admin_password = %SERVICE_PASSWORD%
# signing_dir is configurable, but the default behavior of the authtoken
# middleware should be sufficient.  It will create a temporary directory
# in the home directory for the user the nova process is running as.
#signing_dir = /var/lib/nova/keystone-signing
# Workaround for https://bugs.launchpad.net/nova/+bug/1154809
auth_version = v2.0

# ↓

[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = 10.10.100.51
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = service_pass
signing_dirname = /tmp/keystone-signing-nova
# Workaround for https://bugs.launchpad.net/nova/+bug/1154809
auth_version = v2.0
```

* `/etc/nova/nova.conf`

```
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
iscsi_helper=tgtadm
libvirt_use_virtio_for_bridges=True
connection_type=libvirt
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
volumes_path=/var/lib/nova/volumes
enabled_apis=ec2,osapi_compute,metadata

```

を

```
[DEFAULT]
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/run/lock/nova
verbose=True
api_paste_config=/etc/nova/api-paste.ini
compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
rabbit_host=10.10.100.51
nova_url=http://10.10.100.51:8774/v1.1/
sql_connection=mysql://novaUser:novaPass@10.10.100.51/nova
root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

# Auth
use_deprecated_auth=false
auth_strategy=keystone

# Imaging service
glance_api_servers=10.10.100.51:9292
image_service=nova.image.glance.GlanceImageService

# Vnc configuration
novnc_enabled=true
novncproxy_base_url=http://192.168.1.200:6080/vnc_auto.html
novncproxy_port=6080
vncserver_proxyclient_address=10.10.100.51
vncserver_listen=0.0.0.0

# Metadata
service_quantum_metadata_proxy = True
quantum_metadata_proxy_shared_secret = helloOpenStack

# Network settings
network_api_class=nova.network.quantumv2.api.API
quantum_url=http://10.10.100.51:9696
quantum_auth_strategy=keystone
quantum_admin_tenant_name=service
quantum_admin_username=quantum
quantum_admin_password=service_pass
quantum_admin_auth_url=http://10.10.100.51:35357/v2.0
libvirt_vif_driver=nova.virt.libvirt.vif.QuantumLinuxBridgeVIFDriver
linuxnet_interface_driver=nova.network.linux_net.LinuxBridgeInterfaceDriver
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

# Compute #
compute_driver=libvirt.LibvirtDriver

# Cinder #
volume_api_class=nova.volume.cinder.API
osapi_volume_listen_port=5900
```

* `/etc/nova/nova-compute.conf`

```
[DEFAULT]
libvirt_type=kvm
compute_driver=libvirt.LibvirtDriver

# ↓

[DEFAULT]
libvirt_type=kvm
compute_driver=libvirt.LibvirtDriver
libvirt_vif_type=ethernet
libvirt_vif_driver=nova.virt.libvirt.vif.QuantumLinuxBridgeVIFDriver
```

* DB同期

```
nova-manage db sync
```

* Nova系のサービスを再起動

```
cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done

nova-api stop/waiting
nova-api start/running, process 16640
nova-cert stop/waiting
nova-cert start/running, process 16651
nova-compute stop/waiting
nova-compute start/running, process 16663
nova-conductor stop/waiting
nova-conductor start/running, process 16674
nova-consoleauth stop/waiting
nova-consoleauth start/running, process 16693
nova-novncproxy stop/waiting
nova-novncproxy start/running, process 16704
nova-scheduler stop/waiting
nova-scheduler start/running, process 16717
```

* Novaのインストールが正常に完了していることを確認する。スマイリーマークが表示される。

```
nova-manage service list

Binary           Host                                 Zone             Status     State Updated_At
nova-cert        stack01                              internal         enabled    :-)   2013-09-21 04:13:23
nova-consoleauth stack01                              internal         enabled    :-)   2013-09-21 04:13:23
nova-conductor   stack01                              internal         enabled    :-)   2013-09-21 04:13:23
nova-scheduler   stack01                              internal         enabled    :-)   2013-09-21 04:13:24
nova-compute     stack01                              nova             enabled    :-)   2013-09-21 04:13:24
```

## Cinder

ブロックストレージコンポーネント。  
LVMのボリュームグループを `volume_group00` ブロックデバイスとして切り出す。  
AWSのEBSに相当。

```
apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms
```

* ISCSIサービスを設定

```
sed -i 's/false/true/g' /etc/default/iscsitarget
```

* サービス再起動

```
# service iscsitarget start
 * Starting iSCSI enterprise target service                      [ OK ]
                                                                 [ OK ]
# service open-iscsi start
 * Setting up iSCSI targets                                      [ OK ]
```

* MySQL DB用意

```
mysql -u root -ppassword
CREATE DATABASE cinder;
GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass';
quit;
```

* authtoken設定 `/etc/cinder/api-paste.ini`

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
service_protocol = http
service_host = 127.0.0.1
service_port = 5000
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = %SERVICE_TENANT_NAME%
admin_user = %SERVICE_USER%
admin_password = %SERVICE_PASSWORD%
signing_dir = /var/lib/cinder
```

を

```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
service_protocol = http
service_host = 192.168.1.200
service_port = 5000
auth_host = 10.10.100.51
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = cinder
admin_password = service_pass
```

* `/etc/cinder/cinder.conf` Cinder設定

```
[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
```

```
[DEFAULT]
rootwrap_config=/etc/cinder/rootwrap.conf
sql_connection = mysql://cinderUser:cinderPass@10.10.100.51/cinder
api_paste_config = /etc/cinder/api-paste.ini
iscsi_helper=ietadm
volume_name_template = volume-%s
volume_group = volume_group00
verbose = True
auth_strategy = keystone
#osapi_volume_listen_port=5900
```

とする。

* DB同期

```
cinder-manage db sync
```

* サービス再起動

```
cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done

cinder-api stop/waiting
cinder-api start/running, process 20212
cinder-scheduler stop/waiting
cinder-scheduler start/running, process 20223
stop: Unknown instance:
cinder-volume start/running, process 20234
```

* サービスがきちんと動作していることを確認する

```
cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done

cinder-api start/running, process 20268
cinder-scheduler start/running, process 20279
cinder-volume start/running, process 20290
```

## Horizon

長い長い・・・。  
やっとWeb UIのインストールです。

```
apt-get -y install openstack-dashboard memcached
service apache2 restart; service memcached restart
```

これでHorizonにアクセスできる。

- http://192.168.1.200/horizon
- ID: `admin`
- PASS: `admin_pass`

## 仮想マシンインスタンスの作成・起動

* テナントの作成

```
keystone tenant-create --name project_one

+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | 0edd3bab16854ccc9c6d0f02990ee0d0 |
|     name    |           project_one            |
+-------------+----------------------------------+
```

* ユーザーの作成

```
put_id_of_project_one=0edd3bab16854ccc9c6d0f02990ee0d0
keystone user-create --name=user_one --pass=user_one --tenant-id $put_id_of_project_one --email=user_one@domain.com
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |       user_one@domain.com        |
| enabled  |               True               |
|    id    | 6fb1604169fd457a900f3e758fbc0428 |
|   name   |             user_one             |
| tenantId | 0edd3bab16854ccc9c6d0f02990ee0d0 |
+----------+----------------------------------+


keystone role-list
+----------------------------------+----------------------+
|                id                |         name         |
+----------------------------------+----------------------+
| e9c22d652a174f33897b05763932d435 |    KeystoneAdmin     |
| 82d65726132447dabd2c86ffcfe84291 | KeystoneServiceAdmin |
| 0682b13aba9a4842aa4186642b06c09f |        Member        |
| 9fe2ff9ee4384b1894a90878d3e92bab |       _member_       |
| 37c8c3238bfc454b9593b2d7475bbe4e |        admin         |
+----------------------------------+----------------------+


put_id_of_user_one=6fb1604169fd457a900f3e758fbc0428
put_id_of_member_role=37c8c3238bfc454b9593b2d7475bbe4e
keystone user-role-add --tenant-id $put_id_of_project_one  --user-id $put_id_of_user_one --role-id $put_id_of_member_role
```

* テナントのネットワークを作成する

```
quantum net-create --tenant-id $put_id_of_project_one net_proj_one

Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 1ea47f10-3541-472a-b6a2-24c432754d0a |
| name                      | net_proj_one                         |
| provider:network_type     | local                                |
| provider:physical_network |                                      |
| provider:segmentation_id  |                                      |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | 0edd3bab16854ccc9c6d0f02990ee0d0     |
+---------------------------+--------------------------------------+
```

* テナントネットワーク内にサブネットを作成する

```
quantum subnet-create --tenant-id $put_id_of_project_one net_proj_one 50.50.1.0/24

Created a new subnet:
+------------------+----------------------------------------------+
| Field            | Value                                        |
+------------------+----------------------------------------------+
| allocation_pools | {"start": "50.50.1.2", "end": "50.50.1.254"} |
| cidr             | 50.50.1.0/24                                 |
| dns_nameservers  |                                              |
| enable_dhcp      | True                                         |
| gateway_ip       | 50.50.1.1                                    |
| host_routes      |                                              |
| id               | 1c46a9f0-a37b-4218-9f04-981972d0150a         |
| ip_version       | 4                                            |
| name             |                                              |
| network_id       | 1ea47f10-3541-472a-b6a2-24c432754d0a         |
| tenant_id        | 0edd3bab16854ccc9c6d0f02990ee0d0             |
+------------------+----------------------------------------------+
```

* テナントのルーターを作成する

```
quantum router-create --tenant-id $put_id_of_project_one router_proj_one

Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | aa849839-7c65-4b45-ac6d-ef5b25af0cb1 |
| name                  | router_proj_one                      |
| status                | ACTIVE                               |
| tenant_id             | 0edd3bab16854ccc9c6d0f02990ee0d0     |
+-----------------------+--------------------------------------+
```

* 作ったルーターをサブネットに追加する

```
put_router_proj_one_id_here=aa849839-7c65-4b45-ac6d-ef5b25af0cb1
put_subnet_id_here=1c46a9f0-a37b-4218-9f04-981972d0150a
quantum router-interface-add $put_router_proj_one_id_here $put_subnet_id_here

Added interface to router aa849839-7c65-4b45-ac6d-ef5b25af0cb1
```

* Quantumのサービスをすべて再起動する

```
cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

quantum-dhcp-agent stop/waiting
quantum-dhcp-agent start/running, process 22033
quantum-l3-agent stop/waiting
quantum-l3-agent start/running, process 22046
quantum-metadata-agent stop/waiting
quantum-metadata-agent start/running, process 22055
quantum-plugin-linuxbridge-agent stop/waiting
quantum-plugin-linuxbridge-agent start/running, process 22064
quantum-server stop/waiting
quantum-server start/running, process 22073
```

> That's it ! Log on to your dashboard, create your secure key and modify your security groups then create your first VM.

```
エラー: クォータ情報を取得できません。
```

- [Grizzly Unable to retrieve Quota information - RDO](http://openstack.redhat.com/forum/discussion/70/grizzly-unable-to-retrieve-quota-information/p1)
- [Bug #1040956 ""Unable to get quota information” in horizon when ... : Bugs : OpenStack Dashboard (Horizon)](https://bugs.launchpad.net/horizon/+bug/1040956)
- [Installing and configuring Cinder - OpenStack Install and Deploy Manual - Ubuntu  - Folsom, Compute 2012.2, Network 2012.2, Object Storage 1.4.8](http://docs.openstack.org/folsom/openstack-compute/install/apt/content/osfolubuntu-cinder.html)

なんなの。。。  

* デフォルトだとHorizonのログ出力がほとんど出ないのでデバッグモードを有効にする。  
`/etc/openstack-dashboard/local_settings.py`

```
DEBUG = False

# ↓

DEBUG = True
```

ログを流してみる。

```
tailf /var/log/apache2/error.log
 
(snip)
[Sat Sep 21 05:34:16 2013] [error]
[Sat Sep 21 05:34:16 2013] [error] DEBUG:quantumclient.client:
[Sat Sep 21 05:34:16 2013] [error] REQ: curl -i http://192.168.100.51:9696//v2.0/floatingips.json -X GET -H "User-Agent: python-quantumclient" -H "Content-Type: application/json" -H "Accept: application/json" -H "X-Auth-Token: 26ea9049729282276c82df11e242e401"
(snip)
```






apt-get install -y openstack-dashboard libapache2-mod-wsgi


[Horizon "OfflineGenerationError" · Issue #59 · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/issues/59)



apt-get install memcached libapache2-mod-wsgi openstack-dashboard

[1.1.1. OSのインストール — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_ubuntu1304_apt/openstack_control.html)

keystoneのエンドポイントが間違ってる。。。

```
# keystone endpoint-list
+----------------------------------+-----------+---------------------------------------------+-------------------------------------------+-------------------------------------------+----------------------------------+
|                id                |   region  |                  publicurl                  |                internalurl                |                  adminurl                 |            service_id            |
+----------------------------------+-----------+---------------------------------------------+-------------------------------------------+-------------------------------------------+----------------------------------+
| 062a4d2326554cd28a728df45190292a | RegionOne |         http://192.168.100.51:9696/         |         http://10.10.100.51:9696/         |         http://10.10.100.51:9696/         | 64d4be963d7a4dbbb92d26af7cb05474 |
| 11a164f981514c3e8b69db588c9660ac | RegionOne |          http://192.168.1.200:9696/         |         http://10.10.100.51:9696/         |         http://10.10.100.51:9696/         | 64d4be963d7a4dbbb92d26af7cb05474 |
| 16eb684e5772461bb3838bf463ad66e3 | RegionOne |         http://192.168.1.200:9292/v2        |        http://10.10.100.51:9292/v2        |        http://10.10.100.51:9292/v2        | 75f6ce0701dc44c7a02b7a0e64431a34 |
| 17f3379c699c4096a39af644aa906d67 | RegionOne |        http://192.168.1.200:5000/v2.0       |       http://10.10.100.51:5000/v2.0       |       http://10.10.100.51:35357/v2.0      | 420de4954a5a4d85a29e06c4fd69f2d9 |
| 2920019bcc6e4c648d9804b06f131e58 | RegionOne |         http://192.168.1.200:9292/v2        |        http://10.10.100.51:9292/v2        |        http://10.10.100.51:9292/v2        | 75f6ce0701dc44c7a02b7a0e64431a34 |
| 2a92564bab2246ccb02b4f845693a108 | RegionOne |   http://192.168.1.200:8773/services/Cloud  |  http://10.10.100.51:8773/services/Cloud  |  http://10.10.100.51:8773/services/Admin  | 20087db9bd814b139af357ba3613cde8 |
| 46a3f486dc6f421e8d09999f501bf13b | RegionOne |  http://192.168.1.200:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | 09be24f922d3464c9b1c21b1ac39618e |
| 6619b9d2cb184625921d2fcc94ccc793 | RegionOne | http://192.168.100.51:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | 3c1114018c7945ddb5b6202be3f84d0c |
| 81740d673ae2402a84c21020f8cdca9a | RegionOne |  http://192.168.1.200:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | 3c1114018c7945ddb5b6202be3f84d0c |
| 8f2350649de14147a0c93c8175b03a0b | RegionOne |  http://192.168.100.51:8773/services/Cloud  |  http://10.10.100.51:8773/services/Cloud  |  http://10.10.100.51:8773/services/Admin  | 3a310b52dec54c25aee4b5e56679f5c7 |
| 9a36ba227f004eecab2f1c73364efc76 | RegionOne |   http://192.168.1.200:8773/services/Cloud  |  http://10.10.100.51:8773/services/Cloud  |  http://10.10.100.51:8773/services/Admin  | 20087db9bd814b139af357ba3613cde8 |
| a087e33ec98f453499d7bdbf295bbc94 | RegionOne |  http://192.168.1.200:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | 234663ac6fd742cab73a08db40db10a0 |
| a1989b1b18cc4bf888ecc5854d3ad4ec | RegionOne |          http://192.168.1.200:9696/         |         http://10.10.100.51:9696/         |         http://10.10.100.51:9696/         | 64d4be963d7a4dbbb92d26af7cb05474 |
| af27d6dd6bf54f459996d05d47daa341 | RegionOne |          http://192.168.1.200:9696/         |         http://10.10.100.51:9696/         |         http://10.10.100.51:9696/         | 64d4be963d7a4dbbb92d26af7cb05474 |
| afb54b53549d4de0ac362faa72692d43 | RegionOne |  http://192.168.1.200:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | 3c1114018c7945ddb5b6202be3f84d0c |
| afe9d02c1a19424a814efe5a86d4d08b | RegionOne |        http://192.168.1.200:5000/v2.0       |       http://10.10.100.51:5000/v2.0       |       http://10.10.100.51:35357/v2.0      | 420de4954a5a4d85a29e06c4fd69f2d9 |
| b6db7b9724d143718ca940d6c11f6a69 | RegionOne |        http://192.168.1.200:5000/v2.0       |       http://10.10.100.51:5000/v2.0       |       http://10.10.100.51:35357/v2.0      | 420de4954a5a4d85a29e06c4fd69f2d9 |
| b76948e8695945b6ba3889d809a25e94 | RegionOne |        http://192.168.100.51:9292/v2        |        http://10.10.100.51:9292/v2        |        http://10.10.100.51:9292/v2        | bed10273928e4c92ade16f6d7fa86171 |
| bd9763b891334608a95d8899922529e8 | RegionOne |   http://192.168.1.200:8773/services/Cloud  |  http://10.10.100.51:8773/services/Cloud  |  http://10.10.100.51:8773/services/Admin  | 20087db9bd814b139af357ba3613cde8 |
| d063451bc1f64f019eeeb91fa56e4fb1 | RegionOne |         http://192.168.1.200:9292/v2        |        http://10.10.100.51:9292/v2        |        http://10.10.100.51:9292/v2        | 75f6ce0701dc44c7a02b7a0e64431a34 |
| e3720fcc58fe426f8c2bf5b9b3ac18cc | RegionOne |       http://192.168.100.51:5000/v2.0       |       http://10.10.100.51:5000/v2.0       |       http://10.10.100.51:35357/v2.0      | 420de4954a5a4d85a29e06c4fd69f2d9 |
| efea12555c0f42e59e980a1fabc36904 | RegionOne |  http://192.168.1.200:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | 0afbdd5d4ffe4213bc81b18306cd8d4f |
| fc14c3a8277a4667ae376760354cb591 | RegionOne | http://192.168.100.51:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | 234663ac6fd742cab73a08db40db10a0 |
| ff4f3e29392c4d85a82e8d0d5c38e77a | RegionOne |  http://192.168.1.200:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | 234663ac6fd742cab73a08db40db10a0 |
+----------------------------------+-----------+---------------------------------------------+-------------------------------------------+-------------------------------------------+----------------------------------+

```

```
mysql> drop database keystone;
Query OK, 19 rows affected (1.04 sec)

mysql> CREATE DATABASE keystone;
Query OK, 1 row affected (0.00 sec)

mysql> quit;
Bye

service keystone restart
keystone-manage db_sync

root@stack01:~/b4377e74d3825610780f# keystone endpoint-list
Unable to communicate with identity service: {"error": {"message": "The request you have made requires authentication.", "code": 401, "title": "Not Authorized"}}. (HTTP 401)

oops...

sh keystone_basic.sh
sh keystone_endpoints_basic.sh

root@stack01:~/b4377e74d3825610780f# keystone endpoint-list
+----------------------------------+-----------+--------------------------------------------+-------------------------------------------+-------------------------------------------+----------------------------------+
|                id                |   region  |                 publicurl                  |                internalurl                |                  adminurl                 |            service_id            |
+----------------------------------+-----------+--------------------------------------------+-------------------------------------------+-------------------------------------------+----------------------------------+
| 1a28a47862ed417db353bdf7f78b88e2 | RegionOne |         http://192.168.1.200:9696/         |         http://10.10.100.51:9696/         |         http://10.10.100.51:9696/         | 72a2ff928867457ab400f46f88a66467 |
| 41688030d2974de5b9cbd4ef9bc03254 | RegionOne | http://192.168.1.200:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | http://10.10.100.51:8774/v2/$(tenant_id)s | b0eb8067c4c749dc95b709d09b3e28b6 |
| 41c3c1eb7372484a92a8813d411572bd | RegionOne |       http://192.168.1.200:5000/v2.0       |       http://10.10.100.51:5000/v2.0       |       http://10.10.100.51:35357/v2.0      | 0f21d39fd12a4cf1abd6c137fbceec98 |
| 8438510b93ed4f75a2da7ede25d2ad20 | RegionOne |  http://192.168.1.200:8773/services/Cloud  |  http://10.10.100.51:8773/services/Cloud  |  http://10.10.100.51:8773/services/Admin  | d44e75df616b4bfda775ae59eeb0fe77 |
| 84465790fc0d41fbb61a97be01e8c40a | RegionOne | http://192.168.1.200:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | http://10.10.100.51:8776/v1/$(tenant_id)s | 11fb70a50dd44e21862de434e739e00c |
| cb311b737bd94c4daa4443b548b84d12 | RegionOne |        http://192.168.1.200:9292/v2        |        http://10.10.100.51:9292/v2        |        http://10.10.100.51:9292/v2        | 2dcd41936d3d44f98f32010bfdbfc321 |
+----------------------------------+-----------+--------------------------------------------+-------------------------------------------+-------------------------------------------+----------------------------------+
```

endpoint直った。

```
service apache2 restart; service memcached restart
```





* テナントの作成

```
root@stack01:~/b4377e74d3825610780f# keystone tenant-create --name project_one
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |                                  |
|   enabled   |               True               |
|      id     | aece8dea7b664f99b318705faf698e12 |
|     name    |           project_one            |
+-------------+----------------------------------+
```

* ユーザーの作成

```
root@stack01:~/b4377e74d3825610780f# put_id_of_project_one=aece8dea7b664f99b318705faf698e12
root@stack01:~/b4377e74d3825610780f# keystone user-create --name=user_one --pass=user_one --tenant-id $put_id_of_project_one --email=user_one@domain.com
+----------+----------------------------------+
| Property |              Value               |
+----------+----------------------------------+
|  email   |       user_one@domain.com        |
| enabled  |               True               |
|    id    | c0547fd9d0c140feb1fb204159553205 |
|   name   |             user_one             |
| tenantId | aece8dea7b664f99b318705faf698e12 |
+----------+----------------------------------+


root@stack01:~/b4377e74d3825610780f# keystone role-list
+----------------------------------+----------------------+
|                id                |         name         |
+----------------------------------+----------------------+
| 19dac405d88543b19e6230f7ddb1cb44 |    KeystoneAdmin     |
| 2891487f0108415493a227a9cf5dcd51 | KeystoneServiceAdmin |
| e3d57319c8fc46b096dcd85ff9906617 |        Member        |
| 9fe2ff9ee4384b1894a90878d3e92bab |       _member_       |
| 6ed8d8cf00da4383b69b8eb4e015ba2e |        admin         |
+----------------------------------+----------------------+


put_id_of_user_one=c0547fd9d0c140feb1fb204159553205
put_id_of_member_role=6ed8d8cf00da4383b69b8eb4e015ba2e
keystone user-role-add --tenant-id $put_id_of_project_one  --user-id $put_id_of_user_one --role-id $put_id_of_member_role
```

* テナントのネットワークを作成する

```
root@stack01:~/b4377e74d3825610780f# quantum net-create --tenant-id $put_id_of_project_one net_proj_two
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 81906e14-a6d0-43b3-81a8-3021958837dc |
| name                      | net_proj_two                         |
| provider:network_type     | local                                |
| provider:physical_network |                                      |
| provider:segmentation_id  |                                      |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | aece8dea7b664f99b318705faf698e12     |
+---------------------------+--------------------------------------+
```

* テナントネットワーク内にサブネットを作成する

```
quantum subnet-create --tenant-id $put_id_of_project_one net_proj_two 50.50.2.0/24

Created a new subnet:
+------------------+----------------------------------------------+
| Field            | Value                                        |
+------------------+----------------------------------------------+
| allocation_pools | {"start": "50.50.2.2", "end": "50.50.2.254"} |
| cidr             | 50.50.2.0/24                                 |
| dns_nameservers  |                                              |
| enable_dhcp      | True                                         |
| gateway_ip       | 50.50.2.1                                    |
| host_routes      |                                              |
| id               | 51d51d52-cf04-4c20-aff7-ebb0d6c5b852         |
| ip_version       | 4                                            |
| name             |                                              |
| network_id       | 81906e14-a6d0-43b3-81a8-3021958837dc         |
| tenant_id        | aece8dea7b664f99b318705faf698e12             |
+------------------+----------------------------------------------+
```

* テナントのルーターを作成する

```
quantum router-create --tenant-id $put_id_of_project_one net_proj_two

Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | d73be4ac-d6de-43f7-affa-1e5960bb2443 |
| name                  | net_proj_two                         |
| status                | ACTIVE                               |
| tenant_id             | aece8dea7b664f99b318705faf698e12     |
+-----------------------+--------------------------------------+
```

* 作ったルーターをサブネットに追加する

```
put_router_proj_one_id_here=d73be4ac-d6de-43f7-affa-1e5960bb2443
put_subnet_id_here=51d51d52-cf04-4c20-aff7-ebb0d6c5b852
quantum router-interface-add $put_router_proj_one_id_here $put_subnet_id_here

Added interface to router d73be4ac-d6de-43f7-affa-1e5960bb2443
```

* Quantumのサービスをすべて再起動する

```
cd /etc/init.d/; for i in $( ls quantum-* ); do sudo service $i restart; done

quantum-dhcp-agent stop/waiting
quantum-dhcp-agent start/running, process 31380
quantum-l3-agent stop/waiting
quantum-l3-agent start/running, process 31393
quantum-metadata-agent stop/waiting
quantum-metadata-agent start/running, process 31402
quantum-plugin-linuxbridge-agent stop/waiting
quantum-plugin-linuxbridge-agent start/running, process 31411
quantum-server stop/waiting
quantum-server start/running, process 31420
```




## 参考サイト

- [mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide)
- [OpenStack-Grizzly-Install-Guide/OpenStack_Grizzly_Install_Guide.rst at master · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/blob/master/OpenStack_Grizzly_Install_Guide.rst)
- [OSSはアルミニウムの翼で飛ぶ: OpenStack/Essex Configuration 05:Nova](http://aikotobaha.blogspot.jp/2012/05/openstackessex-configuration-05nova.html)
- [6. KeyStone インストール手順 — OpenStack インストール手順のようなもの(Essex版)](http://2done.org/openstack/essex/install/keystone.html)
- [さくらの専用サーバとOpenStackで作るプライベートクラウド 7ページ | SourceForge.JP Magazine](http://sourceforge.jp/magazine/12/09/18/1126211/7)
- [OpenStack Installation Guide for Ubuntu 12.04 (LTS) - OpenStack Installation Guide for Ubuntu 12.04 (LTS)  - master](http://docs.openstack.org/trunk/openstack-compute/install/apt/content/)
- [Install and configure the dashboard - OpenStack Installation Guide for Ubuntu 12.04 (LTS)  - master](http://docs.openstack.org/trunk/openstack-compute/install/apt/content/installing-openstack-dashboard.html)
- [1.1.1. OSのインストール — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_ubuntu1304_apt/openstack_control.html)
- [10.3.1. keystoneコマンド — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_com/keystone_com.html)
