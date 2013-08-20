# NeutronでVLANとか

VyattaとかAsteriskとか入れたいねん。

## 概要

**このエントリは失敗しています。  
ムダに長いので読まないほうがいいです。**

最近コードネームが Quantum から Neutron に変わりました。  
ご注意を。

### 構成要素

- Quantum Server
- L3 Agent
- DHCP Agent
- Plugin Agent

## Neutronのインストール

```
yum -y install openstack-quantum
```

また、ネットワークノードおよび計算ノードにはopenstack-quantum-linuxbridgeパッケージをインストールしておく。

```
yum -y install openstack-quantum-linuxbridge
```

なお、今回使用したopenstack-quantumパッケージのバージョンは下記のとおりである。

```
rpm -q openstack-quantum
openstack-quantum-2013.1.2-2.el6.noarch
```

## Neutronのサービス登録

OpenStack管理ユーザー `stack` としてログインして `~/keystonerc` が既に読み込まれているものと仮定する。

### 変数設定

リージョン `RegionOne` を指定する。

```
NEUTRON_HOST=stack01
REGION=RegionOne
```

Neutron サービス作成。

```
keystone service-create --name=neutron --type=network --description="Neutron Network Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description |     Neutron Network Service      |
|      id     | c86c3207100a47cba2f541f255bb5e56 |
|     name    |             neutron              |
|     type    |             network              |
+-------------+----------------------------------+
```

### リージョン RegionOne で登録

#### サービスID取得

```
SERVICE_ID=`keystone service-list | grep neutron | awk '{print $2}'`
echo $SERVICE_ID
c86c3207100a47cba2f541f255bb5e56
```

#### エンドポイントの登録

```
keystone endpoint-create --region $REGION --service_id=$SERVICE_ID --publicurl "http://$NEUTRON_HOST:9696/" --adminurl "http://$NEUTRON_HOST:9696/" --internalurl "http://$NEUTRON_HOST:9696/"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |       http://stack01:9696/       |
|      id     | c78d75efab8a46699d4f0441ad1ceb4b |
| internalurl |       http://stack01:9696/       |
|  publicurl  |       http://stack01:9696/       |
|    region   |            RegionOne             |
|  service_id | c86c3207100a47cba2f541f255bb5e56 |
+-------------+----------------------------------+
```

#### サービスの一覧を確認する

```
[stack@stack01 ~]$ keystone service-list
+----------------------------------+----------+--------------+---------------------------+
|                id                |   name   |     type     |        description        |
+----------------------------------+----------+--------------+---------------------------+
| 60f2d756fab54276afe55b9c24c20552 |   ec2    |     ec2      |  EC2 Compatibility Layer  |
| 2ac0ba678edd4b7db33eeece3028ec09 |  glance  |    image     |    Glance Image Service   |
| 0de4603b303e4295b75a25d860d6f694 | keystone |   identity   | Keystone Identity Service |
| c86c3207100a47cba2f541f255bb5e56 | neutron  |   network    |  Neutron Network Service  |
| 4b9aad7a03584f468bc742ef2f9dab69 |   nova   |   compute    |    Nova Compute Service   |
| 2ba468927ff344acb808ffac6d2307b5 |  swift   | object-store |       Swift Service       |
| 6ac3bd80e21c4f47a2f2086cda720844 |  volume  |    volume    |    Nova Volume Service    |
+----------------------------------+----------+--------------+---------------------------+
```

## Neutronの設定

### 制御ノード側

以下のコマンドを実行する。  
実行すると設定ファイルも変更されるのでgit diffとかしてみて変更内容確認してみよう。  
rootで実行する。

ここでやっていることは

- ネットワークプラグインに LinuxBridge を使用するように指定。  
CentOSだとOpen vSwitchに対応していない。  
インストール自体はできるけどなんかうまく動かない。
- 内部ネットワークとの接続を行う、VLANトランクポートとして設定するNICを eth0 で指定している。
- 設定ファイルもこっちでよろしく変えちゃってよろしいか？

```
[root@stack01 quantum]# quantum-server-setup -q password
Please select a plugin from: linuxbridge openvswitch
Choice:
linuxbridge
Quantum plugin: linuxbridge
Plugin: linuxbridge => Database: quantum_linux_bridge
Please enter the password for the 'root' MySQL user: 
Verified connectivity to MySQL.
Please enter network device for VLAN trunking:
eth0
Would you like to update the nova configuration files? (y/n): 
y
Configuration updates complete!
```

CentOS6.3系ではネットワークネームスペースがサポートされていないので

```
allow_overlapping_ips = False
```

とする必要があるらしんですが、6.4系では有効になってるのかなっと思ったら

[CentOS 6.4 で Linux Network Namespace (netns) を使う | CUBE SUGAR STORAGE](http://momijiame.tumblr.com/post/57789158893/centos-6-4-linux-network-namespace-netns)

そのままじゃだめらしい。  
OpenStackのアーキテクチャに相当慣れてからじゃないとCentOSを選定するのは難しそうだってことは分かった。  
とりあえずQuantumの使用感になれてみようか。  
あとでOpenStackをUbuntuで再構築してみよう。

#### RabbitMQの設定

とりあえず以下のメッセージングサービスRabbitMQの設定を有効にする。

```
rpc_backend = quantum.openstack.common.rpc.impl_kombu
rabbit_host = stack01
rabbit_port = 5672
rabbit_userid = guest
rabbit_password = guest
rabbit_virtual_host = /
```

#### api-paste.iniの設定

`[filter:authtoken]` のセクションに以下の内容を追加します。

```
auth_host = stack01
auth_port = 35357
auth_protocol = http
admin_tenant_name = demo
admin_user = admin
admin_password = secrete
```

例によって例のごとくパッチファイル作った。

```
cd /etc/quantum

wget https://raw.github.com/wnoguchi/doc/master/OpenStack/Neutron/config/quantum-server-setup.patch
patch -p1 --dry-run < quantum-server-setup.patch
patch -p1 < quantum-server-setup.patch

wget https://raw.github.com/wnoguchi/doc/master/OpenStack/Neutron/config/quantum-config.patch
patch -p1 --dry-run < quantum-config.patch
patch -p1 < quantum-config.patch
```

#### Neutronスタート

```
service quantum-server start
chkconfig quantum-server on
```

### ネットワークノード側

ネットワークノードでは上記制御ノードと同じ設定に加えて、

- L3 Agentの設定
- DHCP Agentの設定
- プラグインの設定
- sudoersファイルの設定
- ブリッジ関連のカーネルパラメータ変更
- dnsmasqパッケージのバージョン確認とアップデート

をやらないといけないらしい。。。

`~/keystonerc` を予め読み込んであるものと仮定します。

```
source /home/stack/keystonerc
```

#### L3 Agentの設定

```
[root@stack01 ~]# quantum-l3-setup
Please select a plugin from: linuxbridge openvswitch
Choice:
linuxbridge
Quantum plugin: linuxbridge
Please enter the Quantum hostname:
stack01
Configuration updates complete!
```

#### DHCP Agentの設定

```
[root@stack01 ~]# quantum-dhcp-setup
Please select a plugin from: linuxbridge openvswitch
Choice:
linuxbridge
Quantum plugin: linuxbridge
Please enter the Quantum hostname:
stack01
Configuration updates complete!
```

ここまでで `/etc/quantum/` 以下が設定されているので確認しといたほうがいいです。

#### Plugin Agent: linuxbridge-agentの設定

`/etc/quantum/plugins/linuxbridge/linuxbridge_conf.ini` を編集する。

**see patch.**

#### sudoersの設定

```
# visudo
quantum ALL=(ALL) NOPASSWD:ALL
```

#### カーネルパラメータの設定

ブリッジインタフェース通るときにiptablesのフィルタがかからないようにする。  
0になっているか確認する。

```
cat /proc/sys/net/bridge/bridge-nf-call-iptables
1
cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
1
cat /proc/sys/net/bridge/bridge-nf-call-arptables
1
```

だめみたい。0にする。

```
echo 0 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 0 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
echo 0 > /proc/sys/net/bridge/bridge-nf-call-arptables
```

今度はどうだ。

```
[root@stack01 quantum]# cat /proc/sys/net/bridge/bridge-nf-call-iptables
0
[root@stack01 quantum]# cat /proc/sys/net/bridge/bridge-nf-call-ip6tables
0
[root@stack01 quantum]# cat /proc/sys/net/bridge/bridge-nf-call-arptables
0
```

OK。  
次は永続化。

```
# vi /etc/rc.local
echo 0 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 0 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
echo 0 > /proc/sys/net/bridge/bridge-nf-call-arptables
```

> なお、これらのデフォルト値は/etc/sysctl.conf内に記述されているのだが、Red Hat Enterprise Linux 6.3（およびその互換環境）ではbridge-nf-call-iptablesおよびbridge-nf-call-ip6tables、bridge-nf-call-arptablesの値が正しく反映されないというバグが確認されている。もし/etc/sysctl.conf内でこれらの値を0に設定しているにも関わらずデフォルト値が1になっている場合は、下記を/etc/rc.localファイルに記述しておくことで対処できるとのことだ。

##### IPフォワーディング有効かどうか

1ならOK。

```
cat /proc/sys/net/ipv4/ip_forward
1
```

#### dnsmasqのアップデート

```
rpm -q dnsmasq
dnsmasq-2.48-13.el6.x86_64
```

> これらに加え、DHCP Agentが利用するdnsmasqパッケージのバージョンの確認も必要だ。記事作成時点でCentOS 6.3で提供されているdnsmasqのバージョンは2.48なのだが、DHCP Agentはdnsmasq 2.59以上を必要とする。そのため、より新しいバージョンのdnsmasqを別途インストールする必要がある。今回は、DAGリポジトリで提供されているdnsmasqパッケージをインストールすることにする。DAGリポジトリ（x86_64向け）もしくはDAGリポジトリ（x86向け）からdnsmasqパッケージをダウンロードしてインストールしておこう。たとえばdnsmasq-2.63-1.el6.rfx.x86_64.rpmというパッケージをインストールする場合、以下のようにすれば良い。

バージョン古いそうです。  
やること多すぎ・・・。。。。  
アップデートします。

```
rpm -Uvh http://apt.sw.be/redhat/el6/en/x86_64/extras/RPMS/dnsmasq-2.65-1.el6.rfx.x86_64.rpm
```

確認。

```
rpm -q dnsmasq
dnsmasq-2.65-1.el6.rfx.x86_64
```

うん、汚染されちゃったけどOKですね。

```
service quantum-linuxbridge-agent start &&
service quantum-l3-agent start &&
service quantum-dhcp-agent start &&
chkconfig quantum-linuxbridge-agent on &&
chkconfig quantum-l3-agent on &&
chkconfig quantum-dhcp-agent on
```

### 計算機ノード Nova 側設定

```
cd /etc/nova/
vi nova.conf
```

```
network_api_class = nova.network.quantumv2.api.API
quantum_admin_username = quantum
quantum_admin_password = password
quantum_admin_auth_url = http://stack01:35357/v2.0/
quantum_auth_strategy = Keystone
quantum_admin_tenant_name = demo
quantum_url = http://stack01:9696/
libvirt_vif_driver = nova.virt.libvirt.vif.QuantumLinuxBridgeVIFDriver
```

#### sudoersの設定

オールインワン構成なら先ほど設定したので大丈夫です。

```
# visudo
quantum ALL=(ALL) NOPASSWD:ALL
```

#### KVMの設定

`/etc/libvirt/qemu.conf` を編集します。

```
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet", "/dev/net/tun"
]
```

#### 起動設定

```
service openstack-nova-compute restart &&
service libvirtd restart &&
service quantum-linuxbridge-agent start &&
chkconfig openstack-nova-compute on &&
chkconfig libvirtd on &&
chkconfig quantum-linuxbridge-agent on
```

## 運用

制御ノード上で `stack` ユーザーになって行います。  
`keystonerc` は読み込みましょう。

### ネットワークの作成

```
TENAND_ID=45a9e17a91be42d48a7c58b202172f69
quantum net-create --tenant-id=$TENANT_ID net1 --shared --provider:network_type flat --provider:physical_network physnet1
Invalid input for operation: Unknown provider:physical_network physnet1.
```

## Conclusion

現状の自分のスキルでは続行は無理と判断。  
UbuntuとDevStackで構築しなおしてみる。

- Neutronになってコマンド体系がかなり変わってしまった
- LinuxBridgeは流行らない（出てくる記事はOpen vSwitchのものばっかり）
- そもそもCentOSベースで開発されていない（Ubuntuがベース）

疲れた・・・。

## 参考サイト

- [OpenStackの仮想ネットワーク管理機能「Quantum」の基本的な設定 - さくらのナレッジ](http://knowledge.sakura.ad.jp/tech/195/)
- [OpenStack の Quantum + Open vSwitch でVLAN構成を試してみた - Qiita [キータ]](http://qiita.com/m_doi/items/f561894a1f83de40e7dd)
- [OpenStack の Quantum + Linux Bridge でVLAN構成を試してみた - Qiita [キータ]](http://qiita.com/m_doi/items/f3439b78a02b20f1f4a0)
- [CentOS 6.4 で Linux Network Namespace (netns) を使う | CUBE SUGAR STORAGE](http://momijiame.tumblr.com/post/57789158893/centos-6-4-linux-network-namespace-netns)
- [OpenStackの仮想ネットワーク管理機能「Quantum」の基本的な設定 | SourceForge.JP Magazine](http://sourceforge.jp/magazine/13/04/11/080000)
- [Virtual Network Service (Quantum)のインストール — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_folsom_ubuntu1204_apt/quantum_install.html)
