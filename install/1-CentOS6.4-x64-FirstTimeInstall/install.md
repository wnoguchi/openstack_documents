# 初めてのOpenStackインストールの記録

初めてOpenStackを導入したときの記録です。  
まだ幼い。  
CentOS 6.4 x64 + epel で組みました。
**OpenStack Grizzly** です。

なんか、元々RackspaceではUbuntuベースでOpenStackの開発が進められているみたいで、  
CentOSでインストールする記事は少なめ。  
やっぱりUbuntuメインかなあ。

## インストール

とりあえずオールインワン構成でインストールしてみよう。  
インストールに関しては  
[3. Openstackインストール手順(Grizzly)CentOS6.4(パッケージ)編 —
オープンソースに関するドキュメント 1.1
documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/)  
を大いに参考にさせていただいています。

最近はこのへんのことを全自動でやってくれるものがいろいろあるようですが、  
これからプライベートクラウドを運用しようとしてこれだけ複雑なものを
初めから全自動でインストールしようとすると、どこで何が起こっているのかわからないし、  
トラブった時に死ぬので、いったん一からコンポーネントをひとつひとつインストールしていこうと思います。

これでも結構簡単になったほうで、日本語のドキュメントがあるだけありがたいです。

### システム構成

- サーバー: FUJITSU Server PRIMERGY RX100 S7
- OS: CentOS 6.4 x64
- 実装メモリ: 16GB
- ディスク: RAIDなし
  * HDD 1TB Cinder用のディスク
  * SSD 80GB 制御用ノード、Novaノード用のOS格納ディスク、容量はちっちゃくていい
- ホスト名: stack01
- IPアドレス: 192.168.0.10/24

このサーバー好きだわ。
僕が実験で構築したネットワーク構成図を書くとこんな感じ？

![](img/experimental_network.png)

ちなみにこのマークダウンを整理する前の手順書では以下の構成でも動きました。

- サーバー: FUJITSU Server PRIMERGY RX100 S7
- OS: CentOS 6.4 x64
- 実装メモリ: 16GB
- ディスク: HDD 1TB RAIDなし 
- ホスト名: stack01
- IPアドレス: 192.168.0.10/24

Minimalインストール、またはMinimal Desktop構成インストールとします。

参考までにベンチマーク結果を貼っておきます。

```
========================================================================[34/429]
   BYTE UNIX Benchmarks (Version 5.1.3)

   System: stack01: GNU/Linux
   OS: GNU/Linux -- 2.6.32-358.14.1.el6.x86_64 -- #1 SMP Tue Jul 16 23:51:20 UTC
 2013
   Machine: x86_64 (x86_64)
   Language: en_US.utf8 (charmap="UTF-8", collate="UTF-8")
   CPU 0: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz (6185.7 bogomips)
          Hyper-Threading, x86-64, MMX, Physical Address Ext, SYSENTER/SYSEXIT, 
SYSCALL/SYSRET, Intel virtualization
   CPU 1: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz (6185.7 bogomips)
          Hyper-Threading, x86-64, MMX, Physical Address Ext, SYSENTER/SYSEXIT, 
SYSCALL/SYSRET, Intel virtualization
   CPU 2: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz (6185.7 bogomips)
          Hyper-Threading, x86-64, MMX, Physical Address Ext, SYSENTER/SYSEXIT, 
SYSCALL/SYSRET, Intel virtualization
   CPU 3: Intel(R) Xeon(R) CPU E3-1220 V2 @ 3.10GHz (6185.7 bogomips)
          Hyper-Threading, x86-64, MMX, Physical Address Ext, SYSENTER/SYSEXIT, 
SYSCALL/SYSRET, Intel virtualization
   15:38:37 up  5:40,  1 user,  load average: 0.06, 0.03, 0.01; runlevel 3

------------------------------------------------------------------------
Benchmark Run: 火  8月 13 2013 15:38:37 - 16:06:47
4 CPUs in system; running 1 parallel copy of tests

Dhrystone 2 using register variables       35102212.9 lps   (10.0 s, 7 samples)
Double-Precision Whetstone                     3301.3 MWIPS (9.7 s, 7 samples)
Execl Throughput                               4926.6 lps   (30.0 s, 2 samples)
File Copy 1024 bufsize 2000 maxblocks       1032589.7 KBps  (30.0 s, 2 samples)
File Copy 256 bufsize 500 maxblocks          273557.8 KBps  (30.0 s, 2 samples)
File Copy 4096 bufsize 8000 maxblocks       2599445.2 KBps  (30.0 s, 2 samples)
Pipe Throughput                             1857960.4 lps   (10.0 s, 7 samples)
Pipe-based Context Switching                 244547.9 lps   (10.0 s, 7 samples)
Process Creation                              18531.1 lps   (30.0 s, 2 samples)
Shell Scripts (1 concurrent)                   8144.0 lpm   (60.0 s, 2 samples)
Shell Scripts (8 concurrent)                   3016.1 lpm   (60.0 s, 2 samples)
System Call Overhead                        2458175.4 lps   (10.0 s, 7 samples)

System Benchmarks Index Values               BASELINE       RESULT    INDEX
Dhrystone 2 using register variables         116700.0   35102212.9   3007.9
Double-Precision Whetstone                       55.0       3301.3    600.2
Execl Throughput                                 43.0       4926.6   1145.7
File Copy 1024 bufsize 2000 maxblocks          3960.0    1032589.7   2607.5
File Copy 256 bufsize 500 maxblocks            1655.0     273557.8   1652.9
File Copy 4096 bufsize 8000 maxblocks          5800.0    2599445.2   4481.8
Pipe Throughput                               12440.0    1857960.4   1493.5
Pipe-based Context Switching                   4000.0     244547.9    611.4
Process Creation                                126.0      18531.1   1470.7
Shell Scripts (1 concurrent)                     42.4       8144.0   1920.8
Shell Scripts (8 concurrent)                      6.0       3016.1   5026.8
System Call Overhead                          15000.0    2458175.4   1638.8
                                                                   ========
System Benchmarks Index Score                                        1754.2

------------------------------------------------------------------------
Benchmark Run: 火  8月 13 2013 16:06:47 - 16:34:54
4 CPUs in system; running 4 parallel copies of tests

Dhrystone 2 using register variables      140402904.0 lps   (10.0 s, 7 samples)
Double-Precision Whetstone                    13079.8 MWIPS (9.6 s, 7 samples)
Execl Throughput                              23040.9 lps   (30.0 s, 2 samples)
File Copy 1024 bufsize 2000 maxblocks       1275057.3 KBps  (30.0 s, 2 samples)
File Copy 256 bufsize 500 maxblocks          325174.4 KBps  (30.0 s, 2 samples)
File Copy 4096 bufsize 8000 maxblocks       3751435.5 KBps  (30.0 s, 2 samples)
Pipe Throughput                             7381949.3 lps   (10.0 s, 7 samples)
Pipe-based Context Switching                1573885.9 lps   (10.0 s, 7 samples)
Process Creation                              72698.9 lps   (30.0 s, 2 samples)
Shell Scripts (1 concurrent)                  25306.1 lpm   (60.0 s, 2 samples)
Shell Scripts (8 concurrent)                   3401.9 lpm   (60.0 s, 2 samples)
System Call Overhead                        8031145.3 lps   (10.0 s, 7 samples)

System Benchmarks Index Values               BASELINE       RESULT    INDEX
Dhrystone 2 using register variables         116700.0  140402904.0  12031.1
Double-Precision Whetstone                       55.0      13079.8   2378.2
Execl Throughput                                 43.0      23040.9   5358.3
File Copy 1024 bufsize 2000 maxblocks          3960.0    1275057.3   3219.8
File Copy 256 bufsize 500 maxblocks            1655.0     325174.4   1964.8
File Copy 4096 bufsize 8000 maxblocks          5800.0    3751435.5   6468.0
Pipe Throughput                               12440.0    7381949.3   5934.0
Pipe-based Context Switching                   4000.0    1573885.9   3934.7
Process Creation                                126.0      72698.9   5769.8
Shell Scripts (1 concurrent)                     42.4      25306.1   5968.4
Shell Scripts (8 concurrent)                      6.0       3401.9   5669.9
System Call Overhead                          15000.0    8031145.3   5354.1
                                                                   ========
System Benchmarks Index Score                                        4809.4
```

### IPアドレス固定化

```
# /etc/sysconfig/network-scripts/ifcfg-eth0
BOOTPROTO=static
IPADDR=192.168.0.10
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
```

### IPv6無効化

コメントアウトもしくは削除。
stack01のホスト名前解決を追加。

```
# /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 
#::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
127.0.0.1   stack01
```

### SELinux無効化

よろしくやっておいてください。

### 必要最低限のパッケージを入れる

gitは必ずしも必要じゃないですけど、設定ファイルのバージョン管理には便利です。  
デフォルトの設定ファイルからどれだけ変更が加えられているのかがdiffで一目瞭然なので。

```
yum install -y ntp man openssh-clients wget git &&
service ntpd start &&
chkconfig ntpd on
```

### EPELリポジトリーを有効化する

まずepel基本パッケージを有効化します。

```
wget ftp://ftp.riken.jp/Linux/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm &&
rpm -ivh epel-release-6-8.noarch.rpm
```

さらにOpenStack Grizzlyのepelリポジトリも有効化します。

```
cat <<EOF >/etc/yum.repos.d/openstack-grizzly.repo
[epel-openstack-grizzly]
name=OpenStack Grizzly Repository for EPEL 6
baseurl=http://repos.fedorapeople.org/repos/openstack/openstack-grizzly/epel-6/
enabled=1
skip_if_unavailable=1
gpgcheck=0
EOF
yum clean all
yum -y update
```

これやらないとあとで

[Keystoneが起動しない件](Keystoneが起動しない件.md)

みたいなことになるので注意。

**また、このGrizzlyのepelリポジトリを有効にすることによってカーネルイメージの更新等がかかるので、
相当派手な更新がかかるものと思ったほうがいいと思います。
なので安定稼働中の既存のシステムにOpenStackを組み込むのはやめといたほうがいいです。
というか推奨しません。**

### MySQLのセットアップ

```
yum -y install mysql-server &&
service mysqld start &&
chkconfig mysqld on
mysql -uroot -pnova -e "GRANT ALL PRIVILEGES ON *.* TO root@stack01 IDENTIFIED BY 'nova' WITH GRANT OPTION;"
mysql -uroot -pnova -e "set password for root@localhost=password('nova');"
mysql -uroot -pnova -e "set password for root@127.0.0.1=password('nova');"
mysql -uroot -pnova -e "set password for root@stack01=password('nova');"
```

ところで、上のやつで

```
mysql -uroot -pnova -e "GRANT ALL PRIVILEGES ON *.* TO root@stack01 IDENTIFIED BY 'nova' WITH GRANT OPTION;"
```

をやらないでいると

```
ERROR 1133 (42000) at line 1: Can't find any matching row in the user table
```

と怒られるので注意。  
MySQLインストール時にそのホスト名で予めレコードが作成されている模様。

### memcachedのセットアップ

```
yum -y install memcached &&
service memcached start &&
chkconfig memcached on
```

### Apache Qpidのインストール

認証オフ。

```
yum -y install qpid-cpp-server &&
sed -i 's/auth=yes/auth=no/' /etc/qpidd.conf &&
service qpidd restart &&
chkconfig qpidd on
```

### 仮想化関連パッケージのインストール

KVM関係はもちろんのこと、iSCSI関係のパッケージもインストールします。  
インストールが終わったらひたすらサービスの再起動を行います。  
OpenStackぐらいコンポーネントがガチャガチャに組み合わさってくるとタイミングの問題とかでうまくいかないことがあって、
ひたすらサービスの再起動をよくやります。

```
yum install -y iscsi-initiator-utils qemu-kvm libvirt bridge-utils libvirt-python avahi &&
service messagebus restart &&
service avahi-daemon restart &&
service libvirtd restart &&
service messagebus start &&
service avahi-daemon start
```

**この時点で既にカーネルのアップデートがかかっているのでいったんシステムを再起動します。**

```
reboot
```

### Identity Service(Keystone)のインストール

```
yum install -y openstack-keystone
```

設定については基本的にコメント外すだけですが、直す箇所が多いのでパッチで見たほうが便利です。  
現時点での私の設定変更内容がgithubにあがっています。  
もちろんGrizzlyのコンポーネント設定ファイルのデフォルトが変更になった場合等はうまくパッチがあたらないので  
参考程度にとらえてください。  
git形式のパッチなのでp0ではなくp1で当ててください。  
不安な人は `--dry-run` してみてください。  
最後の最終行の改行で引っかからなければいいんだけど・・・。

```
cd /etc/keystone/
パッチファイルをGitHubからダウンロード
wget https://raw.github.com/wnoguchi/doc/master/OpenStack/install/1-CentOS6.4-x64-FirstTimeInstall/config/keystone-config.patch
正常にパッチがあたるかdry-runで確認
patch -p1 --dry-run < keystone-config.patch
うまくいけばパッチ適用。
patch -p1 < keystone-config.patch
```

ちなみに、パッチの内容は以下のとおりです。

```diff
diff --git a/keystone.conf b/keystone.conf
index 2375b32..3335b43 100644
--- a/keystone.conf
+++ b/keystone.conf
@@ -1,16 +1,16 @@
 [DEFAULT]
 log_file = /var/log/keystone/keystone.log
 # A "shared secret" between keystone and other openstack services
-# admin_token = ADMIN
+admin_token = ADMIN
 
 # The IP address of the network interface to listen on
-# bind_host = 0.0.0.0
+bind_host = 0.0.0.0
 
 # The port number which the public service listens on
-# public_port = 5000
+public_port = 5000
 
 # The port number which the public admin listens on
-# admin_port = 35357
+admin_port = 35357
 
 # The base endpoint URLs for keystone that are advertised to clients
 # (NOTE: this does NOT affect how keystone listens for connections)
@@ -18,10 +18,10 @@ log_file = /var/log/keystone/keystone.log
 # admin_endpoint = http://localhost:%(admin_port)d/
 
 # The port number which the OpenStack Compute service listens on
-# compute_port = 8774
+compute_port = 8774
 
 # Path to your policy definition containing identity actions
-# policy_file = policy.json
+policy_file = policy.json
 
 # Rule to check if no matching policy definition is found
 # FIXME(dolph): This should really be defined as [policy] default_rule
@@ -38,10 +38,10 @@ log_file = /var/log/keystone/keystone.log
 # === Logging Options ===
 # Print debugging output
 # (includes plaintext request logging, potentially including passwords)
-# debug = False
+debug = True
 
 # Print more verbose output
-# verbose = False
+verbose = True
 
 # Name of log file to output to. If not set, logging will go to stdout.
 # log_file = keystone.log
@@ -75,12 +75,13 @@ log_file = /var/log/keystone/keystone.log
 # onready = keystone.common.systemd
 
 [sql]
-connection = mysql://keystone:keystone@localhost/keystone
+#connection = mysql://keystone:keystone@localhost/keystone
 # The SQLAlchemy connection string used to connect to the database
 # connection = sqlite:///keystone.db
+connection = mysql://keystone:password@stack01/keystone?charset=utf8
 
 # the timeout before idle sql connections are reaped
-# idle_timeout = 200
+idle_timeout = 200
 
 [identity]
 driver = keystone.identity.backends.sql.Identity
@@ -119,7 +120,7 @@ driver = keystone.token.backends.sql.Token
 # expiration = 86400
 
 [policy]
-# driver = keystone.policy.backends.sql.Policy
+#driver = keystone.policy.backends.sql.Policy
 
 [ec2]
 driver = keystone.contrib.ec2.backends.sql.Ec2
@@ -133,7 +134,7 @@ driver = keystone.contrib.ec2.backends.sql.Ec2
 #cert_required = True
 
 [signing]
-#token_format = PKI
+token_format = UUID
 #certfile = /etc/keystone/ssl/certs/signing_cert.pem
 #keyfile = /etc/keystone/ssl/private/signing_key.pem
 #ca_certs = /etc/keystone/ssl/certs/ca.pem
@@ -221,8 +222,10 @@ driver = keystone.contrib.ec2.backends.sql.Ec2
 
 [auth]
 methods = password,token
-password = keystone.auth.plugins.password.Password
-token = keystone.auth.plugins.token.Token
+password = keystone.auth.methods.password.Password
+token = keystone.auth.methods.token.Token
+#password = keystone.auth.plugins.password.Password
+#token = keystone.auth.plugins.token.Token
 
 [filter:debug]
 paste.filter_factory = keystone.common.wsgi:Debug.factory
```

#### MySQL関係の設定

```
MYSQL_PASS_KEYSTONE=password
NOVA_CONTOLLER_HOSTNAME=stack01
mysql -uroot -pnova -e "drop database if exists keystone;"
mysql -uroot -pnova -e "create database keystone character set utf8;"
mysql -uroot -pnova -e "grant all privileges on keystone.* to 'keystone'@'%' identified by '$MYSQL_PASS_KEYSTONE';"
mysql -uroot -pnova -e "grant all privileges on keystone.* to 'keystone'@'localhost' identified by '$MYSQL_PASS_KEYSTONE';"
mysql -uroot -pnova -e "grant all privileges on keystone.* to 'keystone'@'$NOVA_CONTOLLER_HOSTNAME' identified by '$MYSQL_PASS_KEYSTONE';"
keystone-manage db_sync
```

```
chown keystone:keystone /var/log/keystone -R &&
service openstack-keystone restart &&
chkconfig openstack-keystone on
service openstack-keystone status

keystone (pid  6358) を実行中...
```

#### データ投入テスト

```
cd /usr/local/src ; cp -a /usr/share/openstack-keystone/sample_data.sh .

export CONTROLLER_HOST=stack01
export SERVICE_ENDPOINT=http://$CONTROLLER_HOST:35357/v2.0
sed -i "s/localhost/$CONTROLLER_HOST/" /usr/local/src/sample_data.sh
export ENABLE_ENDPOINTS=yes
/usr/local/src/sample_data.sh

-------------+-------------------------------------+
|   Property  |                Value                |
+-------------+-------------------------------------+
|   adminurl  |  http://stack01:$(admin_port)s/v2.0 |
|      id     |   7ceb3a26dfe44c57bdd465986f41beee  |
| internalurl | http://stack01:$(public_port)s/v2.0 |
|  publicurl  | http://stack01:$(public_port)s/v2.0 |
|    region   |              RegionOne              |
|  service_id |   b5432d5af04944d8b8ecf76d17f7da7d  |
+-------------+-------------------------------------+
+-------------+----------------------------------------------------+
|   Property  |                       Value                        |
+-------------+----------------------------------------------------+
|   adminurl  | http://stack01:$(compute_port)s/v1.1/$(tenant_id)s |
|      id     |          3321f331d25b48bfbe78db2465ecd178          |
| internalurl | http://stack01:$(compute_port)s/v1.1/$(tenant_id)s |
|  publicurl  | http://stack01:$(compute_port)s/v1.1/$(tenant_id)s |
|    region   |                     RegionOne                      |
|  service_id |          fcd9f77749ad41b6a1c651efade50f18          |
+-------------+----------------------------------------------------+
+-------------+--------------------------------------+
|   Property  |                Value                 |
+-------------+--------------------------------------+
|   adminurl  | http://stack01:8776/v1/$(tenant_id)s |
|      id     |   c9c73f1c82f945e09e30eb13c37141bf   |
| internalurl | http://stack01:8776/v1/$(tenant_id)s |
|  publicurl  | http://stack01:8776/v1/$(tenant_id)s |
|    region   |              RegionOne               |
|  service_id |   c047229f68a9422a82ce05031680b1f3   |
+-------------+--------------------------------------+
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
|   adminurl  |       http://stack01:9292        |
|      id     | a7426193194b4279a90bd0cb6ba4de32 |
| internalurl |       http://stack01:9292        |
|  publicurl  |       http://stack01:9292        |
|    region   |            RegionOne             |
|  service_id | 154112af665f4d978cd32fd4464d68c1 |
+-------------+----------------------------------+
+-------------+------------------------------------+
|   Property  |               Value                |
+-------------+------------------------------------+
|   adminurl  | http://stack01:8773/services/Admin |
|      id     |  73cf69449b194d02928bcffd976c8460  |
| internalurl | http://stack01:8773/services/Cloud |
|  publicurl  | http://stack01:8773/services/Cloud |
|    region   |             RegionOne              |
|  service_id |  f1ddbb5a05c046feaf12e6eeee846bd3  |
+-------------+------------------------------------+
+-------------+-------------------------------------------+
|   Property  |                   Value                   |
+-------------+-------------------------------------------+
|   adminurl  |           http://stack01:8888/v1          |
|      id     |      66cd05cabc654f77bf694749f2a74ab6     |
| internalurl | http://stack01:8888/v1/AUTH_$(tenant_id)s |
|  publicurl  | http://stack01:8888/v1/AUTH_$(tenant_id)s |
|    region   |                 RegionOne                 |
|  service_id |      7d3ee110b7cb4a6581770282c7d7ca12     |
+-------------+-------------------------------------------+

[root@stack01 src]# export OS_SERVICE_TOKEN=ADMIN
[root@stack01 src]# export OS_SERVICE_ENDPOINT=http://stack01:35357/v2.0/


[root@stack01 src]# keystone tenant-list
+----------------------------------+---------+---------+
|                id                |   name  | enabled |
+----------------------------------+---------+---------+
| 439df062c81244a1af4f977f1450990c |   demo  |   True  |
| 8949bace05894bc4bfac9f6b1d93fdf3 | service |   True  |
+----------------------------------+---------+---------+

[root@stack01 src]# keystone user-list
+----------------------------------+--------+---------+-------+
|                id                |  name  | enabled | email |
+----------------------------------+--------+---------+-------+
| e9dafc41e0b747dd8947bb573a4a617a | admin  |   True  |       |
| e3930fbde3064bd497280bacf52f86ea |  ec2   |   True  |       |
| 159911ddec4f4c8da26f227f6b5b2886 | glance |   True  |       |
| 19748c3737404b12b22a54ae7c66e31c |  nova  |   True  |       |
| 81d5b89a3e7a42bb80b7081ad88e9aae | swift  |   True  |       |
+----------------------------------+--------+---------+-------+

[root@wnoguchi src]# keystone role-list
+----------------------------------+----------+
|                id                |   name   |
+----------------------------------+----------+
| 9fe2ff9ee4384b1894a90878d3e92bab | _member_ |
| 0f930c30105848aea75c599954a3de60 |  admin   |
+----------------------------------+----------+

[root@wnoguchi src]# keystone service-list
+----------------------------------+----------+--------------+---------------------------+
|                id                |   name   |     type     |        description        |
+----------------------------------+----------+--------------+---------------------------+
| f1ddbb5a05c046feaf12e6eeee846bd3 |   ec2    |     ec2      |  EC2 Compatibility Layer  |
| 154112af665f4d978cd32fd4464d68c1 |  glance  |    image     |    Glance Image Service   |
| b5432d5af04944d8b8ecf76d17f7da7d | keystone |   identity   | Keystone Identity Service |
| fcd9f77749ad41b6a1c651efade50f18 |   nova   |   compute    |    Nova Compute Service   |
| 7d3ee110b7cb4a6581770282c7d7ca12 |  swift   | object-store |       Swift Service       |
| c047229f68a9422a82ce05031680b1f3 |  volume  |    volume    |    Nova Volume Service    |
+----------------------------------+----------+--------------+---------------------------+

[root@wnoguchi src]# keystone endpoint-list
+----------------------------------+-----------+----------------------------------------------------+----------------------------------------------------+----------------------------------------------------+----------------------------------+
|                id                |   region  |                     publicurl                      |                    internalurl                     |                      adminurl                      |            service_id            |
+----------------------------------+-----------+----------------------------------------------------+----------------------------------------------------+----------------------------------------------------+----------------------------------+
| 3321f331d25b48bfbe78db2465ecd178 | RegionOne | http://stack01:$(compute_port)s/v1.1/$(tenant_id)s | http://stack01:$(compute_port)s/v1.1/$(tenant_id)s | http://stack01:$(compute_port)s/v1.1/$(tenant_id)s | fcd9f77749ad41b6a1c651efade50f18 |
| 66cd05cabc654f77bf694749f2a74ab6 | RegionOne |     http://stack01:8888/v1/AUTH_$(tenant_id)s      |     http://stack01:8888/v1/AUTH_$(tenant_id)s      |               http://stack01:8888/v1               | 7d3ee110b7cb4a6581770282c7d7ca12 |
| 73cf69449b194d02928bcffd976c8460 | RegionOne |         http://stack01:8773/services/Cloud         |         http://stack01:8773/services/Cloud         |         http://stack01:8773/services/Admin         | f1ddbb5a05c046feaf12e6eeee846bd3 |
| 7ceb3a26dfe44c57bdd465986f41beee | RegionOne |        http://stack01:$(public_port)s/v2.0         |        http://stack01:$(public_port)s/v2.0         |         http://stack01:$(admin_port)s/v2.0         | b5432d5af04944d8b8ecf76d17f7da7d |
| a7426193194b4279a90bd0cb6ba4de32 | RegionOne |                http://stack01:9292                 |                http://stack01:9292                 |                http://stack01:9292                 | 154112af665f4d978cd32fd4464d68c1 |
| c9c73f1c82f945e09e30eb13c37141bf | RegionOne |        http://stack01:8776/v1/$(tenant_id)s        |        http://stack01:8776/v1/$(tenant_id)s        |        http://stack01:8776/v1/$(tenant_id)s        | c047229f68a9422a82ce05031680b1f3 |
+----------------------------------+-----------+----------------------------------------------------+----------------------------------------------------+----------------------------------------------------+----------------------------------+
```

### Image delivery and registration (Glance)のインストール

```
yum -y install openstack-glance
```

設定します。

```
cd /etc/glance/
パッチファイルをGitHubからダウンロード
wget https://raw.github.com/wnoguchi/doc/master/OpenStack/install/1-CentOS6.4-x64-FirstTimeInstall/config/glance-config.patch
正常にパッチがあたるかdry-runで確認
patch -p1 --dry-run < glance-config.patch
うまくいけばパッチ適用。
patch -p1 < glance-config.patch
```

ちなみに、パッチの内容は以下のとおりです。

```diff
diff --git a/glance-api.conf b/glance-api.conf
index 56e9496..9ee2714 100644
--- a/glance-api.conf
+++ b/glance-api.conf
@@ -46,7 +46,8 @@ backlog = 4096
 # SQLAlchemy connection string for the reference implementation
 # registry server. Any valid SQLAlchemy connection string is fine.
 # See: http://www.sqlalchemy.org/docs/05/reference/sqlalchemy/connections.html#sqlalchemy.create_engine
-sql_connection = mysql://glance:glance@localhost/glance
+#sql_connection = mysql://glance:glance@stack01/glance
+sql_connection = mysql://glance:password@stack01/glance?charset=utf8
 
 # Period in seconds after which SQLAlchemy should reestablish its connection
 # to the database.
@@ -163,7 +164,7 @@ notifier_strategy = noop
 
 # Configuration options if sending notifications via rabbitmq (these are
 # the defaults)
-rabbit_host = localhost
+rabbit_host = stack01
 rabbit_port = 5672
 rabbit_use_ssl = false
 rabbit_userid = guest
@@ -177,7 +178,7 @@ rabbit_durable_queues = False
 # the defaults)
 qpid_notification_exchange = glance
 qpid_notification_topic = notifications
-qpid_host = localhost
+qpid_host = stack01
 qpid_port = 5672
 qpid_username =
 qpid_password =
@@ -207,8 +208,8 @@ swift_store_auth_version = 2
 # Address where the Swift authentication service lives
 # Valid schemes are 'http://' and 'https://'
 # If no scheme specified,  default to 'https://'
-# For swauth, use something like '127.0.0.1:8080/v1.0/'
-swift_store_auth_address = 127.0.0.1:5000/v2.0/
+# For swauth, use something like 'stack01:8080/v1.0/'
+swift_store_auth_address = stack01:5000/v2.0/
 
 # User to authenticate against the Swift authentication service
 # If you use Swift authentication service, set it to 'account':'user'
@@ -271,7 +272,7 @@ swift_enable_snet = False
 # Address where the S3 authentication service lives
 # Valid schemes are 'http://' and 'https://'
 # If no scheme specified,  default to 'http://'
-s3_store_host = 127.0.0.1:8080/v1.0/
+s3_store_host = stack01:8080/v1.0/
 
 # User to authenticate against the S3 authentication service
 s3_store_access_key = <20-char AWS access key>
@@ -338,12 +339,15 @@ scrubber_datadir = /var/lib/glance/scrubber
 image_cache_dir = /var/lib/glance/image-cache/
 
 [keystone_authtoken]
-auth_host = 127.0.0.1
+auth_host = stack01
 auth_port = 35357
 auth_protocol = http
-admin_tenant_name = %SERVICE_TENANT_NAME%
-admin_user = %SERVICE_USER%
-admin_password = %SERVICE_PASSWORD%
+#admin_tenant_name = %SERVICE_TENANT_NAME%
+#admin_user = %SERVICE_USER%
+#admin_password = %SERVICE_PASSWORD%
+admin_tenant_name = service
+admin_user = glance
+admin_password = glance
 
 [paste_deploy]
 # Name of the paste configuration file that defines the available pipelines
@@ -354,3 +358,5 @@ admin_password = %SERVICE_PASSWORD%
 # [pipeline:glance-api-keystone], you would configure the flavor below
 # as 'keystone'.
 #flavor=
+flavor = keystone
+
diff --git a/glance-registry.conf b/glance-registry.conf
index 193a402..5d49d76 100644
--- a/glance-registry.conf
+++ b/glance-registry.conf
@@ -25,7 +25,8 @@ backlog = 4096
 # SQLAlchemy connection string for the reference implementation
 # registry server. Any valid SQLAlchemy connection string is fine.
 # See: http://www.sqlalchemy.org/docs/05/reference/sqlalchemy/connections.html#sqlalchemy.create_engine
-sql_connection = mysql://glance:glance@localhost/glance
+#sql_connection = mysql://glance:glance@stack01/glance
+sql_connection = mysql://glance:password@stack01/glance?charset=utf8
 
 # Period in seconds after which SQLAlchemy should reestablish its connection
 # to the database.
@@ -72,12 +73,15 @@ limit_param_default = 25
 #ca_file = /path/to/cafile
 
 [keystone_authtoken]
-auth_host = 127.0.0.1
+auth_host = stack01
 auth_port = 35357
 auth_protocol = http
-admin_tenant_name = %SERVICE_TENANT_NAME%
-admin_user = %SERVICE_USER%
-admin_password = %SERVICE_PASSWORD%
+#admin_tenant_name = %SERVICE_TENANT_NAME%
+#admin_user = %SERVICE_USER%
+#admin_password = %SERVICE_PASSWORD%
+admin_tenant_name = service
+admin_user = glance
+admin_password = glance
 
 [paste_deploy]
 # Name of the paste configuration file that defines the available pipelines
@@ -88,3 +92,5 @@ admin_password = %SERVICE_PASSWORD%
 # [pipeline:glance-registry-keystone], you would configure the flavor below
 # as 'keystone'.
 #flavor=
+flavor = keystone
+
```

#### MySQLにGlanceのデータベースを構築

```
MYSQL_PASS_GLANCE=password
NOVA_CONTOLLER_HOSTNAME=stack01
mysql -u root -pnova -e "drop database if exists glance;"
mysql -u root -pnova -e "create database glance character set utf8;"
mysql -u root -pnova -e "grant all privileges on glance.* to 'glance'@'%' identified by '$MYSQL_PASS_GLANCE';"
mysql -u root -pnova -e "grant all privileges on glance.* to 'glance'@'localhost' identified by '$MYSQL_PASS_GLANCE';"
mysql -u root -pnova -e "grant all privileges on glance.* to 'glance'@'$NOVA_CONTOLLER_HOSTNAME' identified by '$MYSQL_PASS_GLANCE';"
sudo glance-manage db_sync
```

#### Glanceのサービス起動・自動起動設定。

```
chown glance:glance /var/log/glance -R
for i in api registry
do
  service openstack-glance-$i restart
done
for i in api registry
do
  chkconfig openstack-glance-$i on
done
```

#### サンプルのイメージをGlanceに登録してみる

##### 環境変数設定

```
export OS_USERNAME=admin
export OS_PASSWORD=secrete
export OS_TENANT_NAME=demo
export OS_AUTH_URL=http://stack01:35357/v2.0/
```

##### イメージのダウンロード

```
mkdir -p /opt/virt/fedora17 ; cd /opt/virt/fedora17
curl -O http://berrange.fedorapeople.org/images/2012-11-15/f17-x86_64-openstack-sda.qcow2
```

ダウンロードしたAMIイメージをGlanceに登録します。  
拡張子から見るにqcow2形式のイメージみたいです。  
どうやらAWSとかのクラウドコンピューティングインフラではqcow2でイメージを圧縮して管理して、  
永続化データは抽象化したストレージをブロックストレージ（EBSに相当）として切り出してiSCSIで  
仮想マシンのインスタンス（EC2相当）にアタッチするようです。  
これならインスタンス稼働中にイメージやブロックストレージのスナップショットをとってバックアップを飛ばすといったことができますね。

```
glance image-create --name="f17-jeos" --is-public=true --disk-format=qcow2 --container-format=ovf < /opt/virt/fedora17/f17-x86_64-openstack-sda.qcow2

+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 1f104b5667768964d5df8c4ad1d7cd27     |
| container_format | ovf                                  |
| created_at       | 2013-08-05T09:33:23                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | dd8143e2-e54b-4908-b56e-c76f1b5c14ac |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | f17-jeos                             |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 251985920                            |
| status           | active                               |
| updated_at       | 2013-08-05T09:33:24                  |
+------------------+--------------------------------------+

[root@wnoguchi glance]# glance image-list
+--------------------------------------+----------+-------------+------------------+-----------+--------+
| ID                                   | Name     | Disk Format | Container Format | Size      | Status |
+--------------------------------------+----------+-------------+------------------+-----------+--------+
| dd8143e2-e54b-4908-b56e-c76f1b5c14ac | f17-jeos | qcow2       | ovf              | 251985920 | active |
+--------------------------------------+----------+-------------+------------------+-----------+--------+

[root@wnoguchi glance]# glance image-show dd8143e2-e54b-4908-b56e-c76f1b5c14ac
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 1f104b5667768964d5df8c4ad1d7cd27     |
| container_format | ovf                                  |
| created_at       | 2013-08-05T09:33:23                  |
| deleted          | False                                |
| disk_format      | qcow2                                |
| id               | dd8143e2-e54b-4908-b56e-c76f1b5c14ac |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | f17-jeos                             |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 251985920                            |
| status           | active                               |
| updated_at       | 2013-08-05T09:33:24                  |
+------------------+--------------------------------------+
```

どこかで以下のようなメッセージを受け取ったのですが、どこで出てきたのか失念してしまったので、  
とりあえず貼っておきます。無視してください。

```
NotAuthenticated: 401 Unauthorized
This server could not verify that you are authorized to access the document you requested. Either you supplied the wrong credentials (e.g., bad password), or your browser does not understand how to supply the credentials required.
 Authentication required  
 (HTTP 500)
```

### Volume Service(Cinder)のインストール

AWSのEBSに相当するコンポーネントです。  
以前のバージョンではNovaの中の一つになってましたが最近分離されました。

ここではストレージにLVMを使用します。  
事前に `cinder-volumes` の名前のボリュームグループが定義されているものとします。

```
yum -y install openstack-cinder
```

#### Cinderの設定

hagiさん [3.2.5. Volume Service(Cinder)のインストール — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/cinder_install.html) の設定だと動くのですが、
wnoguchiのクソパッチだと動かないです。  
見た目動いているように見えますけど

```
[root@stack01 cinder]# cinder list
ERROR: Malformed request url
```

な感じのエラーが出ます。
とりあえずあとで勉強のために残しています。  
ここではhagiさんのパッチをあてます。

```
cd /etc/cinder/
パッチファイルをGitHubからダウンロード
wget https://raw.github.com/wnoguchi/doc/master/OpenStack/install/1-CentOS6.4-x64-FirstTimeInstall/config/cinder-config-hagi.patch
正常にパッチがあたるかdry-runで確認
patch -p1 --dry-run < cinder-config-hagi.patch
うまくいけばパッチ適用。
patch -p1 < cinder-config-hagi.patch
```

ちなみに、パッチの内容は以下のとおりです。

```diff
diff --git a/api-paste.ini b/api-paste.ini
index 1838881..045cfe9 100644
--- a/api-paste.ini
+++ b/api-paste.ini
@@ -51,6 +51,13 @@ paste.filter_factory = cinder.api.middleware.auth:CinderKeystoneContext.factory
 [filter:authtoken]
 paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
 service_protocol = http
-service_host = 127.0.0.1
+service_host = stack01
 service_port = 5000
+auth_host = stack01
+auth_port = 35357
+auth_protocol = http
+admin_tenant_name = service
+admin_user = nova
+admin_password = nova
 signing_dir = /var/lib/cinder
+
diff --git a/cinder.conf b/cinder.conf
index 89105a6..aa739f5 100644
--- a/cinder.conf
+++ b/cinder.conf
@@ -1,18 +1,34 @@
 [DEFAULT]
-logdir = /var/log/cinder
+#misc
+verbose = True
+auth_strategy = keystone
+rootwrap_config = /etc/cinder/rootwrap.conf
+api_paste_config = /etc/cinder/api-paste.ini
+auth_strategy = keystone
 state_path = /var/lib/cinder
-lock_path = /var/lib/cinder/tmp
-volumes_dir = /etc/cinder/volumes
-iscsi_helper = tgtadm
-sql_connection = mysql://cinder:cinder@localhost/cinder
+volumes_dir = /var/lib/cinder/volumes
+
+#log
+log_file=cinder.log
+log_dir=/var/log/cinder
+
+#osapi
+osapi_volume_extension = cinder.api.openstack.volume.contrib.standard_extensions
+
+#qpid
 rpc_backend = cinder.openstack.common.rpc.impl_qpid
-rootwrap_config = /etc/cinder/rootwrap.conf
 
-[keystone_authtoken]
-admin_tenant_name = %SERVICE_TENANT_NAME%
-admin_user = %SERVICE_USER%
-admin_password = %SERVICE_PASSWORD%
-auth_host = 127.0.0.1
-auth_port = 35357
-auth_protocol = http
-signing_dirname = /tmp/keystone-signing-cinder
+qpid_port = 5672
+#qpid_username =
+#qpid_password =
+
+#sql
+sql_connection = mysql://cinder:password@stack01/cinder?charset=utf8
+
+#volume
+volume_name_template = volume-%s
+volume_group = cinder-volumes
+
+#iscsi
+iscsi_helper = tgtadm
+

```

#### MySQLにCinderのデータベース作成

```
MYSQL_PASS_CINDER=password
NOVA_CONTOLLER_HOSTNAME=stack01
mysql -uroot -pnova -e "drop database if exists cinder;"
mysql -uroot -pnova -e "create database cinder character set utf8;"
mysql -uroot -pnova -e "grant all privileges on cinder.* to 'cinder'@'%' identified by '$MYSQL_PASS_CINDER';"
mysql -uroot -pnova -e "grant all privileges on cinder.* to 'cinder'@'localhost' identified by '$MYSQL_PASS_CINDER';"
mysql -uroot -pnova -e "grant all privileges on cinder.* to 'cinder'@'$NOVA_CONTOLLER_HOSTNAME' identified by '$MYSQL_PASS_CINDER';"
mysql -uroot -pnova -e "flush privileges;"
cinder-manage db sync
2013-08-13 23:28:08     INFO [migrate.versioning.api] 0 -> 1... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 1 -> 2... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 2 -> 3... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 3 -> 4... 
2013-08-13 23:28:08     INFO [004_volume_type_to_uuid] Created foreign key volume_type_extra_specs_ibfk_1
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 4 -> 5... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 5 -> 6... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 6 -> 7... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 7 -> 8... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
2013-08-13 23:28:08     INFO [migrate.versioning.api] 8 -> 9... 
2013-08-13 23:28:08     INFO [migrate.versioning.api] done
```

#### tgtの設定（iSCSIの設定）

```
sed -i "s@# include /etc/cinder/volumes/@include /etc/cinder/volumes/@" /etc/tgt/conf.d/cinder.conf
echo "include /etc/tgt/conf.d/*.conf" >> /etc/tgt/targets.conf
echo "include /var/lib/cinder/volumes/*" >> /etc/tgt/conf.d/cinder.conf
```

tgtサービス自動起動設定。

```
service tgtd restart &&
chkconfig tgtd on
```

#### Cinderサービス自動起動設定。

```
chown cinder:cinder /var/log/cinder/*
for i in volume api scheduler
do
  service openstack-cinder-$i restart
done
for i in volume api scheduler
do
  chkconfig openstack-cinder-$i on
done
```

#### Cinderでボリュームが作成できるかどうかテスト

```
[root@stack01 cinder]# cinder list
```

何も出力されないことを確認。  
実際に作成してみます。

```
[root@stack01 cinder]# cinder create --display_name cinder_test 1
+---------------------+--------------------------------------+
|       Property      |                Value                 |
+---------------------+--------------------------------------+
|     attachments     |                  []                  |
|  availability_zone  |                 nova                 |
|       bootable      |                false                 |
|      created_at     |      2013-08-13T14:29:30.753647      |
| display_description |                 None                 |
|     display_name    |             cinder_test              |
|          id         | 2723f4a4-7125-43e2-aa61-f03ffc0fc254 |
|       metadata      |                  {}                  |
|         size        |                  1                   |
|     snapshot_id     |                 None                 |
|     source_volid    |                 None                 |
|        status       |               creating               |
|     volume_type     |                 None                 |
+---------------------+--------------------------------------+
```

作成できたかどうかリストを確認します。

```
[root@stack01 cinder]# cinder list
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
| 2723f4a4-7125-43e2-aa61-f03ffc0fc254 | available | cinder_test  |  1   |     None    |  false   |             |
+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
```

ここで Status が error とかなる場合は LVM のボリュームグループ名の指定方法などに問題がある可能性があります。  
available になっているのが確認できたら削除します。  
実際のブロックストレージの作成はあとでWebインタフェース Horizon からやってみます。

```
[root@stack01 cinder]# cinder delete $(cinder list | grep cinder_test | awk '{print $2}')
```

削除されるまでちょっとだけ時間がかかります。

```
[root@stack01 cinder]# cinder list
```

### Cloud compute (Nova)のインストール

いよいよ計算機リソースを制御するコンポーネントをインストールします。  
一口にNovaと言ってもその中にめちゃくちゃサブコンポーネントが詰まってて複雑なんだそうです。

```
yum install -y openstack-nova
http://repos.fedorapeople.org/repos/openstack/openstack-grizzly/epel-6/repodata/61fe111b1f291be718a48d77219aef4c1171e05d10115a71da113b8f7e08301d-filelists.sqlite.bz2: [Errno 14] PYCURL ERROR 22 - "The requested URL returned error: 404 Not Found"
Trying other mirror.
Error: failure: repodata/61fe111b1f291be718a48d77219aef4c1171e05d10115a71da113b8f7e08301d-filelists.sqlite.bz2 from epel-openstack-grizzly: [Errno 256] No more mirrors to try.
 You could try using --skip-broken to work around the problem
 You could try running: rpm -Va --nofiles --nodigest
```

なんだか何回やってもこのエラー出ますね。なんなんでしょう。  
URL変わったのかな。  
リポジトリメタデータクリアする。

[yumリポジトリのキャッシュをクリアする - think-tの晴耕雨読](http://d.hatena.ne.jp/think-t/20130408/p1)

```
yum clean all
```

もう一回。

```
yum install -y openstack-nova
```

今度はうまくいった。

#### Novaの設定を行う

詳細はこちらを。

[3.2.6. Cloud compute (Nova)のインストール — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/nova_install.html)

ここではパッチをあてるだけにします。  
複雑なのであとで勉強。

```
cd /etc/nova/
パッチファイルをGitHubからダウンロード
wget https://raw.github.com/wnoguchi/doc/master/OpenStack/install/1-CentOS6.4-x64-FirstTimeInstall/config/nova-config.patch
正常にパッチがあたるかdry-runで確認
patch -p1 --dry-run < nova-config.patch
うまくいけばパッチ適用。
patch -p1 < nova-config.patch
```

#### MySQLにNovaデータベースを作成する

```
MYSQL_PASS_NOVA=password
NOVA_CONTOLLER_HOSTNAME=stack01
mysql -uroot -pnova -e "drop database if exists nova;"
mysql -uroot -pnova -e "create database nova;"
mysql -uroot -pnova -e "grant all privileges on nova.* to 'nova'@'%' identified by '$MYSQL_PASS_NOVA';"
mysql -uroot -pnova -e "grant all privileges on nova.* to 'nova'@'localhost' identified by '$MYSQL_PASS_NOVA';"
mysql -uroot -pnova -e "grant all privileges on nova.* to 'nova'@'$NOVA_CONTOLLER_HOSTNAME' identified by '$MYSQL_PASS_NOVA';"
nova-manage db sync
```

サービスめっちゃ多い。  
シェルスクリプトでぶん回します。hagiさんの。

```
for proc in api metadata-api cert network compute objectstore console scheduler consoleauth xvpvncproxy conductor
do
  service openstack-nova-$proc restart
done
for proc in api metadata-api cert network compute objectstore console scheduler consoleauth xvpvncproxy conductor
do
  chkconfig openstack-nova-$proc on
done
```

サービスがちゃんと起動しているか確認。

```
[root@stack01 nova]# nova-manage service list
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        stack01                              internal         enabled    :-)   2013-08-13 14:45:03
nova-console     stack01                              internal         enabled    :-)   2013-08-13 14:45:04
nova-scheduler   stack01                              internal         enabled    :-)   2013-08-13 14:45:04
nova-consoleauth stack01                              internal         enabled    :-)   2013-08-13 14:45:04
nova-conductor   stack01                              internal         enabled    :-)   2013-08-13 14:45:05
nova-network     stack01                              internal         enabled    :-)   2013-08-13 14:45:04
nova-compute     stack01                              nova             enabled    :-)   None      
```

### Dashboard (Horizon)のインストール

いよいよWebインタフェースをインストールします。  
Pythonで書いてあります。  
Djangoです。  
海外の方はほんとPythonが好きだなあって思います。

```
yum -y install openstack-dashboard openstack-nova-novncproxy &&
service openstack-nova-novncproxy start &&
chkconfig openstack-nova-novncproxy on
```

#### iptablesの設定及び再起動

iptablesの設定が必要な場合です。
iptablesにMySQL(3306)、noVNC(6080)、Apache(443,80)、Glance(9292)、Memcached(11211)、Qpid(5672)のポート開放を設定します。
ちょっとここまでは手が回ってません。

```
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 11211 -j ACCEPT' /etc/sysconfig/iptables
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 9292 -j ACCEPT' /etc/sysconfig/iptables
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 6080 -j ACCEPT' /etc/sysconfig/iptables
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 5672 -j ACCEPT' /etc/sysconfig/iptables
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT' /etc/sysconfig/iptables
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 443 -j ACCEPT' /etc/sysconfig/iptables
sed -i '10a-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT' /etc/sysconfig/iptables
service iptables restart
```

#### Apacheの再起動及び自動起動の設定

```
service httpd restart &&
service openstack-nova-novncproxy restart &&
chkconfig httpd on &&
chkconfig openstack-nova-novncproxy on
```

### OpenStack利用開始

やっと使える。。。

```
useradd stack
cat << KEYSTONERC | tee /home/stack/keystonerc > /dev/null
export OS_NO_CACHE=True
export ADMIN_TOKEN=ADMIN
export OS_USERNAME=admin
export OS_PASSWORD=secrete
export OS_TENANT_NAME=demo
export OS_AUTH_URL=http://stack01:35357/v2.0/
KEYSTONERC
chown stack:stack /home/stack/keystonerc
```

#### novaマネージネットワークの構築

自宅内LANでは `192.168.0.0/24` のネットワーク体系を使用しているのでこのネットワークに干渉しない設定ならOKです。
サブネットを25より広くしようとすると

```
2013-08-13 23:51:39.722 WARNING nova.network.manager [req-68dfba3f-88cd-4562-8bc7-d923856a86ee None None] Subnet(s) too large, defaulting to /25.  To override, specify network_size flag.
```

と怒られます。しょうがないのでデフォルトの25を指定します。

```
nova-manage network create \
--label nova_network1        \
--fixed_range_v4=10.0.0.0/25 \
--bridge_interface=eth0      \
--multi_host=T
2013-08-05 19:17:12.835 8906 INFO nova.network.driver [-] Loading network driver 'nova.network.linux_net'
```

作られたネットワークを見る。

```
[root@stack01 nova]# nova-manage network list
id      IPv4                    IPv6            start address   DNS1           DNS2             VlanID          project         uuid           
2       10.0.0.0/25             None            10.0.0.2        8.8.4.4        None             None            None            028454c6-5608-4db3-adab-9637f3524c06
```

#### フローティングIPの定義

使っていない社内LAN IPのレンジを切り出して割り当てましょう。

```
nova-manage floating create --ip_range=192.168.0.112/28
```

28ビットマスクなのでネットワークアドレスとブロードキャストアドレスを除いて
14IP使用できる計算になります。

```
[root@wnoguchi nova]# nova-manage floating list
None    192.168.0.113   None    nova    br100
None    192.168.0.114   None    nova    br100
None    192.168.0.115   None    nova    br100
None    192.168.0.116   None    nova    br100
None    192.168.0.117   None    nova    br100
None    192.168.0.118   None    nova    br100
None    192.168.0.119   None    nova    br100
None    192.168.0.120   None    nova    br100
None    192.168.0.121   None    nova    br100
None    192.168.0.122   None    nova    br100
None    192.168.0.123   None    nova    br100
None    192.168.0.124   None    nova    br100
None    192.168.0.125   None    nova    br100
None    192.168.0.126   None    nova    br100
```

#### キーペアの作成

このあたりで「あーこれAWSのマネジメントコンソールで似たようなこと見たことある」ってなりました。

```
cd /home/stack
[root@stack01 stack]# nova keypair-add mykey > mykey
[root@stack01 stack]# chown stack:stack mykey
[root@stack01 stack]# chmod 600 mykey
[root@stack01 stack]# nova keypair-list
+-------+-------------------------------------------------+
| Name  | Fingerprint                                     |
+-------+-------------------------------------------------+
| mykey | 16:cb:af:e8:87:bf:58:05:38:e0:55:98:dd:b4:d9:8a |
+-------+-------------------------------------------------+
```

#### セキュリティグループの設定

セキュリティグループの設定もEC2のインスタンスで既視感。  
だからOpenStackとEC2は互換性があるって言われるのか。

```
[root@stack01 stack]# nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| icmp        | -1        | -1      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
[root@stack01 stack]# nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| tcp         | 22        | 22      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
[root@stack01 stack]# nova secgroup-list
+---------+-------------+
| Name    | Description |
+---------+-------------+
| default | default     |
+---------+-------------+
```

icmpにポートとかいう概念はないから-1になってます。

```
[root@stack01 stack]# nova secgroup-list-rules default
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| icmp        | -1        | -1      | 0.0.0.0/0 |              |
| tcp         | 22        | 22      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
```

ホスト名はOpenStackをインストールしたサーバーのアドレスを指定。
`/dashboard` を入れないとApacheのデフォルトページが表示されるので注意が必要です。

http://192.168.0.10/dashboard

デフォルト

- ID: `admin`
- PASS: `secrete`

ここでNovaの設定ミスっててインスタンス動きませんでした。（[Novaが起動しない件](Novaが起動しない件.md)）

#### インスタンスを実際に作成してみる

**ここで実際にインスタンスを起動してみます。**

[3.2.8. OpenStackの利用 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/openstack_use.html)

が詳しいです。

これってけっこうタイミングが重要で、私最初起動のときインスタンス起動エラーになって
「なんでだー」って思ったりしたんです。

次の2パターンを繰り返したら直る時があるってことがわかりました。

##### iptablesを停止する

殺したはずなのにいつの間にか立ち上がってるときがあります。かんべん。  
まあ、iptablesちゃんと設定すればいいだけなんですけどね。。。

```
service iptables stop
chkconfig iptables off
```

#### ひたすらNovaのコンポーネントを再起動する

全部OKになるまでひたすらぶん回します。

```
for proc in api metadata-api cert network compute objectstore console scheduler consoleauth xvpvncproxy conductor
do
  service openstack-nova-$proc restart
done
for proc in api metadata-api cert network compute objectstore console scheduler consoleauth xvpvncproxy conductor
do
  chkconfig openstack-nova-$proc on
done

openstack-nova-api を停止中:                               [  OK  ]
openstack-nova-api を起動中:                               [  OK  ]
openstack-nova-metadata-api を停止中:                      [  OK  ]
openstack-nova-metadata-api を起動中:                      [  OK  ]
openstack-nova-cert を停止中:                              [  OK  ]
openstack-nova-cert を起動中:                              [  OK  ]
openstack-nova-network を停止中:                           [  OK  ]
openstack-nova-network を起動中:                           [  OK  ]
openstack-nova-compute を停止中:                           [  OK  ]
openstack-nova-compute を起動中:                           [  OK  ]
openstack-nova-objectstore を停止中:                       [  OK  ]
openstack-nova-objectstore を起動中:                       [  OK  ]
openstack-nova-console を停止中:                           [  OK  ]
openstack-nova-console を起動中:                           [  OK  ]
openstack-nova-scheduler を停止中:                         [  OK  ]
openstack-nova-scheduler を起動中:                         [  OK  ]
openstack-nova-consoleauth を停止中:                       [  OK  ]
openstack-nova-consoleauth を起動中:                       [  OK  ]
openstack-nova-xvpvncproxy を停止中:                       [  OK  ]
openstack-nova-xvpvncproxy を起動中:                       [  OK  ]
openstack-nova-conductor を停止中:                         [  OK  ]
openstack-nova-conductor を起動中:                         [  OK  ]

```

設定ファイル

#### OpenStack内部ネットワークにてpingの導通を確認。

```
[root@stack01 stack]# ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.220 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=0.175 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.206 ms
^C
--- 10.0.0.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2332ms
rtt min/avg/max/mdev = 0.175/0.200/0.220/0.022 ms
```

このままだと外からSSHできないのでフローティングIPあてました。

```
localhost:OpenStack noguchiwataru$ ping 192.168.0.114
PING 192.168.0.114 (192.168.0.114): 56 data bytes
64 bytes from 192.168.0.114: icmp_seq=0 ttl=63 time=1.839 ms
64 bytes from 192.168.0.114: icmp_seq=1 ttl=63 time=1.018 ms
64 bytes from 192.168.0.114: icmp_seq=2 ttl=63 time=0.849 ms
^C
--- 192.168.0.114 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.849/1.235/1.839/0.432 ms
```

ちゃんとKVMにインスタンス割り当てられてるかな。

```
[root@stack01 stack]# virsh list --all
 Id    名前                         状態
----------------------------------------------------
 1     instance-00000005              実行中
```

うまくいってる。
ためしにSSHしてみる。

```
localhost:OpenStack noguchiwataru$ ssh root@192.168.0.114
The authenticity of host '192.168.0.114 (192.168.0.114)' can't be established.
RSA key fingerprint is ab:8a:97:a4:bf:ab:cb:cd:18:d8:dc:85:0b:2c:39:b4.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.0.114' (RSA) to the list of known hosts.
Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

rootはそのままログイン出来ない。  
AWSでおなじみの `ec2-user` でさっき作った鍵をコピペして鍵指定してつないでみる。

```
localhost:OpenStack noguchiwataru$ vi ~/.ssh/openstack-default.key
localhost:OpenStack noguchiwataru$ chmod 600 ~/.ssh/openstack-default.key
localhost:OpenStack noguchiwataru$ ssh ec2-user@192.168.0.114 -i ~/.ssh/openstack-default.key
[ec2-user@f17-003 ~]$ ls
[ec2-user@f17-003 ~]$ ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.0.2  netmask 255.255.255.128  broadcast 10.0.0.127
        inet6 fe80::f816:3eff:fee0:6678  prefixlen 64  scopeid 0x20<link>
        ether fa:16:3e:e0:66:78  txqueuelen 1000  (Ethernet)
        RX packets 393  bytes 56621 (55.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 300  bytes 39705 (38.7 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 16436
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

外にもいける。
さっき作ったsshキーペアで `ec2-user` にログインできた。
`root` は現在のところパス無しでいけるみたい。

このあたりまでくるともうEC2とおんなじですね。
AWSは超便利版みたいな感じ。

リブートしてもいける。

ログイン画面  
![](img/login.png)

ログイン後  
![](img/dashboard.png)

イメージの一覧  
![](img/images.png)

インスタンス作成画面  
インスタンスのリソーステンプレートも定義できるみたい。  
![](img/create_instance.png)

インスタンスがアクティブになったことを確認する  
![](img/instance_active.png)

フローティングIPを割り当てた  
![](img/float_ip.png)

シリアルコンソールも使える  
![](img/serial.png)

なんというか、動いたときは感動したのですが、動かなかった時の絶望感は半端じゃ無いですね。。。  
コンポーネントがたくさんあってそれに加えて膨大なログが分散してるのでどこでエラーが発生したのか特定するのが大変。。。  
とりあえずそれっぽく使えるようになったけど、もっと勉強しなきゃだなあ・・・。

## 参考サイト

- [3.1. Openstack概要 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/openstack_summary.html)
- [3. Openstackインストール手順(Grizzly)CentOS6.4(パッケージ)編 —
オープンソースに関するドキュメント 1.1
documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/)
