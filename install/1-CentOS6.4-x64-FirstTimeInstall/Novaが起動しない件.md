# Novaが起動しない件

起動しない。。。

```
nova boot --flavor 1 --image dd8143e2-e54b-4908-b56e-c76f1b5c14ac f17_001 --key_name mykey

[root@wnoguchi stack]# nova image-list
+--------------------------------------+----------+--------+--------+
| ID                                   | Name     | Status | Server |
+--------------------------------------+----------+--------+--------+
| dd8143e2-e54b-4908-b56e-c76f1b5c14ac | f17-jeos | ACTIVE |        |
+--------------------------------------+----------+--------+--------+
[root@wnoguchi stack]# nova flavor-list
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public | extra_specs |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
| 1  | m1.tiny   | 512       | 0    | 0         |      | 1     | 1.0         | True      | {}          |
| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      | {}          |
| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      | {}          |
| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      | {}          |
| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      | {}          |
+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+-------------+
[root@wnoguchi stack]# passwd stack
ユーザー stack のパスワードを変更。
新しいパスワード:
新しいパスワードを再入力してください:
passwd: 全ての認証トークンが正しく更新できました。
[root@wnoguchi stack]# nova list
+--------------------------------------+--------------+--------+----------+
| ID                                   | Name         | Status | Networks |
+--------------------------------------+--------------+--------+----------+
| 0e34a951-3204-43d6-9d69-b8d97d482cfe | vmtest       | ERROR  |          |
| 12b86b96-bbcb-4b4f-8b96-4eed67ef6cec | wataru-no-vm | ERROR  |          |
+--------------------------------------+--------------+--------+----------+


nova boot --flavor 1 --image dd8143e2-e54b-4908-b56e-c76f1b5c14ac f17_001 --key_name mykey


+-------------------------------------+--------------------------------------+
| Property                            | Value                                |
+-------------------------------------+--------------------------------------+
| OS-EXT-STS:task_state               | scheduling                           |
| image                               | f17-jeos                             |
| OS-EXT-STS:vm_state                 | building                             |
| OS-EXT-SRV-ATTR:instance_name       | instance-00000003                    |
| flavor                              | m1.tiny                              |
| id                                  | e5cb7f4a-a752-499a-ae74-cb0357b2d95f |
| security_groups                     | [{u'name': u'default'}]              |
| user_id                             | e9dafc41e0b747dd8947bb573a4a617a     |
| OS-DCF:diskConfig                   | MANUAL                               |
| accessIPv4                          |                                      |
| accessIPv6                          |                                      |
| progress                            | 0                                    |
| OS-EXT-STS:power_state              | 0                                    |
| OS-EXT-AZ:availability_zone         | nova                                 |
| config_drive                        |                                      |
| status                              | BUILD                                |
| updated                             | 2013-08-05T10:43:56Z                 |
| hostId                              |                                      |
| OS-EXT-SRV-ATTR:host                | None                                 |
| key_name                            | mykey                                |
| OS-EXT-SRV-ATTR:hypervisor_hostname | None                                 |
| name                                | f17_001                              |
| adminPass                           | mepEtDjG8dL7                         |
| tenant_id                           | 439df062c81244a1af4f977f1450990c     |
| created                             | 2013-08-05T10:43:56Z                 |
| metadata                            | {}                                   |
+-------------------------------------+--------------------------------------+
```

```
for proc in api metadata-api cert network compute objectstore console scheduler consoleauth xvpvncproxy conductor
do
  service openstack-nova-$proc restart
done
```

ごにょごにょ。

```
[root@wnoguchi stack]# nova list
+--------------------------------------+--------------+--------+------------------------+
| ID                                   | Name         | Status | Networks               |
+--------------------------------------+--------------+--------+------------------------+
| e5cb7f4a-a752-499a-ae74-cb0357b2d95f | f17_001      | ERROR  |                        |
| 84e1c17b-6692-4b64-b12d-cf742c4d140b | f17_002      | ERROR  |                        |
| 18c186c4-9386-44ed-80ac-7a258a9e7f25 | f17_003      | BUILD  | nova_network1=10.0.0.2 |
| 0e34a951-3204-43d6-9d69-b8d97d482cfe | vmtest       | ERROR  |                        |
| 12b86b96-bbcb-4b4f-8b96-4eed67ef6cec | wataru-no-vm | ERROR  |                        |
+--------------------------------------+--------------+--------+------------------------+
```

ああ、やっと起動した。

```
vi /etc/nova/nova.conf
```

のIP設定が間違ってたのか。。。
詳しくはnova設定のパッチ見てください。
