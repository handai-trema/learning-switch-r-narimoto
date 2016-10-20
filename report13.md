#課題レポート(マルチプルテーブルを読む)
更新日：(2016.10.20)

##課題
```
課題内容：
OpenFlow1.3版スイッチの動作を説明する．
スイッチ動作の各ステップについて，trema dump_flowsの出力を混じえながら動作を説明する．
```  

##手順  
下記の手順で動作させ，スイッチの動作を説明する．
```
スイッチ(lsw)dpid:0xabc,ポート[1-2]:host[1-2](192.168.0.[1-2])
1.host1からhost2へパケットを送信
2.host2からhost1へパケットを送信
3.もう一度1.を実行
```  

##説明  
###0.テーブルの初期化  
tremaを開始し，コントローラに対してスイッチが接続された状態(switch_readyハンドラが呼び出された後)の時のフローテーブルは次のようになっている．  
```
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=71.85s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=71.813s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=71.813s, table=0, n_packets=0, n_bytes=0, priority=1 actions=goto_table:1
cookie=0x0, duration=71.813s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=71.813s, table=1, n_packets=0, n_bytes=0, priority=1 actions=CONTROLLER:65535
```  
各行の意味は下記の通り．  
1. 宛先MACアドレスがマルチキャストアドレスの時，パケットをdropする．優先度：2（table_id:0, Filtering Table）  
2. 宛先MACアドレスがipv6マルチキャストアドレスの時，パケットをdropする．優先度：2(table_id:0, Filtering Table)  
3. goto Fowarding Table 優先度：１
4. 宛先MACアドレスがブロードキャストアドレスの時，パケットをフラッディングする．優先度：3
5. コントローラにpacket_inする．優先度：１   

###1.host1からhost2へパケットを送信  
実行結果は下記の通り．  
```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=128.687s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=128.65s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=128.65s, table=0, n_packets=1, n_bytes=42, priority=1 actions=goto_table:1
cookie=0x0, duration=128.65s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=128.65s, table=1, n_packets=1, n_bytes=42, priority=1 actions=CONTROLLER:65535
```  
フローテーブルに変化はない．  
これは，host2宛のパケットはマルチキャストでもフラッディングでもない為，フローテーブルのエントリのほとんどを通過し，最終的にコントローラに対してpacket_inが発生するためである．
この時コントローラはfdbに対してhost1の情報を記録し，packet_outにてフラッディングを行う．
故に，フローテーブルに対して変化が起こらない．

###2.host2からhost1へパケットを送信  
実行結果は下記の通り．  
```
$ ./bin/trema send_packets --source host2 --dest host1
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=165.032s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=164.995s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=164.995s, table=0, n_packets=2, n_bytes=84, priority=1 actions=goto_table:1
cookie=0x0, duration=164.995s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=24.36s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=54:0d:70:53:a2:b0,dl_dst=41:2a:6b:a9:8e:77 actions=output:1
cookie=0x0, duration=164.995s, table=1, n_packets=2, n_bytes=84, priority=1 actions=CONTROLLER:65535
```  
Fowarding Table(table_id:1)に次のエントリが追加されている．  
```
cookie=0x0, duration=24.36s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=54:0d:70:53:a2:b0,dl_dst=41:2a:6b:a9:8e:77 actions=output:1
```  
これは，送信元ポートが2,送信元MACアドレスが54:0d:70:53:a2:b0，宛先MACアドレスが41:2a:6b:a9:8e:77の時，パケットをポート1へpacket_outする，優先度2の処理となっている．  
また，[.lib/learning_switch13.rb](https://github.com/handai-trema/learning-switch-r-narimoto/blob/master/lib/learning_switch13.rb)にて次のように実装されている．  
```ruby
def add_forwarding_flow_and_packet_out(packet_in)
  port_no = @fdb.lookup(packet_in.destination_mac)
  add_forwarding_flow_entry(packet_in, port_no) if port_no
  packet_out(packet_in, port_no || :flood)
end
```  
fdbに登録されているMACアドレスの場合，port_noにポート番号が入る．そして，ポート番号が入っている場合にはFowarding Tableにそのパケットに関するエントリを追加し，packet_outする．  
Fowarding Tableにエントリを追加するadd_forwarding_flow_entryメソッドの実装は下記の通り．  
```ruby
def add_forwarding_flow_entry(packet_in, port_no)
  send_flow_mod_add(
    packet_in.datapath_id,
    table_id: FORWARDING_TABLE_ID,
    idle_timeout: AGING_TIME,
    priority: 2,
    match: Match.new(in_port: packet_in.in_port,
                     destination_mac_address: packet_in.destination_mac,
                     source_mac_address: packet_in.source_mac),
    instructions: Apply.new(SendOutPort.new(port_no))
  )
end
```  
Forwarding Tableに，優先度２で発生したpacket_inの送信元，宛先MACアドレス及び送信元ポートのマッチング条件にて，packet_outをするエントリを追加するflow_modメッセージを送っている．  

###3.もう一度1.を実行  
ここまでの結果を踏まえ，もう一度1.を実行すると，「送信元ポート１，送信元MACアドレスhost1，宛先MACアドレスhost2，優先度２，table_id1(Fowarding Table)，宛先ポート2」のエントリが追加されると予測される．  
実行結果は下記の通り．  
```
$ ./bin/trema send_packets --source host1 --dest host2
$ ./bin/trema dump_flows lsw
cookie=0x0, duration=185.423s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=01:00:00:00:00:00/ff:00:00:00:00:00 actions=drop
cookie=0x0, duration=185.386s, table=0, n_packets=0, n_bytes=0, priority=2,dl_dst=33:33:00:00:00:00/ff:ff:00:00:00:00 actions=drop
cookie=0x0, duration=185.386s, table=0, n_packets=3, n_bytes=126, priority=1 actions=goto_table:1
cookie=0x0, duration=185.386s, table=1, n_packets=0, n_bytes=0, priority=3,dl_dst=ff:ff:ff:ff:ff:ff actions=FLOOD
cookie=0x0, duration=44.751s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=2,dl_src=54:0d:70:53:a2:b0,dl_dst=41:2a:6b:a9:8e:77 actions=output:1
cookie=0x0, duration=7.2s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=1,dl_src=41:2a:6b:a9:8e:77,dl_dst=54:0d:70:53:a2:b0 actions=output:2
cookie=0x0, duration=185.386s, table=1, n_packets=3, n_bytes=126, priority=1 actions=CONTROLLER:65535
```  
結果より，以下のエントリが追加されていることがわかる．  
```
cookie=0x0, duration=7.2s, table=1, n_packets=0, n_bytes=0, idle_timeout=180, priority=2,in_port=1,dl_src=41:2a:6b:a9:8e:77,dl_dst=54:0d:70:53:a2:b0 actions=output:2
```  
これは，予測通り「送信元ポート１，送信元MACアドレスhost1，宛先MACアドレスhost2，優先度２，table_id1(Fowarding Table)，宛先ポート2」のエントリである．
