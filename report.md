#課題レポート(複数スイッチ対応版 ラーニングスイッチ)
更新日：(2016.10.17)  

###課題
```
課題内容：
複数スイッチに対応したラーニングスイッチ(multi_learning_switch.rb)の動作を説明する．
```

####ソースコード  
* start\(\)  
以下のように，FDB\(ForwardingDataBase\)を連想配列として扱うことで，それぞれのスイッチに対応するFDBを持つようにしている．こうすることで，複数スイッチへ対応している．
```
@fdbs = {}
```

* switch_ready\(\)  
コントローラーにスイッチが接続された際に呼び出されるハンドラである．
ここで，以下のように接続されたスイッチのデータパスIDを用いて連想配列@fdbsに新たなFDBとして追加している．こうすることで，スイッチごとに独立したFDBへアクセスするようになっている．
```
@fdbs[datapath_id] = FDB.new
```

* packet_in\(\)  
packet_inが発生した際， 各スイッチに対応したFDBを用いる必要があるため，fetch()を用いてpacket_inの送信元のデータパスIDのFDBを用いるようにしている．

* age_fdbs\(\)  
FDBのエージングも，管理しているすべてのFDBに対して行う必要があるため，連想配列@fdbsのeach_value\(\)を用いてそれぞれのFDBのエージングを行っている．

* flow_mod_and_packet_out\(\)  
packet_inのあったスイッチに対応するFDBを参照（@fdbs.fetch\(\)）し，宛先MACアドレスに対応するエントリがあるか調べる．ある場合はport_noにポート番号が格納される．その後はエントリの有無（port_noの中身の有無）に応じて処理を変えている．具体的には，ポート番号が見つかった場合にはflow_modメッセージを生成し，フローテーブルにエントリを追加する．続くpacket_out処理では，有る場合はそのままそのポート番号に対してpacket_out，ない場合には:floodオプションをつけることでフラッディングを行うようになっている．

* flow_mod\(\),packet_out\(\)  
これら２つのメソッドについては，複数スイッチ対応のために特別に変更点は無い．

####動作確認  
以下に示す環境にて複数スイッチ対応の確認を行った．
[trema.multi.conf](https://github.com/handai-trema/learning-switch-r-narimoto/blob/master/trema.multi.conf)
```
vswitch('lsw1') { datapath_id 0x1 }
vswitch('lsw2') { datapath_id 0x2 }

vhost('host1-1')
vhost('host1-2')
vhost('host2-1')
vhost('host2-2')

link 'lsw1', 'host1-1'
link 'lsw1', 'host1-2'
link 'lsw2', 'host2-1'
link 'lsw2', 'host2-2'
```
イメージ図：
![connection](https://raw.githubusercontent.com/handai-trema/learning-switch-r-narimoto/master/report/connection.png)

* テスト：host1-1，host1-2間，host2-1,host2-2間，host1-1，host2-1間の送受信  

手順:
```
1.host1-1からhost1-2へパケットを送信
2.host1-2からhost1-1へパケットを送信
3.もう一度1.を実行
4.1から3と同様のことをhost2-1とhost2-2に対して行う
5.host1-1からhost2-1へパケットを送信
※各ステップごとに，パケットの送信状況及びフローテーブルの状態を確認する．
```

####1.host1-1からhost1-2へパケットを送信  
実行結果を以下に示す．
```
$./bin/trema send_packets --source host1-1 --dest host1-2
$ ./bin/trema show_stats host1-1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ ./bin/trema show_stats host1-2
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ ./bin/trema dump_flows lsw1

$
```
イメージ図:
![step1](https://raw.githubusercontent.com/handai-trema/learning-switch-r-narimoto/master/report/1.png)
host1-1\(192.168.0.1\)からhost1-2\(192.168.0.2\)へパケットが送信できていること，FDBにはまだ何も登録されていないことがわかる．これは，host1-1がスイッチlsw1に対してパケットを送信するが，フローテーブルにはまだ何も入っていない為，packet_inが発生するが，FDBにもまだ何も入っていない為，FDBにエントリが追加されるのみでフローテーブルには追加されない（flow_mod）メッセージは発生しない.  

####2.host1-2からhost1-1へパケット送信  
実行結果を以下に示す．
```
$ ./bin/trema send_packets --source host1-2 --dest host1-1
$ ./bin/trema show_stats host1-1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host1-2
Packets sent:
  192.168.0.2 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 1 packet
$ ./bin/trema dump_flows lsw1
cookie=0x0, duration=11.735s, table=0, n_packets=0, n_bytes=0, idle_age=11, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=e5:f2:3d:29:b4:f0,dl_dst=83:d2:5a:52:30:fe,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
$
```
イメージ図:
![step2](https://raw.githubusercontent.com/handai-trema/learning-switch-r-narimoto/master/report/2.png)
1.とは逆方向に対してもパケットが送信できていることが確認できる．また，フローテーブルにhost1-2からhost1-1へのエントリが追加されていることがわかる．host1-2がlsw1に対してhost1-1に対するパケットを送信した際はエントリがまだ無い為，packet_inが発生する．この時，FDBにはhost1-1の情報が入っている為lsw1に対してhost1-2からhost1-1へパケットを送信す為のflow_modメッセージが生成される．そのため，このような結果となる．  

####3.host1-1からhost1-2へパケットをもう１度送信  
実行結果を以下に示す．
```
$ ./bin/trema send_packets --source host1-1 --dest host1-2
$ ./bin/trema show_stats host1-1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 2 packets
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host1-2
Packets sent:
  192.168.0.2 -> 192.168.0.1 = 1 packet
Packets received:
  192.168.0.1 -> 192.168.0.2 = 2 packets
$ ./bin/trema dump_flows lsw1
cookie=0x0, duration=195.135s, table=0, n_packets=0, n_bytes=0, idle_age=195, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=e5:f2:3d:29:b4:f0,dl_dst=83:d2:5a:52:30:fe,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=7.488s, table=0, n_packets=0, n_bytes=0, idle_age=7, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=83:d2:5a:52:30:fe,dl_dst=e5:f2:3d:29:b4:f0,nw_src=192.168.0.1,nw_dst=192.168.0.2,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$
```
イメージ図:
![step3](https://raw.githubusercontent.com/handai-trema/learning-switch-r-narimoto/master/report/3.png)
パケットが正しく送受信できていること，フローテーブルにhost1-1からhost1-2へのエントリが追加されていることが確認できる．  

####4.1から3と同様のことをhost2-1とhost2-2に対して行う  
実行結果を以下に示す．
```
$ ./bin/trema send_packets --source host2-1 --dest host2-2
$ ./bin/trema show_stats host2-1
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
$ ./bin/trema show_stats host2-2
Packets received:
  192.168.0.3 -> 192.168.0.4 = 1 packet
$ ./bin/trema dump_flows lsw1
cookie=0x0, duration=326.052s, table=0, n_packets=0, n_bytes=0, idle_age=326, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=e5:f2:3d:29:b4:f0,dl_dst=83:d2:5a:52:30:fe,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=138.405s, table=0, n_packets=0, n_bytes=0, idle_age=138, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=83:d2:5a:52:30:fe,dl_dst=e5:f2:3d:29:b4:f0,nw_src=192.168.0.1,nw_dst=192.168.0.2,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$ ./bin/trema dump_flows lsw2

$./bin/trema send_packets --source host2-2 --dest host2-1
$ ./bin/trema show_stats host2-1
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
Packets received:
  192.168.0.4 -> 192.168.0.3 = 1 packet
$ ./bin/trema show_stats host2-2
Packets sent:
  192.168.0.4 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.3 -> 192.168.0.4 = 1 packet
$ ./bin/trema dump_flows lsw1
cookie=0x0, duration=349.952s, table=0, n_packets=0, n_bytes=0, idle_age=349, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=e5:f2:3d:29:b4:f0,dl_dst=83:d2:5a:52:30:fe,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=162.305s, table=0, n_packets=0, n_bytes=0, idle_age=162, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=83:d2:5a:52:30:fe,dl_dst=e5:f2:3d:29:b4:f0,nw_src=192.168.0.1,nw_dst=192.168.0.2,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$ ./bin/trema dump_flows lsw2
cookie=0x0, duration=11.744s, table=0, n_packets=0, n_bytes=0, idle_age=11, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=cf:c8:22:73:04:e2,dl_dst=55:fe:3c:a1:f9:5b,nw_src=192.168.0.4,nw_dst=192.168.0.3,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
$ ./bin/trema send_packets --source host2-1 --dest host2-2
$ ./bin/trema show_stats host2-1
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 2 packets
Packets received:
  192.168.0.4 -> 192.168.0.3 = 1 packet
$ ./bin/trema show_stats host2-2
Packets sent:
  192.168.0.4 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.3 -> 192.168.0.4 = 2 packets
$ ./bin/trema dump_flows lsw1
cookie=0x0, duration=370.732s, table=0, n_packets=0, n_bytes=0, idle_age=370, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=e5:f2:3d:29:b4:f0,dl_dst=83:d2:5a:52:30:fe,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=183.085s, table=0, n_packets=0, n_bytes=0, idle_age=183, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=83:d2:5a:52:30:fe,dl_dst=e5:f2:3d:29:b4:f0,nw_src=192.168.0.1,nw_dst=192.168.0.2,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$ ./bin/trema dump_flows lsw2
cookie=0x0, duration=33.593s, table=0, n_packets=0, n_bytes=0, idle_age=33, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=cf:c8:22:73:04:e2,dl_dst=55:fe:3c:a1:f9:5b,nw_src=192.168.0.4,nw_dst=192.168.0.3,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=15.848s, table=0, n_packets=0, n_bytes=0, idle_age=15, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=55:fe:3c:a1:f9:5b,dl_dst=cf:c8:22:73:04:e2,nw_src=192.168.0.3,nw_dst=192.168.0.4,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$
```
イメージ図:
![step4](https://raw.githubusercontent.com/handai-trema/learning-switch-r-narimoto/master/report/4.png)
host1-1,host1-2の時と同様の結果が得られ，正しく実行できていることがわかる．また，lsw1に対して変更が加えられていないことも確認し，複数スイッチを独立して管理できていることを確認した．  

####5.host1-1からhost2-1へパケットを送信  
FDBがスイッチごとに独立して管理できていることを更に確認するため，確認を行った．結果を以下に示す．
```
$ ./bin/trema send_packets --source host1-1 --dest host2-1
$ ./bin/trema show_stats host1-1
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 2 packets
  192.168.0.1 -> 192.168.0.3 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
$ ./bin/trema show_stats host2-1
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 2 packets
Packets received:
  192.168.0.4 -> 192.168.0.3 = 1 packet
$ ./bin/trema dump_flows lsw1
cookie=0x0, duration=713.692s, table=0, n_packets=0, n_bytes=0, idle_age=713, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=e5:f2:3d:29:b4:f0,dl_dst=83:d2:5a:52:30:fe,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=526.045s, table=0, n_packets=0, n_bytes=0, idle_age=526, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=83:d2:5a:52:30:fe,dl_dst=e5:f2:3d:29:b4:f0,nw_src=192.168.0.1,nw_dst=192.168.0.2,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$ ./bin/trema dump_flows lsw2
cookie=0x0, duration=374.339s, table=0, n_packets=0, n_bytes=0, idle_age=374, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=cf:c8:22:73:04:e2,dl_dst=55:fe:3c:a1:f9:5b,nw_src=192.168.0.4,nw_dst=192.168.0.3,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=356.594s, table=0, n_packets=0, n_bytes=0, idle_age=356, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=55:fe:3c:a1:f9:5b,dl_dst=cf:c8:22:73:04:e2,nw_src=192.168.0.3,nw_dst=192.168.0.4,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
$
```
イメージ図:
![step5](https://raw.githubusercontent.com/handai-trema/learning-switch-r-narimoto/master/report/5.png)
host1-1からhost2-1へ送信は行っているが，異なるスイッチに接続しており，スイッチ間のリンクも無いのでhost2-1が受信できていないという正しい結果が得られた．また，それぞれのスイッチにおけるフローテーブルも変化がなく，FDBが正しくスイッチごとに独立管理できていることを確認した．
