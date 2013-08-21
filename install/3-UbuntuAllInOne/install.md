# UbuntuにAllInOne構成でOpenStackをインストール(Grizzly)

DevStackを使う。

## インストール

```
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get -y install git
```

### ユーザーstackを追加してパス無しsudoできるようにする

```
sudo adduser stack
sudo visudo
```

```
stack ALL=(ALL) NOPASSWD: ALL
```

```
logout
```

以降はユーザーstackでログインして作業。

```
git clone git://github.com/openstack-dev/devstack.git
cd devstack
git checkout stable/grizzly
```

```
cat <<EOF >localrc
FLOATING_RANGE=192.168.0.96/27
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=eth0
ADMIN_PASSWORD=supersecret
MYSQL_PASSWORD=iheartdatabases
RABBIT_PASSWORD=flopsymopsy
SERVICE_PASSWORD=iheartksl
EOF
```

パスワード入力を促されるがそのまま空エンターでOK。

```
./stack.sh
(snip)
[ERROR] ./stack.sh:638 nova-api did not start
```

なんやねん・・・。

[python - Error in devstack script. nova-api did not start? - Stack Overflow](http://stackoverflow.com/questions/17079919/error-in-devstack-script-nova-api-did-not-start)

もういっかい。

```
./unstack.sh
./stack.sh
```

回線細すぎてエラーになる。  
一回手動でnovaのリポジトリはとってこよう。

```
git clone https://github.com/openstack/nova.git /opt/stack/nova
error: RPC failed; result=18, HTTP code = 200 MiB | 32 KiB/s   
fatal: The remote end hung up unexpectedly
fatal: early EOF
fatal: index-pack failed
```

HTTPSだとだめみたい。
gitプロトコルに切り替える。

```
stack@wstack:~/devstack$ git clone git://github.com/openstack/nova.git /opt/stack/nova
Cloning into '/opt/stack/nova'...
remote: Counting objects: 199622, done.
remote: Compressing objects: 100% (50544/50544), done.
remote: Total 199622 (delta 159731), reused 182185 (delta 143338)
Receiving objects: 100% (199622/199622), 122.27 MiB | 96 KiB/s, done.
Resolving deltas: 100% (159731/159731), done.
```

```
./unstack.sh
./stack.sh


Horizon is now available at http://192.168.0.10/
Keystone is serving at http://192.168.0.10:5000/v2.0/
Examples on using novaclient command line is in exercise.sh
The default users are: admin and demo
The password: supersecret
This is your host ip: 192.168.0.10
stack.sh completed in 245 seconds.
```

とりあえずうまくいった。
cirros立ち上げてセキュリティグループも編集したけどping飛ばない。  
ログ見てみた。

```
checking http://169.254.169.254/2009-04-04/instance-id
failed 1/20: up 1.06. request failed
failed 2/20: up 6.07. request failed
failed 3/20: up 9.07. request failed
failed 4/20: up 14.07. request failed
failed 5/20: up 17.07. request failed
failed 6/20: up 22.08. request failed
failed 7/20: up 25.08. request failed
failed 8/20: up 30.09. request failed
failed 9/20: up 33.09. request failed
failed 10/20: up 38.10. request failed
failed 11/20: up 41.10. request failed
failed 12/20: up 46.11. request failed
```

たぶんNICまわりだろうなあ・・・。

```
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service neutron
# Optional, to enable tempest configuration as part of devstack
enable_service tempest
```

かなり何度もunstack and stackを繰り返すことになりそうなのでaliasを定義する。

```
./unstack.sh && ./stack.sh
alias restack='./unstack.sh && ./stack.sh'
```

DevStackはタイミングの問題でエラーになることが多々あるから何回か打ちなおしてみるのもいいんじゃないですかね。

**うお、Quantum追加したらなんかメニュー増えた。**

![](img/quantum1.png)

![](img/quantum2.png)

これはやばいですね・・・。

### Neutron系が意味不明

**わからなくなったら `neutron help` 。**

[quantum · irixjp/openstack-study-9 Wiki](https://github.com/irixjp/openstack-study-9/wiki/quantum)

とりあえずNeutronのHorizonインタフェースは豪華だけど微妙に不完全らしいからコマンド体系を自分で
補完しないといけないっぽい。

```
stack@wstack:~/devstack$ quantum net-list
+--------------------------------------+---------+----------------------------------------------------+
| id                                   | name    | subnets                                            |
+--------------------------------------+---------+----------------------------------------------------+
| 7e842f83-2713-4bba-8e98-2d62b9221667 | public  | 1dd975a1-9c80-4fcc-9478-2d2a4415cd08               |
| a18fea4e-a6e7-43b6-a24c-73858a7b7464 | private | c30bd4f0-a232-459b-98b5-d78fbc08941a 10.11.12.0/24 |
+--------------------------------------+---------+----------------------------------------------------+

stack@wstack:~/devstack$ quantum net-create mynet1
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 45efb95b-448d-4e85-a72c-5956b4ca8435 |
| name            | mynet1                               |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 135c9a897878463f988c3ac68516b759     |
+-----------------+--------------------------------------+

stack@wstack:~/devstack$ quantum net-list
+--------------------------------------+---------+----------------------------------------------------+
| id                                   | name    | subnets                                            |
+--------------------------------------+---------+----------------------------------------------------+
| 45efb95b-448d-4e85-a72c-5956b4ca8435 | mynet1  |                                                    |
| 7e842f83-2713-4bba-8e98-2d62b9221667 | public  | 1dd975a1-9c80-4fcc-9478-2d2a4415cd08               |
| a18fea4e-a6e7-43b6-a24c-73858a7b7464 | private | c30bd4f0-a232-459b-98b5-d78fbc08941a 10.11.12.0/24 |
+--------------------------------------+---------+----------------------------------------------------+

stack@wstack:~/devstack$ quantum subnet-create --ip-version 4 --gateway 172.26.0.254 45efb95b-448d-4e85-a72c-5956b4ca8435 172.26.0.0/24
Created a new subnet:
+------------------+------------------------------------------------+
| Field            | Value                                          |
+------------------+------------------------------------------------+
| allocation_pools | {"start": "172.26.0.1", "end": "172.26.0.253"} |
| cidr             | 172.26.0.0/24                                  |
| dns_nameservers  |                                                |
| enable_dhcp      | True                                           |
| gateway_ip       | 172.26.0.254                                   |
| host_routes      |                                                |
| id               | 5fe01e0e-de3b-4fac-ac13-c6d0bc56684a           |
| ip_version       | 4                                              |
| name             |                                                |
| network_id       | 45efb95b-448d-4e85-a72c-5956b4ca8435           |
| tenant_id        | 135c9a897878463f988c3ac68516b759               |
+------------------+------------------------------------------------+

stack@wstack:~/devstack$ quantum router-create myrouter1
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 43274dec-30a7-485c-ba9b-4e27175970f2 |
| name                  | myrouter1                            |
| status                | ACTIVE                               |
| tenant_id             | 135c9a897878463f988c3ac68516b759     |
+-----------------------+--------------------------------------+


stack@wstack:~/devstack$ sudo ip netns exec qrouter-43274dec-30a7-485c-ba9b-4e27175970f2 route -n
Cannot open network namespace: No such file or directory
stack@wstack:~/devstack$ ip netns list
qrouter-cc84daca-df5b-4521-a477-01919ff396ef
qdhcp-a18fea4e-a6e7-43b6-a24c-73858a7b7464
qrouter-7be8690e-8a61-48c2-926f-f87a90a55ee7
qdhcp-b3ffd65b-dc25-4d9d-a729-1342122cfcf4



```

[Question #206604 : Questions : neutron](https://answers.launchpad.net/neutron/+question/206604)

```
stack@wstack:~/devstack$ quantum router-list
+--------------------------------------+-----------+--------------------------------------------------------+
| id                                   | name      | external_gateway_info                                  |
+--------------------------------------+-----------+--------------------------------------------------------+
| 43274dec-30a7-485c-ba9b-4e27175970f2 | myrouter1 | null                                                   |
| cc84daca-df5b-4521-a477-01919ff396ef | router1   | {"network_id": "7e842f83-2713-4bba-8e98-2d62b9221667"} |
+--------------------------------------+-----------+--------------------------------------------------------+
```


増殖する仕様

```


stack@wstack:~/devstack$ ip netns list
qrouter-7f95083d-2e80-413b-bd1b-5ece1a16f193
qdhcp-da940418-988a-484a-a32a-eabad1a62cc4

stack@wstack:~/devstack$ neutron router-list
+--------------------------------------+---------+--------------------------------------------------------+
| id                                   | name    | external_gateway_info                                  |
+--------------------------------------+---------+--------------------------------------------------------+
| 7f95083d-2e80-413b-bd1b-5ece1a16f193 | router1 | {"network_id": "c23a6e36-977e-4936-932b-75d2dad0bfcf"} |
+--------------------------------------+---------+--------------------------------------------------------+


stack@wstack:~/devstack$ neutron net-list
+--------------------------------------+---------+----------------------------------------------------+
| id                                   | name    | subnets                                            |
+--------------------------------------+---------+----------------------------------------------------+
| c23a6e36-977e-4936-932b-75d2dad0bfcf | public  | b86dc4ed-89a7-4b1b-8e55-cd97dc98434e               |
| da940418-988a-484a-a32a-eabad1a62cc4 | private | dd641ea4-4a94-497c-b392-c1cc28d831ec 10.11.12.0/24 |
+--------------------------------------+---------+----------------------------------------------------+

```

Neutron系のコマンド体系を諦めてQuantum系へシフト。

```
quantum subnet-create --ip-version 4 --gateway 192.168.0.1 04a49673-b5b6-4b1b-bde9-99ad97045a0a 192.168.0.0/24

stack@wstack:~/devstack$ quantum net-create mynet1
Created a new network:
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | 04a49673-b5b6-4b1b-bde9-99ad97045a0a |
| name            | mynet1                               |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         |                                      |
| tenant_id       | 69f39d019d6e4b15a0f48dea264646ef     |
+-----------------+--------------------------------------+


stack@wstack:~/devstack$ quantum subnet-create --ip-version 4 --gateway 192.168.0.1 04a49673-b5b6-4b1b-bde9-99ad97045a0a 192.168.0.0/24
Created a new subnet:
+------------------+--------------------------------------------------+
| Field            | Value                                            |
+------------------+--------------------------------------------------+
| allocation_pools | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| cidr             | 192.168.0.0/24                                   |
| dns_nameservers  |                                                  |
| enable_dhcp      | True                                             |
| gateway_ip       | 192.168.0.1                                      |
| host_routes      |                                                  |
| id               | 20730efe-0e7f-414d-bd9f-1f06342128a8             |
| ip_version       | 4                                                |
| name             |                                                  |
| network_id       | 04a49673-b5b6-4b1b-bde9-99ad97045a0a             |
| tenant_id        | 69f39d019d6e4b15a0f48dea264646ef                 |
+------------------+--------------------------------------------------+


stack@wstack:~/devstack$ quantum router-create myrouter1
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 72a879bd-3f70-4e91-becb-a17b1191a83d |
| name                  | myrouter1                            |
| status                | ACTIVE                               |
| tenant_id             | 69f39d019d6e4b15a0f48dea264646ef     |
+-----------------------+--------------------------------------+

ルーターがっちゃんこ

stack@wstack:~/devstack$ quantum router-gateway-set 72a879bd-3f70-4e91-becb-a17b1191a83d c23a6e36-977e-4936-932b-75d2dad0bfcf
Set gateway for router 72a879bd-3f70-4e91-becb-a17b1191a83d
```

現在の状態

![](img/network_state.png)

```
stack@wstack:~/devstack$ ping 192.168.0.98
PING 192.168.0.98 (192.168.0.98) 56(84) bytes of data.
64 bytes from 192.168.0.98: icmp_req=1 ttl=64 time=0.299 ms
64 bytes from 192.168.0.98: icmp_req=2 ttl=64 time=0.049 ms
64 bytes from 192.168.0.98: icmp_req=3 ttl=64 time=0.055 ms
^C
--- 192.168.0.98 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1998ms
rtt min/avg/max/mdev = 0.049/0.134/0.299/0.116 ms
stack@wstack:~/devstack$ ping 192.168.0.99
PING 192.168.0.99 (192.168.0.99) 56(84) bytes of data.
64 bytes from 192.168.0.99: icmp_req=1 ttl=64 time=0.327 ms
64 bytes from 192.168.0.99: icmp_req=2 ttl=64 time=0.045 ms
64 bytes from 192.168.0.99: icmp_req=3 ttl=64 time=0.054 ms
^C
--- 192.168.0.99 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
rtt min/avg/max/mdev = 0.045/0.142/0.327/0.130 ms
stack@wstack:~/devstack$ ping 192.168.0.100
PING 192.168.0.100 (192.168.0.100) 56(84) bytes of data.
^C
--- 192.168.0.100 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 999ms

```

pingとぶ


```
quantum subnet-create --ip-version 4 --gateway 172.16.0.1 fd4e5c96-2333-4832-b383-24fedc2765bf 172.26.0.0/24




stack@wstack:~/devstack$ quantum help | grep show
  --version             show program's version number and exit
  -h, --help            show this help message and exit
  --debug               show tracebacks on errors
  agent-show                     Show information of a given agent.
  ext-show                       Show information of a given resource.
  floatingip-show                Show information of a given floating ip.
  lb-healthmonitor-show          Show information of a given healthmonitor.
  lb-member-show                 Show information of a given member.
  lb-pool-show                   Show information of a given pool.
  lb-vip-show                    Show information of a given vip.
  net-gateway-show               Show information of a given network gateway.
  net-show                       Show information of a given network.
  port-show                      Show information of a given port.
  queue-show                     Show information of a given queue.
  quota-show                     Show quotas of a given tenant
  router-show                    Show information of a given router.
  security-group-rule-show       Show information of a given security group rule.
  security-group-show            Show information of a given security group.
  subnet-show                    Show information of a given subnet.



stack@wstack:~/devstack$ quantum net-show private
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | da940418-988a-484a-a32a-eabad1a62cc4 |
| name            | private                              |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         | dd641ea4-4a94-497c-b392-c1cc28d831ec |
| tenant_id       | 69f39d019d6e4b15a0f48dea264646ef     |
+-----------------+--------------------------------------+
stack@wstack:~/devstack$ quantum router-interface-add 72a879bd-3f70-4e91-becb-a17b1191a83d dd641ea4-4a94-497c-b392-c1cc28d831ec
Invalid input for operation: IP address 10.0.0.1 is not a valid IP for the defined subnet.



stack@wstack:~/devstack$ quantum subnet-create --ip-version 4 --gateway 10.0.0.1 private 10.0.10.0/24
Created a new subnet:
+------------------+----------------------------------------------+
| Field            | Value                                        |
+------------------+----------------------------------------------+
| allocation_pools | {"start": "10.0.10.1", "end": "10.0.10.254"} |
| cidr             | 10.0.10.0/24                                 |
| dns_nameservers  |                                              |
| enable_dhcp      | True                                         |
| gateway_ip       | 10.0.0.1                                     |
| host_routes      |                                              |
| id               | 4287b9bb-440e-4ca7-90ea-c7119d69bb7a         |
| ip_version       | 4                                            |
| name             |                                              |
| network_id       | da940418-988a-484a-a32a-eabad1a62cc4         |
| tenant_id        | 69f39d019d6e4b15a0f48dea264646ef             |
+------------------+----------------------------------------------+



stack@wstack:~/devstack$ quantum router-interface-add 72a879bd-3f70-4e91-becb-a17b1191a83d 4287b9bb-440e-4ca7-90ea-c7119d69bb7a
Invalid input for operation: IP address 10.0.0.1 is not a valid IP for the defined subnet.



stack@wstack:~/devstack$ quantum subnet-create --ip-version 4 --gateway 10.0.20.1 private 10.0.20.0/24
Created a new subnet:
+------------------+----------------------------------------------+
| Field            | Value                                        |
+------------------+----------------------------------------------+
| allocation_pools | {"start": "10.0.20.2", "end": "10.0.20.254"} |
| cidr             | 10.0.20.0/24                                 |
| dns_nameservers  |                                              |
| enable_dhcp      | True                                         |
| gateway_ip       | 10.0.20.1                                    |
| host_routes      |                                              |
| id               | 78214993-0de5-44ec-b364-a3e7fe5cb031         |
| ip_version       | 4                                            |
| name             |                                              |
| network_id       | da940418-988a-484a-a32a-eabad1a62cc4         |
| tenant_id        | 69f39d019d6e4b15a0f48dea264646ef             |
+------------------+----------------------------------------------+


stack@wstack:~/devstack$ quantum router-interface-add 72a879bd-3f70-4e91-becb-a17b1191a83d 78214993-0de5-44ec-b364-a3e7fe5cb031
Added interface 121b6830-c1da-460a-b8a3-bb01e524e45e to router 72a879bd-3f70-4e91-becb-a17b1191a83d.


```

結線されました。

![](img/network_state2.png)

インスタンス起動したら変なサブネットに結びついちゃった。

```
stack@wstack:~/devstack$ neutron subnet-list
+--------------------------------------+------+----------------+--------------------------------------------------+
| id                                   | name | cidr           | allocation_pools                                 |
+--------------------------------------+------+----------------+--------------------------------------------------+
| 20730efe-0e7f-414d-bd9f-1f06342128a8 |      | 192.168.0.0/24 | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| 4287b9bb-440e-4ca7-90ea-c7119d69bb7a |      | 10.0.10.0/24   | {"start": "10.0.10.1", "end": "10.0.10.254"}     |
| 78214993-0de5-44ec-b364-a3e7fe5cb031 |      | 10.0.20.0/24   | {"start": "10.0.20.2", "end": "10.0.20.254"}     |
| dd641ea4-4a94-497c-b392-c1cc28d831ec |      | 10.11.12.0/24  | {"start": "10.11.12.1", "end": "10.11.12.254"}   |
+--------------------------------------+------+----------------+--------------------------------------------------+

stack@wstack:~/devstack$ neutron help | grep subnet
  subnet-create                  Create a subnet for a given tenant.
  subnet-delete                  Delete a given subnet.
  subnet-list                    List networks that belong to a given tenant.
  subnet-show                    Show information of a given subnet.
  subnet-update                  Update subnet's information.


stack@wstack:~/devstack$ neutron subnet-show 4287b9bb-440e-4ca7-90ea-c7119d69bb7a
+------------------+----------------------------------------------+
| Field            | Value                                        |
+------------------+----------------------------------------------+
| allocation_pools | {"start": "10.0.10.1", "end": "10.0.10.254"} |
| cidr             | 10.0.10.0/24                                 |
| dns_nameservers  |                                              |
| enable_dhcp      | True                                         |
| gateway_ip       | 10.0.0.1                                     |
| host_routes      |                                              |
| id               | 4287b9bb-440e-4ca7-90ea-c7119d69bb7a         |
| ip_version       | 4                                            |
| name             |                                              |
| network_id       | da940418-988a-484a-a32a-eabad1a62cc4         |
| tenant_id        | 69f39d019d6e4b15a0f48dea264646ef             |
+------------------+----------------------------------------------+

デフォゲが間違ってるので修正する。


xxxxxxxxxx

```

ちょっとノイズになりそうな定義が増えすぎたので片っ端から削除する。

![](img/network_state3.png)

```
stack@wstack:~/devstack$ neutron subnet-update 4287b9bb-440e-4ca7-90ea-c7119d69bb7a  --gateway 10.0.10.1
400-{u'QuantumError': u"Unrecognized attribute(s) 'gateway'"}
stack@wstack:~/devstack$ neutron subnet-update 4287b9bb-440e-4ca7-90ea-c7119d69bb7a --gateway_ip 10.0.10.1                                                         
409-{u'QuantumError': u'Gateway ip 10.0.10.1 conflicts with allocation pool 10.0.10.1-10.0.10.254'}
stack@wstack:~/devstack$ restack
コマンド 'restack' は見つかりませんでした。もしかして:
 Command 'rxstack' from package 'regina-rexx' (universe)
restack: コマンドが見つかりません
stack@wstack:~/devstack$ neutron subnet-delete 20730efe-0e7f-414d-bd9f-1f06342128a8
Deleted subnet: 20730efe-0e7f-414d-bd9f-1f06342128a8
stack@wstack:~/devstack$ neutron subnet-delete 4287b9bb-440e-4ca7-90ea-c7119d69bb7a
409-{u'QuantumError': u'Unable to complete operation on subnet 4287b9bb-440e-4ca7-90ea-c7119d69bb7a. One or more ports have an IP allocation from this subnet.'}
stack@wstack:~/devstack$ neutron subnet-delete 4287b9bb-440e-4ca7-90ea-c7119d69bb7a
Deleted subnet: 4287b9bb-440e-4ca7-90ea-c7119d69bb7a
stack@wstack:~/devstack$ neutron subnet-delete dd641ea4-4a94-497c-b392-c1cc28d831ec
Deleted subnet: dd641ea4-4a94-497c-b392-c1cc28d831ec
stack@wstack:~/devstack$ neutron router-delete router1
Deleted router: router1
stack@wstack:~/devstack$ neutron net-delete mynet1
Deleted network: mynet1




stack@wstack:~/devstack$ sudo ip netns exec qrouter-72a879bd-3f70-4e91-becb-a17b1191a83d iptables -nvL -t nat
Chain PREROUTING (policy ACCEPT 6 packets, 1624 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   26  2872 quantum-l3-agent-PREROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain INPUT (policy ACCEPT 24 packets, 2704 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 quantum-l3-agent-OUTPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain POSTROUTING (policy ACCEPT 2 packets, 168 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    2   168 quantum-l3-agent-POSTROUTING  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    2   168 quantum-postrouting-bottom  all  --  *      *       0.0.0.0/0            0.0.0.0/0           

Chain quantum-l3-agent-OUTPUT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 DNAT       all  --  *      *       0.0.0.0/0            192.168.0.100        to:10.0.20.3

Chain quantum-l3-agent-POSTROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 ACCEPT     all  --  !qg-3c17523a-29 !qg-3c17523a-29  0.0.0.0/0            0.0.0.0/0            ! ctstate DNAT

Chain quantum-l3-agent-PREROUTING (1 references)
 pkts bytes target     prot opt in     out     source               destination         
   18  1080 REDIRECT   tcp  --  *      *       0.0.0.0/0            169.254.169.254      tcp dpt:80 redir ports 9697
    2   168 DNAT       all  --  *      *       0.0.0.0/0            192.168.0.100        to:10.0.20.3

Chain quantum-l3-agent-float-snat (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 SNAT       all  --  *      *       10.0.20.3            0.0.0.0/0            to:192.168.0.100

Chain quantum-l3-agent-snat (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    2   168 quantum-l3-agent-float-snat  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
    0     0 SNAT       all  --  *      *       10.0.20.0/24         0.0.0.0/0            to:192.168.0.99

Chain quantum-postrouting-bottom (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    2   168 quantum-l3-agent-snat  all  --  *      *       0.0.0.0/0            0.0.0.0/0           



stack@wstack:~/devstack$ sudo ip netns exec qrouter-72a879bd-3f70-4e91-becb-a17b1191a83d ip addr
16: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
17: qg-3c17523a-29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:63:e7:93 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.99/27 brd 192.168.0.127 scope global qg-3c17523a-29
    inet 192.168.0.100/32 brd 192.168.0.100 scope global qg-3c17523a-29
    inet6 fe80::f816:3eff:fe63:e793/64 scope link 
       valid_lft forever preferred_lft forever
18: qr-121b6830-c1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN 
    link/ether fa:16:3e:8b:fd:4b brd ff:ff:ff:ff:ff:ff
    inet 10.0.20.1/24 brd 10.0.20.255 scope global qr-121b6830-c1
    inet6 fe80::f816:3eff:fe8b:fd4b/64 scope link 
       valid_lft forever preferred_lft forever



stack@wstack:~/devstack$ sudo ovs-vsctl show
63d51e05-b512-4964-ae6e-11f5ecc8c521
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-3c17523a-29"
            Interface "qg-3c17523a-29"
                type: internal
    Bridge br-int
        Port "tape3724794-bf"
            tag: 1
            Interface "tape3724794-bf"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port "qr-121b6830-c1"
            tag: 1
            Interface "qr-121b6830-c1"
                type: internal
        Port "qvo8838a7aa-f5"
            tag: 1
            Interface "qvo8838a7aa-f5"
    ovs_version: "1.4.0+build0"



```

。。。。。わからない。

### 試行錯誤中

いじくりまわしてるうちにpingにもssh接続もできなくなってしまった。  
しかも決まってstack.shしたときに再現する。  
リブートしない限り解決できない。  
なんで。

### 仕切り直し

現在の設定を見なおした。  
ネットワークのセグメント設定がめちゃくちゃな気がしたので整理した。  
IPアドレスは16IP幅に設計して、192.168.0.100/28が入るようなネットワークを選択した。  
この `FLOATING_RANGE` は実際にはグローバルIPになるのかな。  
とりあえずQuantumはオフにした。  
これでとりあえずネット繋がるといいなあ。

[ネットワークアドレス計算フォーム（実用版）](http://www.cityjp.com/javascript/network/ipcal.html)

```
FLOATING_RANGE=192.168.0.96/28
FIXED_RANGE=10.0.0.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=eth0
ADMIN_PASSWORD=supersecret
MYSQL_PASSWORD=iheartdatabases
RABBIT_PASSWORD=flopsymopsy
SERVICE_PASSWORD=iheartksl
SERVICE_TOKEN=a91e1afd9aef8879326c

# Quantumはとりあえずオフにしておく（問題の切り分けのため）
#disable_service n-net
#enable_service q-svc
#enable_service q-agt
#enable_service q-dhcp
#enable_service q-l3
#enable_service q-meta
#enable_service neutron
## Optional, to enable tempest configuration as part of devstack
#enable_service tempest
```

ok.

やっぱりping飛ばない。。。

固定IPにしてみる。

むり。
有効化。

```
disable_service n-net
disable_service n-obj
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
```

oh god.

[devstackを利用したquantumのインストールについて - Google グループ](https://groups.google.com/forum/#!topic/openstack-ja/2QkazjDLva8)

## 参考リンク

- [Single Machine Guide - DevStack](http://devstack.org/guides/single-machine.html)
- [openstack-dev/devstack](https://github.com/openstack-dev/devstack/tree/stable/grizzly)
- [「オープンソース」を使ってみよう (第23回 DevStackでラクラク導入！ OpenStackを使ってみよう編)](http://www.ospn.jp/press/20120828no27-useit-oss.html)

### Neutron系

[quantum · irixjp/openstack-study-9 Wiki](https://github.com/irixjp/openstack-study-9/wiki/quantum)
[Running Virtual Machine Instances - OpenStack Installation Guide for Red Hat Enterprise Linux, CentOS, and Fedora  - master](http://docs.openstack.org/trunk/openstack-compute/install/yum/content/running-an-instance.html)
