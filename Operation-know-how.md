# 運用ノウハウ

OpenStackの運用ノウハウを書きなぐる。

## Topics

- 最近ネットワークコンポーネントのコードネームがQuantumからNeutronに変更になったらしい。  
[Fwd: [Openstack] Quantum is changing its name to... - Google グループ](https://groups.google.com/forum/#!topic/openstack-ja/ScQA_eLd2Gw)

## インスタンスの構成

必要最低限のOSイメージを作成して  
コンテンツは必要になった分だけCinderでボリュームを切り出して接続する  
イメージ、コンテンツのバックアップはスナップショットをとって大容量ストレージに流す。  
Glacierとか？
