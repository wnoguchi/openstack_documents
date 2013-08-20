# Glance

オブジェクトストアコンポーネントGlanceによるイメージの管理。

## 独自イメージの作成

僕はCentOS信者なのでCentOSのイメージを作成します。

```
yum install python-virtinst
```

```
mkdir -p /opt/virt/CentOS6.4
mkdir /mnt/ec2-ami
qemu-img create -f raw /opt/virt/CentOS6.4/CentOS6.4.img 10G
mke2fs -t ext4 -F -j /opt/virt/CentOS6.4/CentOS6.4.img

mount -o loop /opt/virt/CentOS6.4/CentOS6.4.img /mnt/ec2-ami
```

スペシャルデバイスファイルを作成する

```
mkdir /mnt/ec2-ami/dev
cd /mnt/ec2-ami/dev
[root@stack01 dev]# for i in console null zero ; do /sbin/MAKEDEV -d /mnt/ec2-ami/dev -x $i; done
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
MAKEDEV: mkdir: File exists
[root@stack01 dev]# ls
console  null  tty  zero
```

```
mkdir /mnt/ec2-ami/etc
cat << FSTAB_AMI | tee /mnt/ec2-ami/etc/fstab > /dev/null
LABEL=uec-rootfs / ext4 defaults 1 1
tmpfs /dev/shm tmpfs defaults 0 0
devpts /dev/pts devpts gid=5,mode=620 0 0
sysfs /sys sysfs defaults 0 0
proc /proc proc defaults 0 0
/dev/sda2 /mnt ext3 defaults 0 0
/dev/sda3 swap swap defaults 0 0
FSTAB_AMI
```

```
mkdir /mnt/ec2-ami/etc
cat << YUM_AMI | tee /mnt/ec2-ami/etc/yum-ami.conf > /dev/null
[main]
cachedir=/var/cache/yum
debuglevel=2
logfile=/var/log/yum.log
exclude=*-debuginfo
gpgcheck=0
obsoletes=1
reposdir=/dev/null

[base]
name=CentOS Linux - Base
baseurl=http://ftp.jaist.ac.jp/pub/Linux/CentOS/6.4/os/x86_64/
enabled=1
gpgcheck=0

[updates]
name=CentOS-6 - Updates
baseurl=http://ftp.jaist.ac.jp/pub/Linux/CentOS/6.4/updates/x86_64/
enabled=1
gpgcheck=0
YUM_AMI
```

```
mkdir /mnt/ec2-ami/proc
mount -t proc none /mnt/ec2-ami/proc
yum -c /mnt/ec2-ami/etc/yum-ami.conf --installroot=/mnt/ec2-ami -y groupinstall Core
yum -c /mnt/ec2-ami/etc/yum-ami.conf --installroot=/mnt/ec2-ami -y groupinstall Base
```

欲しいパッケージはここでいれとくとすごーくべんり

```
chroot /mnt/ec2-ami cp -p /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.org
chroot /mnt/ec2-ami sed -i 's/^#baseurl/baseurl/g' /etc/yum.repos.d/CentOS-Base.repo
chroot /mnt/ec2-ami sed -i 's/$releasever/6.4/g' /etc/yum.repos.d/CentOS-Base.repo
```

chrootにこんな使い方あるのか。しらんかった。
baseurlとかいじれると自社内の高速リポジトリとか指定できたりしてすごくべんり。

```
cat << NIC_AMI | tee /mnt/ec2-ami/etc/sysconfig/network-scripts/ifcfg-eth0 > /dev/null
DEVICE=eth0
BOOTPROTO=dhcp
ONBOOT=yes
TYPE=Ethernet
USERCTL=yes
PEERDNS=yes
IPV6INIT=no
NIC_AMI
```

```
cat << NETWORKING_AMI | tee /mnt/ec2-ami/etc/sysconfig/network > /dev/null
NETWORKING=yes
NETWORKING_AMI
```

```
cat << DNS_AMI | tee /mnt/ec2-ami/etc/resolv.conf > /dev/null
nameserver 8.8.8.8
nameserver 8.8.4.4
DNS_AMI
```

```
[root@stack01 dev]# cd ..
[root@stack01 ec2-ami]# ls -l
合計 104
dr-xr-xr-x.   2 root root  4096  8月  7 22:06 2013 bin
dr-xr-xr-x.   4 root root  4096  8月  7 22:07 2013 boot
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 dev
drwxr-xr-x.  75 root root  4096  8月  7 22:06 2013 etc
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 home
dr-xr-xr-x.  10 root root  4096  8月  7 22:05 2013 lib
dr-xr-xr-x.   9 root root 12288  8月  7 22:05 2013 lib64
drwx------.   2 root root 16384  8月  7 21:34 2013 lost+found
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 media
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 mnt
drwxr-xr-x.   3 root root  4096  8月  7 22:06 2013 opt
dr-xr-xr-x. 240 root root     0  9月 23 20:50 2011 proc
dr-xr-x---.   2 root root  4096  8月  7 21:59 2013 root
dr-xr-xr-x.   2 root root 12288  8月  7 22:06 2013 sbin
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 selinux
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 srv
drwxr-xr-x.   2 root root  4096  9月 23 20:50 2011 sys
drwxrwxrwt.   2 root root  4096  8月  7 22:07 2013 tmp
drwxr-xr-x.  13 root root  4096  8月  7 21:54 2013 usr
drwxr-xr-x.  19 root root  4096  8月  7 22:06 2013 var
[root@stack01 ec2-ami]# cd -
/mnt/ec2-ami/dev
```

おー、ディレクトリできてる。

```
chroot /mnt/ec2-ami yum install curl -y

[root@stack01 dev]# chroot /mnt/ec2-ami yum install curl -y
Traceback (most recent call last):
  File "/usr/bin/yum", line 4, in <module>
    import yum
  File "/usr/lib/python2.6/site-packages/yum/__init__.py", line 46, in <module>
    import tempfile
  File "/usr/lib64/python2.6/tempfile.py", line 34, in <module>
    from random import Random as _Random
  File "/usr/lib64/python2.6/random.py", line 873, in <module>
    _inst = Random()
  File "/usr/lib64/python2.6/random.py", line 96, in __init__
    self.seed(x)
  File "/usr/lib64/python2.6/random.py", line 110, in seed
    a = long(_hexlify(_urandom(16)), 16)
OSError: [Errno 2] No such file or directory: '/dev/urandom'
```

`/dev/urandom` のスペシャルデバイスファイルが無いよって言っていると推測した。

```
/sbin/MAKEDEV -d /mnt/ec2-ami/dev -x urandom
```

もういっぱつ

```
chroot /mnt/ec2-ami yum install curl -y
```

うまくいった！

epel入れたいなあ

```
chroot /mnt/ec2-ami yum install wget -y

chroot /mnt/ec2-ami wget ftp://ftp.riken.jp/Linux/fedora/epel/6/x86_64/epel-release-6-8.noarch.rpm
chroot /mnt/ec2-ami rpm -ivh epel-release-6-8.noarch.rpm

chroot /mnt/ec2-ami -y install tmux
```

tmux入った。
これ必須だよね。

```
cat << RC_LOCAL_AMI | tee -a /mnt/ec2-ami/etc/rc.local > /dev/null
depmod -a
modprobe acpiphp
/usr/local/sbin/get-credentials.sh
RC_LOCAL_AMI

chroot /mnt/ec2-ami sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
chroot /mnt/ec2-ami chkconfig ip6tables off
chroot /mnt/ec2-ami chkconfig iptables off
chroot /mnt/ec2-ami chkconfig ntpd on
```

```
cat << CREDS_AMI | tee -a /mnt/ec2-ami/usr/local/sbin/get-credentials.sh > /dev/null
#!/bin/bash

# Retreive the credentials from relevant sources.

# Fetch any credentials presented at launch time and add them to
# root's public keys

PUB_KEY_URI=http://169.254.169.254/1.0/meta-data/public-keys/0/openssh-key
PUB_KEY_FROM_HTTP=/tmp/openssh_id.pub
PUB_KEY_FROM_EPHEMERAL=/mnt/openssh_id.pub
ROOT_AUTHORIZED_KEYS=/root/.ssh/authorized_keys



# We need somewhere to put the keys.
if [ ! -d /root/.ssh ] ; then
mkdir -p /root/.ssh
chmod 700 /root/.ssh
fi

# Fetch credentials...

# First try http
curl --retry 3 --retry-delay 0 --silent --fail -o $PUB_KEY_FROM_HTTP $PUB_KEY_URI
if [ $? -eq 0 -a -e $PUB_KEY_FROM_HTTP ] ; then
if ! grep -q -f $PUB_KEY_FROM_HTTP $ROOT_AUTHORIZED_KEYS
then
cat $PUB_KEY_FROM_HTTP >> $ROOT_AUTHORIZED_KEYS
echo "New key added to authrozied keys file from parameters"|logger -t "ec2"
fi
chmod 600 $ROOT_AUTHORIZED_KEYS
rm -f $PUB_KEY_FROM_HTTP

elif [ -e $PUB_KEY_FROM_EPHEMERAL ] ; then
# Try back to ephemeral store if http failed.
# NOTE: This usage is deprecated and will be removed in the future
if ! grep -q -f $PUB_KEY_FROM_EPHEMERAL $ROOT_AUTHORIZED_KEYS
then
cat $PUB_KEY_FROM_EPHEMERAL >> $ROOT_AUTHORIZED_KEYS
echo "New key added to authrozied keys file from ephemeral store"|logger -t "ec2"

fi
chmod 600 $ROOT_AUTHORIZED_KEYS
chmod 600 $PUB_KEY_FROM_EPHEMERAL

fi

if [ -e /mnt/openssh_id.pub ] ; then
if ! grep -q -f /mnt/openssh_id.pub /root/.ssh/authorized_keys
then
cat /mnt/openssh_id.pub >> /root/.ssh/authorized_keys
echo "New key added to authrozied keys file from ephemeral store"|logger -t "ec2"

fi
chmod 600 /root/.ssh/authorized_keys
fi
CREDS_AMI
```

```
chmod 755 /mnt/ec2-ami/usr/local/sbin/get-credentials.sh

cp /mnt/ec2-ami/boot/initramfs-*.x86_64.img /opt/virt/CentOS6.4
cp /mnt/ec2-ami/boot/vmlinuz-*.x86_64 /opt/virt/CentOS6.4
```

ルートに戻らないとビジー状態でアンマウントできない。

```
cd /
umount /mnt/ec2-ami/proc
umount /mnt/ec2-ami
```

OpenStackで利用するために仮想マシンのOSイメージのラベルを変更しておく。

```
tune2fs -L uec-rootfs /opt/virt/CentOS6.4/CentOS6.4.img
tune2fs 1.41.12 (17-May-2010)
```




### RAMイメージ登録

```
cd /opt/virt/CentOS6.4
glance add name="centos64_ramdisk" is_public=true container_format=ari disk_format=ari < $(ls | grep initram)
Added new image with ID: 4cdd4415-0b01-4c7f-afff-5f94a7480d8c

glance image-create --name="centos64_ramdisk" --is-public=true --container-format=ari --disk-format=ari < $(ls | grep initram)
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ddf2aa27623731a6e592fe1a69494c56     |
| container_format | ari                                  |
| created_at       | 2013-08-07T13:47:20                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | ari                                  |
| id               | b035b594-2af0-42de-a1c5-9b2b05358bdb |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | centos64_ramdisk                     |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 16214072                             |
| status           | active                               |
| updated_at       | 2013-08-07T13:47:20                  |
+------------------+--------------------------------------+
```

### カーネルイメージ登録

```
glance image-create --name="centos64_kernel" --is-public=true --container-format=aki --disk-format=aki < $(ls | grep vmlinuz)
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 89a029d37633f120aafc7babeda54b66     |
| container_format | aki                                  |
| created_at       | 2013-08-07T13:48:20                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | aki                                  |
| id               | 698d61f5-8c6c-4c01-9e54-bbf54eef7f28 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | centos64_kernel                      |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 4045680                              |
| status           | active                               |
| updated_at       | 2013-08-07T13:48:21                  |
+------------------+--------------------------------------+
```


登録したRAMとKERNELのIDが必要です。
イメージは圧縮しておいたほうが容量削減にもなりますし、イメージ登録や仮想マシンの作成も速いです。

```
↓複数イメージが有ると動かない、あうと
#RAMDISK_ID=$(glance image-list | grep centos64_ramdisk | awk -F"|" '{print $2}' | sed -e 's/^[ ]*//g')
#KERNEL_ID=$(glance image-list | grep centos64_kernel | awk -F"|" '{print $2}' | sed -e 's/^[ ]*//g')


[root@stack01 CentOS6.4]# echo $RAMDISK_ID
b035b594-2af0-42de-a1c5-9b2b05358bdb
[root@stack01 CentOS6.4]# echo $KERNEL_ID
698d61f5-8c6c-4c01-9e54-bbf54eef7f28
```

RAWか圧縮するかどっちか選ぶ。
圧縮したほうが速いらしい。

### RAWイメージのままでOSイメージを登録

```
glance image-create --name="centos64_ami" --is-public=true --container-format=ami --disk-format=ami --property kernel_id=$KERNEL_ID --property ramdisk_id=$RAMDISK_ID < CentOS6.4.img
+-----------------------+--------------------------------------+
| Property              | Value                                |
+-----------------------+--------------------------------------+
| Property 'kernel_id'  | 698d61f5-8c6c-4c01-9e54-bbf54eef7f28 |
| Property 'ramdisk_id' | b035b594-2af0-42de-a1c5-9b2b05358bdb |
| checksum              | 5a46facf5953fc758c2ca3b89809a514     |
| container_format      | ami                                  |
| created_at            | 2013-08-07T13:51:01                  |
| deleted               | False                                |
| deleted_at            | None                                 |
| disk_format           | ami                                  |
| id                    | cd280a7a-32f7-4b48-9076-d99a62cf8aa0 |
| is_public             | True                                 |
| min_disk              | 0                                    |
| min_ram               | 0                                    |
| name                  | centos64_ami                         |
| owner                 | 439df062c81244a1af4f977f1450990c     |
| protected             | False                                |
| size                  | 10737418240                          |
| status                | active                               |
| updated_at            | 2013-08-07T13:52:43                  |
+-----------------------+--------------------------------------+
```

### 圧縮してOSイメージを登録

```
イメージコンバート
qemu-img convert -O qcow2 -c CentOS6.4.img CentOS6.4.qcow2

登録（圧倒的にイメージ登録のスピードが速かった）
glance image-create --name="centos64_ami_qcow" --is-public=true --container-format=ami --disk-format=ami --property kernel_id=$KERNEL_ID --property ramdisk_id=$RAMDISK_ID < CentOS6.4.qcow2
+-----------------------+--------------------------------------+
| Property              | Value                                |
+-----------------------+--------------------------------------+
| Property 'kernel_id'  | 698d61f5-8c6c-4c01-9e54-bbf54eef7f28 |
| Property 'ramdisk_id' | b035b594-2af0-42de-a1c5-9b2b05358bdb |
| checksum              | cebd8013fdbaceb56aa5b25d0e7964e3     |
| container_format      | ami                                  |
| created_at            | 2013-08-07T13:55:36                  |
| deleted               | False                                |
| deleted_at            | None                                 |
| disk_format           | ami                                  |
| id                    | 04f3347d-cbd8-4634-98b7-20a0d2692e94 |
| is_public             | True                                 |
| min_disk              | 0                                    |
| min_ram               | 0                                    |
| name                  | centos64_ami_qcow                    |
| owner                 | 439df062c81244a1af4f977f1450990c     |
| protected             | False                                |
| size                  | 560267776                            |
| status                | active                               |
| updated_at            | 2013-08-07T13:55:38                  |
+-----------------------+--------------------------------------+
```

登録が終了したのでOpenStackから利用できるようになっています。
例えば以下のようにブートします。

一般ユーザstackにて

```
nova boot --flavor 1 --image centos64_ami  centos64_001 --key_name mykey
```

但し、rootユーザのパスワードを設定などしていないため公開鍵認証でしかログインできませんので気をつけてください。
もしrootのパスワードを設定する場合は途中で設定しておけばよいでしょう。

**まだまだいろいろあるけど、できそうなことは分かった。**

あれ、物理マシンごと再起動したらインスタンス起動エラーになっちゃった。  
サンプルで紹介されてるfedoraのイメージは元気に動いてる。  
この違いはなんだ。

## virt-installコマンドを使ったインストール

これが一番簡単そう。

```
qemu-img create -f qcow2 /tmp/centos-6.4.qcow2 10G
virt-install --virt-type kvm --name centos-6.4-x64 --ram 1024 \
--disk /tmp/centos-6.4.qcow2,format=qcow2 \
--network network=default \
--nographics \
--os-type=linux --os-variant=rhel6 \
--location=/var/samba/notrsync/OS/Linux/CentOS-6.4-x86_64-bin-DVD1.iso \
--extra-args='console=tty0 console=ttyS0,115200n8'
```

基本的にVNC嫌いなのでグラフィクス切ってます。
流れに沿ってインストール。
rootでログイン。

<!--
CD-ROMのデバイス名を調べて排出する。

```
virsh dumpxml centos-6.4-x64 | grep cdrom -5
      <source file='/tmp/centos-6.4.qcow2'/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </disk>
    <disk type='block' device='cdrom'>
      <driver name='qemu' type='raw'/>
      <target dev='hdc' bus='ide'/>
      <readonly/>
      <alias name='ide0-1-0'/>
      <address type='drive' controller='0' bus='1' target='0' unit='0'/>
```

コンソールに戻る。

```
virsh console centos-6.4-x64
```

空エンター。

-->

鍵の自動取得を設定。
ec2-userが作られる。

```
[root@localhost ~]# ifup eth0

eth0 のIP情報を検出中... 完了。
[root@localhost ~]# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=45 time=42.9 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=45 time=41.9 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1419ms
rtt min/avg/max/mdev = 41.966/42.466/42.967/0.541 ms
[root@localhost ~]# ping www.google.co.jp
PING www.google.co.jp (74.125.235.184) 56(84) bytes of data.
64 bytes from nrt19s12-in-f24.1e100.net (74.125.235.184): icmp_seq=1 ttl=53 time=10.3 ms

--- www.google.co.jp ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 795ms
rtt min/avg/max/mdev = 10.338/10.338/10.338/0.000 ms

rpm -Uvh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
yum -y install cloud-init
```

このあたりでひととおり便利なパッケージは予め入れておこう。

```
yum -y groupinstall "Development Tools"
yum -y install wget vim-enhanced tmux nmap bind-utils openssh-clients whatmask
yum -y update
```

仮想マシン停める。

```
halt
```

MACアドレスとかrules記述を削除する。  
インスタンス複製した時にコンフリクトして問題になるから。

```
yum -y install libguestfs-tools
virt-sysprep -d centos-6.4-x64
（少し時間がかかる）
```

定義削除。

```
virsh undefine centos-6.4-x64
```

あとはバックアップどっかにとっておいてGlanceにイメージ登録すればOK。

```
mv /tmp/centos-6.4.qcow2 /var/samba/
chown nobody:nobody /var/samba/centos-6.4.qcow2
(Windows PCあたりからsambaの共有ディレクトリをマウントして取得してそれをs3にぶん投げるとか)
```

```
glance image-create --name="centos64_ami_qcow" --is-public=true --container-format=ami --disk-format=ami --property kernel_id=$KERNEL_ID --property ramdisk_id=$RAMDISK_ID < CentOS6.4.qcow2
+-----------------------+--------------------------------------+
| Property              | Value                                |
+-----------------------+--------------------------------------+
| Property 'kernel_id'  | None                                 |
| Property 'ramdisk_id' | None                                 |
| checksum              | 924adc867b15f4fe63ade06a16e8a90c     |
| container_format      | ami                                  |
| created_at            | 2013-08-10T17:47:20                  |
| deleted               | False                                |
| deleted_at            | None                                 |
| disk_format           | ami                                  |
| id                    | 80d3b621-b9f4-49c6-a145-58d604e9df3c |
| is_public             | True                                 |
| min_disk              | 0                                    |
| min_ram               | 0                                    |
| name                  | centos64_x64_ami                     |
| owner                 | 439df062c81244a1af4f977f1450990c     |
| protected             | False                                |
| size                  | 1803026432                           |
| status                | active                               |
| updated_at            | 2013-08-10T17:47:35                  |
+-----------------------+--------------------------------------+
```

普通にインスタンス起動したらping帰ってこないんだけど。。。  
DHCPでONBOOT=yesにしたらいけた。  
フローティングipもok  
ec2-userではログイン出来ないみたい。  
rootで鍵を指定すればいいみたい。  
あとシリアルコンソール指定しないと。

```
serial –unit=0 –speed=115200
terminal –timeout=10 console serial
# Edit the kernel line to add the console entries
kernel ... console=tty0 console=ttyS0,115200n8
```

rootへの入り方は以下の様な感じ。

```
$ ssh root@192.168.0.116 -i ~/.ssh/ostackkey.key
```

-------------------------------------------------------------

```
virt-install --connect qemu:///system \
             --name centos-6.4-x64 \
             --ram=1024 \
             --vcpus=1 \
             --hvm \
             --os-type=Linux \
             --os-variant=virtio26 \
             --disk=/var/lib/libvirt/images/example.com.img,device=disk,bus=virtio,size=50,sparse=true,format=raw,cache=writeback \
             --network bridge=br0,model=virtio \
             --nographics \
             --keymap ja \
             --location=/var/tmp/CentOS-6.3-x86_64-bin-DVD1.iso \
             --extra-args='console=tty0 console=ttyS0,115200n8'
```

- [Example: CentOS image - OpenStack Virtual Machine Image Guide  - master](http://docs.openstack.org/trunk/openstack-image/content/centos-image.html)

----------------------------------------------------------------------------

```
glance image-create --name="CentOS64_x64_ami_qcow" --is-public=true --container-format=ami < /var/samba/CentOS6.4-x64.qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 9a66061578a3687b10808d14ed91f197     |
| container_format | ami                                  |
| created_at       | 2013-08-11T10:58:46                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | ami                                  |
| id               | 79ffeaaa-90a2-4b02-9f1e-b64b43aed211 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CentOS64_x64_ami_qcow                |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 1641152512                           |
| status           | active                               |
| updated_at       | 2013-08-11T10:58:52                  |
+------------------+--------------------------------------+
```

virt-sysprepはディスクイメージに対しても使える。
便利。

```
virt-sysprep -a UnicastLinux-x64.qcow2
```

```
glance image-create --name="UnicastLinux_x64_ami_qcow" --is-public=true --container-format=ami < /var/samba/UnicastLinux-x64.qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 22b479c55b92aa9599f5bd60cf4da7f9     |
| container_format | ami                                  |
| created_at       | 2013-08-11T11:30:47                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | ami                                  |
| id               | 5b45b832-1dab-47e2-8cc3-c6197d7df3a8 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | UnicastLinux_x64_ami_qcow            |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 1641218048                           |
| status           | active                               |
| updated_at       | 2013-08-11T11:30:53                  |
+------------------+--------------------------------------+
```

## 参考サイト

- [yumによるOSイメージ作成 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_faq_grizzly/yum/make_ami_centos.html)  
とても参考になった。
- [OpenStack Folsom構築 on Wheezy (8) 独自イメージ作成 | 外道父の匠](http://blog.father.gedow.net/2013/04/08/openstack-folsom-on-wheezy-vol-08/)  
あまり参考にならなかった。
- [virt-install(1): provision new virtual machines - Linux man page](http://linux.die.net/man/1/virt-install)
