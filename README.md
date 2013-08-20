# OpenStack

プライベートクラウドを構築するコンポーネント群のOSS。  
OpenStackを導入するにあたって参考にさせていただいたサイトや手順等をメモしていきます。

## 概要

OpenStackは大きく2つの種類に分類される。

- Cloud Controller node
- Compute Nodes

Cloud Controller nodeは制御ノードと呼ばれて、novaとかglanceとかいろいろ入ります。  
制御用のノードなので比較的計算機リソースはディスクが30GBぐらい、  
メモリが12GBぐらい必要になります。  
でも実験用とかだったら仮想マシンのイメージとかブロックストレージ等の永続化データも  
この制御ノードと一緒にされるのでディスクは1〜2TB必要。  
NIC二枚差し推奨らしい。実験用ならNIC一枚でも動きます。  

Compute Nodesは実際に仮想マシンを稼働させるノード。計算機リソースとしてのノードなのでディスクは30GBと少なめ。  
ただメモリは32GB以上推奨。  
仮想化対応必須。KVMとかVMwareとかXenとか。  
うちはKVMですね。

どのみちストレージが必要なのでCinderをどっかで稼働させないといけない。  
Compute Nodesに入れるのかな。  
要調査。

[AWS](http://aws.amazon.com/jp/) と互換性があるのでAWS触っておくと理解が速いと思う。

### 構成されるコンポーネント

- Identity Service(Keystone)
- Image delivery and registration (Glance)
- Volume Service(Cinder)
- Cloud compute (Nova)
- Dashboard (Horizon)

他に必要なのは

- Neutron（旧：Quantum）  
[ネットワークコンポーネント](Neutron/Neutron.md) 。
- Swift  
Amazon s3的なやつ。

## 近況

最近活動がとても活発です。  
将来が楽しみです。

![](img/trends.png)

- **ネットワーク管理コンポーネントのQuantumはNeutronに名前が変わった。**  
[Fwd: [Openstack] Quantum is changing its name to... - Google グループ](https://groups.google.com/forum/#!topic/openstack-ja/ScQA_eLd2Gw)

## インストール

私のOpenStackインストールの記録です。

1. [初めてのOpenStackインストールの記録](install/1-CentOS6.4-x64-FirstTimeInstall/install.md)
1. [1つの仮想マシンの中にDevStackでオールインワンなOpenStack環境を作ってみる](install/2-OpenStackOnVM/install.md)
1. [UbuntuにAllInOne構成でOpenStackをインストール(Grizzly)](install/3-UbuntuAllInOne/install.md)

## 参考サイト

### インストール関連

- [3.1. Openstack概要 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/openstack_summary.html)
- [3. Openstackインストール手順(Grizzly)CentOS6.4(パッケージ)編 —
オープンソースに関するドキュメント 1.1
documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_centos64_yum/)
- [1. Openstackインストール手順(Grizzly)Ubuntu13.04(パッケージ)編 — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_grizzly_ubuntu1304_apt/index.html)
- [4. Openstackコマンド関連(Grizzly) — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_com_grizzly/index.html)
- [1. OpenStackFAQ(Grizzly) — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/openstack_faq_grizzly/index.html)
- [さくらの専用サーバとOpenStackで作るプライベートクラウド | SourceForge.JP Magazine](http://sourceforge.jp/magazine/12/09/18/1126211)  
ちょっと記事が古めで長くて眠くなりそう。  
ただ、データセンター使ってプライベとクラウド作るのでとても参考になりそう。

#### あとで読む的な

- [mseknibilel/OpenStack-Folsom-Install-guide](https://github.com/mseknibilel/OpenStack-Folsom-Install-guide)
- [Chapter 1. OpenStack Basic Install - OpenStack Basic Install](http://docs.openstack.org/folsom/basic-install/content/)
- [OpenStack Installation Guide for Red Hat Enterprise Linux, CentOS, and Fedora - OpenStack Installation Guide for Red Hat Enterprise Linux, CentOS, and Fedora  - Grizzly, 2013.1 with Object Storage 1.8.0](http://docs.openstack.org/grizzly/openstack-compute/install/yum/content/)
- [virtualtech.jp/download/120907OpenStack.pdf](http://virtualtech.jp/download/120907OpenStack.pdf)
ちょい古め。
- [OpenStack Grizzly 構築スクリプト - jedipunkz' blog](http://jedipunkz.github.io/blog/2013/04/20/openstack-grizzly-installation-script/)

### 公式サイト的な

- [OpenStack](https://wiki.openstack.org/wiki/Main_Page)
- [日本OpenStackユーザ会 - Japan OpenStack User Group Japan (JOSUG) openstack.jp](http://openstack.jp/)
- [openstack (OpenStack)](https://github.com/openstack)

### マニュアルとか

- [OpenStack 運用ガイド(PDF)](http://dream.daynight.jp/openstack/openstack-ops/openstack-ops-manual-local.pdf)
- [open OpenStack Docs: current](http://docs.openstack.org/trunk/)
- [これからはじめるOpenStackリンク集 | 外道父の匠](http://blog.father.gedow.net/2013/02/19/openstack-links/)
- [オープンソースに関するドキュメント — オープンソースに関するドキュメント 1.1 documentation](http://oss.fulltrust.co.jp/doc/index.html)

### DevStack

- [openstack-dev/devstack](https://github.com/openstack-dev/devstack)
- [DevStack - Deploying OpenStack for Developers](http://devstack.org/)
- [「オープンソース」を使ってみよう (第23回 DevStackでラクラク導入！ OpenStackを使ってみよう編)](http://www.ospn.jp/press/20120828no27-useit-oss.html)

### その他slideshareとか

- [OpenStack の利用](http://www.slideshare.net/yosshy/openstack-14884093)
- [OpenStack勉強会](http://www.slideshare.net/obara13/open-stack-16166193)
- [OSC2012 Tokyo Fall OpenStack Essex Multinode Demo](http://www.slideshare.net/irix_jp/osc2012-tokyofall-josugopenstackdemo2v2)
- [Control distribution of virtual machines](http://www.slideshare.net/irix_jp/josug-13thstudyregionandazahcells-v2)
- [What's New In Grizzly: Nova](http://www.slideshare.net/mirantis/grizzly-web-cast-nova-22887279)
