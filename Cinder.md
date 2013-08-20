# Cinder

## 概要

OpenStackの構成コンポーネントのひとつであるCinderによるボリューム管理について。  
AWSのEBS(Elastic Block Storage)に相当するサービス。

## 使い方

デフォルトではLVMをボリュームの管理に使用する。
なのでCinderを使ったEBSライクなボリュームの動的接続するにはVGの空きがないといけない。
デフォルトだとボリュームグループの名前が

```
# /etc/cinder/cinder.conf
(snip)
volume_group = cinder-volumes
(snip)
```

となっているので、Cinder入れるところにvgの空きが無いとちゃんとcinderでボリューム切り出せなくてエラーになるよ。  
というか、そもそも openstack-cinder-volume 自体起動に失敗するので無理。

```
mkdir /mnt/ebs
[root@localhost ~]# mkfs.ext4 /dev/vdb 
mke2fs 1.42.3 (14-May-2012)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
3276800 inodes, 13107200 blocks
655360 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
400 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
	4096000, 7962624, 11239424

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done   

mount /dev/vdb /mnt/ebs/
[root@localhost ~]# mount /dev/vdb /mnt/ebs/
[root@localhost ~]# cd /mnt/ebs/
[root@localhost ebs]# ls
lost+found
[root@localhost ebs]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        237M     0  237M   0% /dev
tmpfs           246M     0  246M   0% /dev/shm
tmpfs           246M   23M  224M   9% /run
/dev/vda2       9.1G  945M  8.1G  11% /
tmpfs           246M     0  246M   0% /sys/fs/cgroup
tmpfs           246M     0  246M   0% /media
/dev/vdb         50G   52M   47G   1% /mnt/ebs

```

ちゃんとiSCSIでボリューム接続されてるね。  
そうか、稼働しているインスタンスを停止させずにどうやってストレージ接続するんだろうって思ってたけど、  
iSCSIで接続できるんだね。

## 参考サイト

- [OpenStackの新機能、Cinderを使う | SourceForge.JP Magazine](http://sourceforge.jp/magazine/13/04/03/090000)
- [OpenStack 2012.2で追加された新機能「Cinder」を使う - さくらのナレッジ](http://knowledge.sakura.ad.jp/tech/119/)
- [OpenStack Cinder Deep Dive Grizzly Release](https://wiki.openstack.org/w/images/3/3b/Cinder-grizzly-deep-dive-pub.pdf)
