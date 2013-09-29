# OpenStack Grizzly on Ubuntu 12.04

4度目の正直。（何度目）

[1. Openstackインストール手順(Grizzly)Ubuntu13.04(パッケージ)編 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_ubuntu1304_apt/)

萩原さんのエントリーにならってみる。  
それにしてもOpenStackのインストールって難しくて大変なんだな。。。

できれば13.04にして環境を合わせたいけど、手持ちのイメージが12.04.3なのでこれで試してみる。  
これがトラブルの原因なのかなあ。

## 構成

- Ubuntu Server 12.04.3 LTS
- Software RAID1 2TB
- LVM: `volume_group00` がセットアップされていること
- Mem: 16GB
- NIC:
  - eth0: `192.168.1.200/24`  
    外部ネットワーク接続あり。
  - eth1: `10.10.100.51/24`  
    マネージメントネットワーク。

## OSのインストール

TODO.

## 参考サイト

- [1. Openstackインストール手順(Grizzly)Ubuntu13.04(パッケージ)編 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_ubuntu1304_apt/)
- [Can't ping virtual machine. Virtual Machine doesn't get IP · Issue #127 · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/issues/127)
- [Openstack console can't see IP networks outside · Issue #126 · mseknibilel/OpenStack-Grizzly-Install-Guide](https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide/issues/126)
