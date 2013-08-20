# VyattaのOpenStackイメージ(AMI)を作成する

## virt-installコマンドを使ったインストール

例によってvirt-installを使います。

```
qemu-img create -f qcow2 /tmp/vyatta66-2.qcow2 10G
virt-install --virt-type kvm --name vyatta66-x64 --ram 1024 \
--disk /tmp/vyatta66-2.qcow2,format=qcow2 \
--network network=default \
--nographics \
--os-type=linux --os-variant=debiansqueeze \
--cdrom=/var/samba/notrsync/OS/Vyatta/vyatta-livecd_VC6.6R1_amd64.iso
```

基本的にVNC嫌いなのでグラフィクス切ってます。

まずは初期ユーザーvyattaでログイン。

```
Welcome to Vyatta - vyatta ttyS0

vyatta login: vyatta
Password: 
Linux vyatta 3.3.8-1-amd64-vyatta #1 SMP Wed Mar 13 10:35:28 PDT 2013 x86_64
Welcome to Vyatta.
This system is open-source software. The exact distribution terms for 
each module comprising the full system are described in the individual 
files in /usr/share/doc/*/copyright.
vyatta@vyatta:~$ 
```

> なお、Vyattaのインストール方法としては「system」および「image」の2種類が提供されている。前者は一般的なLinuxと同様、ハードディスクにファイルシステムを作成して必要なファイルをインストールするものだ。いっぽう後者はVyattaの構成ファイルをディスクイメージの状態でHDD内にインストールし、unionfsを使ってそのディスクイメージを/（ルート）にマウントするという構成になる。unionfsはリードオンリーのディレクトリを別の書き込み可能なディレクトリにマウントすることで、リードオンリーのディレクトリ中に含まれるファイルやディレクトリを仮想的に変更可能なように見せる機構だ。imageインストールを利用した場合、複数のディスクイメージをHDD内にインストールして複数環境を同居させられる点や、初期状態に簡単に戻せる点などがメリットとなる。
> どちらの場合もインストール作業やVyattaの使い方に違いは無いが、今回はストレージを容易に差し替えられる仮想マシン上にVyattaをインストールということで、systemインストールを利用することにする。systemインストールを実行するには、Vyattaへのログインを行った後に以下のように「system」オプション付きでinstallコマンドを実行する。するとインストーラが起動し、以下のように対話的にインストールの際の設定が尋ねられる。その際に[]内で表示されているのがデフォルトの設定値で、何も入力せずにEnterキーを押すとその値が使用される。

[高機能なLinuxベースのソフトウェアルーター「Vyatta」を使う - さくらのナレッジ](http://knowledge.sakura.ad.jp/tech/278/)

とのことなので

```
install system




Would you like to continue? (Yes/No) [Yes]: 
Probing drives: OK
Looking for pre-existing RAID groups...none found.
The Vyatta image will require a minimum 1000MB root.
Would you like me to try to partition a drive automatically
or would you rather partition it manually with parted?  If
you have already setup your partitions, you may skip this step.

Partition (Auto/Union/Parted/Skip) [Auto]: 

I found the following drives on your system:
 vda    10737MB


Install the image on? [vda]:

This will destroy all data on /dev/vda.

How big of a root partition should I create? (1000MB - 10737MB) [10737]MB: 

Creating a new disklabel on vda
parted /dev/vda mklabel msdos
Creating filesystem on /dev/vda1: OK
Mounting /dev/vda1 
Copying system files to /dev/vda1: 
 99% [==================================================>]
OK
I found the following configuration files
/opt/vyatta/etc/config/config.boot
Which one should I copy to vda? [/opt/vyatta/etc/config/config.boot]: 


Enter password for administrator account
Enter password for user 'vyatta': 
Retype password for user 'vyatta':
I need to install the GRUB boot loader.
I found the following drives on your system:
 vda    10737MB


Which drive should GRUB modify the boot partition on? [vda]:

Setting up grub: OK
Done!
```

## cloud-init的なことをやる

同じ事を考えている人がいました。

[Creating a Vyatta AMI](http://www.itisopen.net/2012/02/Creating_a_vyatta_ami/)

```
configure
sudo vi /etc/rc.local
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# 2013-08-11 add noguchi start
# Quick & dirty hack to prevent nash from hogging the CPU:
/usr/bin/killall nash-hotplug

# Import vyatta config from user-data:
su -c "/etc/import_vyatta_config.exp"

# Import ssh public key for this instance and disable password authentication:
su -c "/etc/import_ssh_pubkey.exp" vyatta
# 2013-08-11 add noguchi end

# Do not remove the following call to vyatta-postconfig-bootup.script.
# Any boot time workarounds should be put in script below so that they
# get preserved for the new image during image upgrade.
POSTCONFIG=/opt/vyatta/etc/config/scripts/vyatta-postconfig-bootup.script
[ -x $POSTCONFIG ] && $POSTCONFIG
exit 0
```

実行可能にする。

```
sudo chmod 755 /etc/rc.local
```

expectファイルを新規作成する。  
なにこれ？

```
sudo vi /etc/import_vyatta_config.exp
#!/usr/bin/expect
set timeout 60
spawn $env(SHELL)
send "configure\r"
expect -re  ".*# $"
send "load http://169.254.169.254/latest/user-data \r"
expect {
  "? \\\[no\\\] " {send "n\r"}
  -re "### 100.0%.*# $" {send "commit  \r"}
  timeout {send_user "Error: timeout\n"; exit}
  eof {send_user "Error: eof\n"; exit}
}
#expect -re  ".*# $"
#send "save\r"
expect -re  ".*# $"
send "exit\r"
#expect eof
```

これについても実行可能にするのを忘れずに。

```
sudo chmod 755 /etc/import_vyatta_config.exp
```

expectファイルについて調べてみた。

```
vyatta@vyatta:~$ which expect
/usr/bin/expect
```

存在してる。

[expectコマンドの使い方 – No:158 – Linuxで自宅サーバ構築（新森からの雑記）](http://www.uetyi.mydns.jp/wordpress/command/entry-158.html)

こんなものらしい。対話型の自動化コマンドか。  
とりあえず次へ。

```
sudo vi /etc/import_ssh_pubkey.exp
#!/usr/bin/expect
set timeout 30
spawn $env(SHELL)
send "configure\r"
expect -re  ".*# $"
send "loadkey vyatta http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key \r"
expect {
  -re "### 100.0%.*# $" {send "set service ssh disable-password-authentication \r"}
  timeout {send_user "timeout @1\n"; exit}
  eof {send_user "eof @1\n"; exit}
}
expect -re  ".*# $"
send "commit\r"
#expect -re  ".*# $"
#send "save\r"
expect -re  ".*# $"
send "exit\r"
expect {
  -re ":\[~/\]\[#$\] " {send "cat ~vyatta/.ssh/authorized_keys\r"}
  timeout {send_user "timeout @2\n"; exit}
  eof {send_user "eof @2\n"; exit}
}
expect {
  -re ":\[~/\]\[#$\] " {send "exit\r"}
  timeout {send_user "timeout @3\n"; exit}
  eof {send_user "eof @3\n"; exit}
}
#expect eof
```

```
sudo chmod 755 /etc/import_ssh_pubkey.exp
```

メンテナンスモード移行。

```
configure
```

必要最低限の設定をする。  
SSHデーモンの起動とDHCPの設定。  
それとMACアドレスの定義削除する。
`xx:xx:xx:xx:xx:xx` の部分は自分の環境に合わせてください。

```
set interfaces ethernet eth0 address dhcp
delete interfaces ethernet eth0 hw-id xx:xx:xx:xx:xx:xx
set service ssh
```

```
[edit]
vyatta@vyatta# commit
[ interfaces ethernet eth0 address dhcp ]
Starting DHCP client on eth0 ...

[ service ssh ]
Restarting OpenBSD Secure Shell server: sshd.

[edit]
vyatta@vyatta# save
Saving configuration to '/config/config.boot'...
Done
```

VM停止。

```
sudo halt
```

試しにglanceに登録してみてインスタンス起動してpingとか飛んだりvyattaユーザーでログイン出来ればOK。

```

glance image-create --name="Vyatta66_x64_ami_qcow" --is-public=true --container-format=ami < /tmp/vyatta66.qcow2
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | 0d873e6c192e66f05da0cff07de92d9b     |
| container_format | ami                                  |
| created_at       | 2013-08-11T13:23:43                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | ami                                  |
| id               | 4f2de947-c7ba-44e9-a320-74448f9bcaf4 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | Vyatta66_x64_ami_qcow                |
| owner            | 439df062c81244a1af4f977f1450990c     |
| protected        | False                                |
| size             | 961478656                            |
| status           | active                               |
| updated_at       | 2013-08-11T13:23:46                  |
+------------------+--------------------------------------+
```

イメージをs3とかにぶん投げる。

```
cp /tmp/vyatta66.qcow2 /var/samba/
```

どっかでしくって動かない。。。  
user-dataとかに設定とかぶっこまないといけないのかな。。。  
また明日。。。

- [migrate r1-system firewall configure failedÃ¢â‚¬Â¦ | Vyatta.org Community](http://www.vyatta.org/node/5072)
- [VM disk image の mount 方法いくつかメモ - (」・ω・)」うー!(/・ω・)/にゃー!](http://yudoufu.github.io/blog/2012/04/28/vm-mount-memo/)
- [qcow2ファイルをマウント | やってみようよ！](http://www.postcard.st/nosuz/blog/2011/09/10-14)
- [EC2: User Dataを使ってインスタンス起動時の処理を自動化する - aws memo](http://understeer.hatenablog.com/entry/2012/07/19/123050)

```
 interfaces {
     ethernet eth0 {
         address dhcp
     }
     loopback lo {
     }
 }
 service {
     ssh {
     }
 }
 system {
     config-management {
         commit-revisions 20
     }
     console {
         device ttyS0 {
             speed 9600
         }
     }
     login {
         user vyatta {
             authentication {
                 encrypted-password $1$4XHPj9eT$G3ww9B/pYDLSXC8YVvazP0
             }
             level admin
         }
     }
     ntp {
         server 0.vyatta.pool.ntp.org {
         }
         server 1.vyatta.pool.ntp.org {
         }
         server 2.vyatta.pool.ntp.org {
         }
     }
     package {
         repository community {
             components main
             distribution stable
             url http://packages.vyatta.com/vyatta
         }
     }
     syslog {
         global {
             facility all {
                 level notice
             }
             facility protocols {
                 level debug
             }
         }
     }
 }

```

## 参考サイト

- [高機能なLinuxベースのソフトウェアルーター「Vyatta」を使う - さくらのナレッジ](http://knowledge.sakura.ad.jp/tech/278/)
- [Creating a Vyatta AMI](http://www.itisopen.net/2012/02/Creating_a_vyatta_ami/)
- [Vyatta Core 6.3のAMIを作ってみた - log4moto](http://d.hatena.ne.jp/j3tm0t0/20111113/1321192227)
- [CloudInitを使ってEC2インスタンスをスマートに立ち上げる | はったりエンジニアの備忘録](http://blog.manabusakai.com/amazon-ec2-cloudinit/)


[Vyatta System - AMI INSTALLING THE SYSTEM/USERGUIDE]
http://www.vyatta.com/downloads/documentation/VC6.5/Vyatta-InstallUpgradeAMI_6.5R3_v02.pdf


