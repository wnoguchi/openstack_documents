# Keystoneが起動しない件

ハマった。。。  
結論から言うとOpenStack GrizzlyのEPELリポジトリを正しく有効化していなかったからでした。

## 作業トレース

```
chown keystone:keystone /var/log/keystone -R
service openstack-keystone restart
chkconfig openstack-keystone on
service openstack-keystone status
keystone が停止していますが PID ファイルが残っています
```

[GrizzlyのKeystoneインストールについて - Google グループ](https://groups.google.com/forum/#!msg/openstack-ja/DO46zPQ9Wrw/mYkSBSw0UiwJ)

デバッグする。

```
[root@wnoguchi samba]# chsh -s /bin/bash keystone
keystone のシェルを変更します。
シェルを変更しました。
[root@wnoguchi samba]# su - keystone
-bash-4.1$ keystone-all
Traceback (most recent call last):
  File "/usr/bin/keystone-all", line 104, in <module>
    int(CONF.admin_port)))
  File "/usr/bin/keystone-all", line 34, in create_server
    app = deploy.loadapp('config:%s' % conf, name=name)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 247, in loadapp
    return loadobj(APP, uri, name=name, **kw)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 272, in loadobj
    return context.create()
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 710, in create
    return self.object_type.invoke(self)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 144, in invoke
    **context.local_conf)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/util.py", line 56, in fix_call
    val = callable(*args, **kw)
  File "/usr/lib/python2.6/site-packages/paste/urlmap.py", line 25, in urlmap_factory
    app = loader.get_app(app_name, global_conf=global_conf)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 350, in get_app
    name=name, global_conf=global_conf).create()
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 362, in app_context
    APP, name=name, global_conf=global_conf)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 450, in get_context
    global_additions=global_additions)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 562, in _pipeline_app_context
    for name in pipeline[:-1]]
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 458, in get_context
    section)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 517, in _context_from_explicit
    value = import_string(found_expr)
  File "/usr/lib/python2.6/site-packages/PasteDeploy-1.5.0-py2.6.egg/paste/deploy/loadwsgi.py", line 22, in import_string
    return pkg_resources.EntryPoint.parse("x=" + s).load(False)
  File "/usr/lib/python2.6/site-packages/pkg_resources.py", line 1948, in load
    entry = __import__(self.module_name, globals(),globals(), ['__name__'])
ImportError: No module named access
```

iptables止めてみる。

```
[root@stack01 samba]# service iptables stop
iptables: ファイアウォールルールを消去中:                  [  OK  ]
iptables: チェインをポリシー ACCEPT へ設定中nat mangle filt[  OK  ]
iptables: モジュールを取り外し中:                          [  OK  ]
[root@stack01 samba]# chkconfig iptables off
```

症状変わらず。
epel正しく設定してないのがいけなかった。

```
cat <<EOF >/etc/yum.repos.d/openstack-grizzly.repo
[epel-openstack-grizzly]
name=OpenStack Grizzly Repository for EPEL 6
baseurl=http://repos.fedorapeople.org/repos/openstack/openstack-grizzly/epel-$releasever/
enabled=1
skip_if_unavailable=1
gpgcheck=0
EOF
yum clean all
yum -y update
```

今度はpython-migrateでエラーを吐き出す。

- [Bug #1159038 “Crash on start with SQLalchemy 0.8” : Bugs : Keystone](https://bugs.launchpad.net/keystone/+bug/1159038)
- [GrizzlyのKeystoneインストールについて - Google グループ](https://groups.google.com/forum/#!msg/openstack-ja/DO46zPQ9Wrw/mYkSBSw0UiwJ)
- [SIOS "OSSよろず" ブログ出張所: Red Hat のコミュニティ版 OpenStack ディストリビューション 「RDO」を試す。](http://sios-oss.blogspot.jp/2013/05/red-hat-openstack-rdo.html)


```
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



```
[root@wnoguchi keystone]# mysql -uroot -pnova -e "create database keystone character set utf8;"
[root@wnoguchi keystone]# keystone-manage db_sync
[root@wnoguchi keystone]# chown keystone:keystone /var/log/keystone -R
[root@wnoguchi keystone]# service openstack-keystone restart
keystone を停止中:                                         [失敗]
keystone を起動中:                                         [  OK  ]
[root@wnoguchi keystone]# chkconfig openstack-keystone on
[root@wnoguchi keystone]# service openstack-keystone status
keystone (pid  6358) を実行中...
```
