# Cinderが動かない

よく見たらボリューム作成できてない。

```
2013-08-08 00:08:05     INFO [cinder.api.openstack.wsgi] http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes/detail returned with HTTP 200
2013-08-08 00:08:05     INFO [cinder.api.openstack.wsgi] POST http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes
2013-08-08 00:08:05    AUDIT [cinder.api.v1.volumes] Create volume of 50 GB
2013-08-08 00:08:05    AUDIT [cinder.api.v1.volumes] vol={'volume_metadata': [], 'availability_zone': 'nova', 'terminated_at': None, 'updated_at': None, 'snapshot_id': None, 'ec2_id': None, 'mountpoint': None, 'deleted_at': None, 'id': 'a15469cc-b17e-4baa-82df-179c5ef0d7fb', 'size': 50, 'user_id': u'e9dafc41e0b747dd8947bb573a4a617a', 'attach_time': None, 'display_description': u'', 'project_id': u'439df062c81244a1af4f977f1450990c', 'launched_at': None, 'scheduled_at': None, 'status': 'creating', 'volume_type_id': None, 'deleted': False, 'provider_location': None, 'host': None, 'source_volid': None, 'provider_auth': None, 'display_name': u'volvol', 'instance_uuid': None, 'created_at': datetime.datetime(2013, 8, 7, 15, 8, 5, 552787), 'attach_status': 'detached', 'volume_type': None, 'metadata': {}}
2013-08-08 00:08:05     INFO [cinder.api.openstack.wsgi] http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes returned with HTTP 200
2013-08-08 00:08:05    ERROR [cinder.scheduler.manager] Failed to schedule_create_volume: No valid host was found. 
2013-08-08 00:08:05     INFO [cinder.api.openstack.wsgi] GET http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes/detail
2013-08-08 00:08:05    AUDIT [cinder.api.v1.volumes] vol=<cinder.db.sqlalchemy.models.Volume object at 0x37d6b10>
2013-08-08 00:08:05    AUDIT [cinder.api.v1.volumes] vol=<cinder.db.sqlalchemy.models.Volume object at 0x37d6cd0>
2013-08-08 00:08:05     INFO [cinder.api.openstack.wsgi] http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes/detail returned with HTTP 200
```
```
[root@stack01 cinder]# for i in volume api scheduler; do   service openstack-cinder-$i restart; done
openstack-cinder-volume を停止中:                          [失敗]
openstack-cinder-volume を起動中:                          [  OK  ]
openstack-cinder-api を停止中:                             [  OK  ]
openstack-cinder-api を起動中:                             [  OK  ]
openstack-cinder-scheduler を停止中:                       [  OK  ]
openstack-cinder-scheduler を起動中:                       [  OK  ]
```

[OpenStack 2012.2で追加された新機能「Cinder」を使う - さくらのナレッジ](http://knowledge.sakura.ad.jp/tech/119/)

> バックエンドストレージとしてLVMを使用する場合、volume_groupおよびvolume_name_template項目で使用するLVMのボリュームグループ名を指定しておく。デフォルトでは「cinder-volume」となっているが、別のものを指定することも可能だ。

```
volume_group = cinder-volumes
```

```
[root@stack01 cinder]# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "vg_wnoguchi" using metadata type lvm2
```

```
vi /etc/cinder/cinder.conf
volume_group = cinder-volumes
↓
volume_group = vg_wnoguchi
```

もいっちょ

```
[root@stack01 cinder]# for i in volume api scheduler; do   service openstack-cinder-$i restart; done
openstack-cinder-volume を停止中:                          [失敗]
openstack-cinder-volume を起動中:                          [  OK  ]
openstack-cinder-api を停止中:                             [  OK  ]
openstack-cinder-api を起動中:                             [  OK  ]
openstack-cinder-scheduler を停止中:                       [  OK  ]
openstack-cinder-scheduler を起動中:                       [  OK  ]
[root@stack01 cinder]# for i in volume api scheduler; do   service openstack-cinder-$i restart; done
openstack-cinder-volume を停止中:                          [  OK  ]
openstack-cinder-volume を起動中:                          [  OK  ]
openstack-cinder-api を停止中:                             [  OK  ]
openstack-cinder-api を起動中:                             [  OK  ]
openstack-cinder-scheduler を停止中:                       [  OK  ]
openstack-cinder-scheduler を起動中:                       [  OK  ]
```

おー！

```
cinder create --display_name cinder_test 3
+---------------------+--------------------------------------+
|       Property      |                Value                 |
+---------------------+--------------------------------------+
|     attachments     |                  []                  |
|  availability_zone  |                 nova                 |
|       bootable      |                false                 |
|      created_at     |      2013-08-07T15:30:21.480541      |
| display_description |                 None                 |
|     display_name    |             cinder_test              |
|          id         | 402e25e0-eae5-4256-996c-00e7680bd2af |
|       metadata      |                  {}                  |
|         size        |                  3                   |
|     snapshot_id     |                 None                 |
|     source_volid    |                 None                 |
|        status       |               creating               |
|     volume_type     |                 None                 |
+---------------------+--------------------------------------+
```

やっぱだめ、vgの空き領域ないって。そうだよね。

```
2013-08-08 00:33:30     INFO [cinder.api.openstack.wsgi] http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes returned with HTTP 20
0
2013-08-08 00:33:30  WARNING [cinder.scheduler.filters.capacity_filter] Insufficient free space for volume creation (requested / avail): 10/0.
0
2013-08-08 00:33:30    ERROR [cinder.scheduler.manager] Failed to schedule_create_volume: No valid host was found. 
2013-08-08 00:33:30     INFO [cinder.api.openstack.wsgi] GET http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes/detail
2013-08-08 00:33:30    AUDIT [cinder.api.v1.volumes] vol=<cinder.db.sqlalchemy.models.Volume object at 0x46dff10>
2013-08-08 00:33:30     INFO [cinder.api.openstack.wsgi] http://stack01:8776/v1/439df062c81244a1af4f977f1450990c/volumes/detail returned with HTTP 200
2013-08-08 00:33:49     INFO [cinder.volume.manager] Updating volume status


[root@stack01 cinder]# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "vg_wnoguchi" using metadata type lvm2
[root@stack01 cinder]# vgdisplay
  --- Volume group ---
  VG Name               vg_wnoguchi
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               931.02 GiB
  PE Size               4.00 MiB
  Total PE              238341
  Alloc PE / Size       238341 / 931.02 GiB
  Free  PE / Size       0 / 0   
  VG UUID               j4qO0Y-WUg9-srNC-J7cD-GcXA-QWEc-DOr2oq
```

しかたないから縮める

```   
[root@stack01 cinder]# lvscan
  ACTIVE            '/dev/vg_wnoguchi/lv_root' [927.14 GiB] inherit
  ACTIVE            '/dev/vg_wnoguchi/lv_swap' [3.88 GiB] inherit
```

むり、時間ない。あとで。
