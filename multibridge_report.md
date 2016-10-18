学籍番号:33E16006
名前：銀杏一輝

#複数スイッチ対応版ラーニングスイッチ

#コードの説明

```
def switch_ready(datapath_id)
    @fdbs[datapath_id] = FDB.new
  end
```

learning_switch.rbと違い、各スイッチを捕捉し、各スイッチのFDBを作成。  

```
def packet_in(datapath_id, packet_in)
    return if packet_in.destination_mac.reserved?
    @fdbs.fetch(datapath_id).learn(packet_in.source_mac, packet_in.in_port)
    flow_mod_and_packet_out packet_in
  end
```

該当スイッチでのpacket_inの処理  

```
def flow_mod_and_packet_out(packet_in)
    port_no = @fdbs.fetch(packet_in.dpid).lookup(packet_in.destination_mac)
    flow_mod(packet_in, port_no) if port_no
    packet_out(packet_in, port_no || :flood)
  end
```

Flow_ModとPacket_outを同時に行う。該当するスイッチで、パケットの宛先MACアドレスとFDBから、パケットを出力するポート番号を調べる。  
宛先ポートが見つかった場合、以降は同じパケットは同様に転送せよ、というフローエントリをスイッチに書き込む。  
宛先ポートが見つからなかった場合、フラッディングを行う。  


#動作の説明

![fig1](https://github.com/handai-trema/learning-switch-Kazuki-Ginnan/blob/develop/topology.jpg)

図のようなトポロジの動作例を示す

```
./bin/trema send_packets --source host1 --dest host2
```
により、ホスト1からホスト2へパケットを送る。  
packet_in(datapath_id, packet_in)がlsw1,host1からhost2へのパケットを引数に呼び出される。
flow_mod_and_packet_out(packet_in)の関数が呼び出され、lsw1で宛先をさがすために、フラッディングする。


```
 trema dump_flows lsw1
```
により、フローテーブルを確認すると、何も表示されていないことが確認できる。

```
./bin/trema send_packets --source host2 --dest host1
````


を実行すると、
packet_in(datapath_id, packet_in)がlsw1,host2からhost1へのパケットを引数に呼び出される。
flow_mod_and_packet_out(packet_in)が呼び出され、宛先ポートがわかっているので、flow_modを行い、フローエントリに書き込む。

```
trema dump_flows lsw1
```
をすると、

```
cookie=0x0, duration=4.048s, table=0, n_packets=0, n_bytes=0, idle_age=4, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=51:aa:21:30:f5:6c,dl_dst=6e:d0:67:03:8a:3d,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
```

フローテーブルに登録されていることがわかる。

また、
```
trema show_stats host1
```

を実行すると、
```
Packets sent:
  192.168.0.1 -> 192.168.0.2 = 1 packet
Packets received:
  192.168.0.2 -> 192.168.0.1 = 1 packet
```
と、パケットが到着していることが確認できる。



また、

```
./bin/trema send_packets --source host1 --dest host2
```
をもう一度実行して、


```
trema dump_flows lsw1
```

```
cookie=0x0, duration=883.742s, table=0, n_packets=1, n_bytes=42, idle_age=36, priority=65535,udp,in_port=2,vlan_tci=0x0000,dl_src=51:aa:21:30:f5:6c,dl_dst=6e:d0:67:03:8a:3d,nw_src=192.168.0.2,nw_dst=192.168.0.1,nw_tos=0,tp_src=0,tp_dst=0 actions=output:1
cookie=0x0, duration=3.156s, table=0, n_packets=0, n_bytes=0, idle_age=3, priority=65535,udp,in_port=1,vlan_tci=0x0000,dl_src=6e:d0:67:03:8a:3d,dl_dst=51:aa:21:30:f5:6c,nw_src=192.168.0.1,nw_dst=192.168.0.2,nw_tos=0,tp_src=0,tp_dst=0 actions=output:2
```
と、出力ポートが2の場合のフローテーブルが記録されていることも確認できた。

host3, host4の間で行っても同様の結果が確認できる。

また、
```
./bin/trema send_packets --source host2 --dest host3
```

を実行して、
```
trema show_stats host3
```
とすると、

```
Packets sent:
  192.168.0.3 -> 192.168.0.4 = 1 packet
Packets received:
  192.168.0.4 -> 192.168.0.3 = 1 packet
```

となり、host2とhost3が同じスイッチ内に接続されていないため、パケットが届いていないことがわかる。


