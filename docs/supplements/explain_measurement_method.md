# End-to-Endレイテンシの測定方法
CARETでは通信レイテンシ（Pub/Subレイテンシ）とノードレイテンシを結びつけてE2Eレイテンシを算出しています。
ここでは通信レイテンシとノードレイテンシの算出方法について説明します。


## 通信レイテンシとノードレイテンシの算出方法
以下は通信レイテンシとノードレイテンシを表した図です。

![callback_and_node_latency](../imgs/callback_and_node_latency.png)


### 通信レイテンシの算出方法
通信レイテンシはコールバックのpublishの時刻と、後続のコールバックが開始した時刻の差分で計算しています。

#### プロセス内通信
次の表はプロセス内通信のトレースに必要なトレースポイントと、そのトレースポイントの引数（トレースデータとして出力されるもの）です。※簡易版


| トレースポイント名 | 引数1 | 引数2 | 時刻 |
|-|-|-|-|
| rclcpp_intra_publish | publisher_handle_arg | <span style="color: red; ">message_arg</span> | time1 |
| dispatch_intra_process_subscription_callback | <span style="color: red; ">message_arg</span> | <span style="color: green; ">callback_arg</span> | time2 |
| callback_start | <span style="color: green; ">callback_arg</span> | is_intra_process | time3 |

※[引数についてはこちらを参照](https://tier4.github.io/CARET_doc/design/tracepoint_definition/)

message_argとcallback_argにはメッセージのアドレス、コールバックのアドレスが格納されています。
rclcpp_intra_publisherとdispatch_intra_process_subscription_callbackは同じmessage_argを持つトレースポイント同士で紐づけ、dispatch_intra_process_subscription_callbackとcallback_startは同じcallback_argを持つトレースポイント同士で紐づけることにより、下記のような表を作成します。
最後に`callback_start - rclcpp_intra_publish`で**プロセス内通信のレイテンシ**を算出しています。

|idx| rclcpp_intra_publish | dispatch_intra_process_subscription_callback | callback_start |
|-|-|-|-|
|0| time1 | time2 | time3 |
|1| ... | ... | ... |


#### プロセス間通信
以下はプロセス間通信のトレースポイントとその引数です。

| トレースポイント名 | 引数1 | 引数2 | 引数3 | 時刻 |
|-|-|-|-|-|
| rclcpp_publish | publisher_handle_arg | <span style="color: red; ">message_arg</span>@1   |   | time1 |
| rcl_publish | publisher_handle_arg | <span style="color: red; ">message_arg</span>@1   |   | time2 |
| dds_write | publisher_handle_arg | <span style="color: red; ">message_arg</span>@1   |   | time3 |
| dds_bind_addr_to_stamp | <span style="color: red; ">message_arg</span>@1 | <span style="color: green; ">stamp_arg</span> |  | time4 |
| dispatch_subscription_callback | messsage_arg@2 | <span style="color: green; ">stamp_arg</span> | <span style="color: blue; ">callback_arg</span> | time5 |
| callback_start | <span style="color: blue; ">callback_arg</span> | is_intra_process |  | time6 |


プロセス内通信と同様に同じ引数を持つトレースポイント同士を紐づけていき、publishからcallback_startまでの表を作成します（下記表）。
1行が1つのプロセス間通信のチェーンを表し、`callback_start - rclcpp_publish` にて**プロセス間通信のレイテンシ**を算出します。

| idx | rclcpp_publish | rcl_publish | dds_write | dds_bind_addr_to_stamp | dispatch_subscription_callback | callback_start |
|-|-|-|-|-|-|-|
| 0 | time1 | time2 | time3 | time4 | time5 | time6 |
| 1 | ... | ... | ... | ... | ... | ...　|


## ノードレイテンシの算出方法
ノードレイテンシの算出方法は2つあります。
それぞれの手法・長所・短所を説明します。

### コールバックチェーンの利用
コールバックレイテンシの測定は、同じコールバックアドレスを持つcallback_startとrclcpp_publishの差分を使って算出します。
callback_startとrclcpp_publishの紐づけは、rclcpp_publishから見て一番近いcallback_startと紐づけます。

| idx | callback_start (cb_A) [s] | rclcpp_publish (cb_A) [s] | callback_end (cb_A) [s] | callback_arg (cb_A) |
|-|-|-|-|-|
| 0 | 0 | 3 | 4 | 0x1000 |
| 1 | 2 | 5 | 6 | 0x1000 |
| 2 | 4 | 7 | 8 | 0x1000 |
| ... | ... | ... | ... | ... |

| idx | callback_start (cb_B) [s] | rclcpp_publish (cb_B) [s] | callback_end (cb_B) [s] | callback_arg (cb_B) |
|-|-|-|-|-|
| 0 | 4 | 8 | 9 | 0x2000 |
| 1 | 8 | 12 | 13 | 0x2000 |
| 2 | 10 | 14 | 15 | 0x2000 |

上記表のようにコールバックA・B（cb_A・cb_B）が存在し、A→Bと処理が続く時、cb_Aのcallback_endとcb_Bのcallback_startを結び付けて表を作ります。
最後のcallbackだけはpublishの時の時刻を採用し、下記表のように一つのテーブルにします。

| idx | callback_start (cb_A) [s] | callback_end (cb_A) [s] | callback_start (cb_B) [s] | rclcpp_publish (cb_B) [s] |
|-|-|-|-|-|
| 0 | 0 | 4 | 4 | 8 |
| 1 | 2 | 6 | Lost | Lost |
| 2 | 4 | 8 | 8 | 12 |
| ... | ... | ... | ... | ... |

上記のようにコールバックチェーンをつなぎ、```rclcpp_publish (cb_X) - callback_start (cb_Y)```で**ノードレイテンシ**を算出します。

> ※ノード内にコールバックが１つの場合、X, Yは同じものを指します。複数コールバックがある場合は、Xが最後・Yが最初のコールバックを指します。





**想定**

 - コールバック間のキューサイズ１を想定
 - 厳密に測定できるのはSingle Threaded Executorのみ

**長所**

 - ソースコードへの変更が不要
 - 任意のメッセージ型に適用可能

**短所**

 - キューサイズが大きい時、レイテンシが小さめに出るケースがある。
 - multi threaded executor では、レイテンシが大きめに出るケースがある。
 - 実装を読み解くのが難しい


### 入出力のヘッダータイムスタンプのマッチング
入出力のヘッダーでマッチングを取り、入力トピックと出力トピックの時刻の差分からノードレイテンシを算出します。
以下はこのレイテンシ算出方法を表した図です。

<div align="center"><img src="https://user-images.githubusercontent.com/55824710/146482653-376d72d8-d63c-45a7-b67e-86dc9bb6be8b.png" width="600px">

 
**想定**

 - 入出力のタイムスタンプの値でマッチングが取れること（値を書き換えずに、そのまま publish していること）
 - 入出力ともに header をもっていること

**長所**

 - ソースコードへの変更が不要
 - キューサイズに依らない

**短所**

 - ヘッダーが必要
 - publish 前に stamp=現在時刻としているケースは対応不可

