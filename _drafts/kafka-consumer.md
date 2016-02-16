---
title: KafkaConsumer 0.9
categories:
  - 
---

Group Membership APIとOffset Commit/Fetch APIを使ったコンシューマー実装

メモ：
オフセットのメタデータは、コミットする際に指定して取得時にも付いてくるっぽい。コメント的な？

# コンシューマーは起動時に、ブローカーのクラスタのどのサーバーでも構わないので、１つのサーバーに接続を試みる.

ラウンドロビンか？

# 接続できた場合、トピックのメタデータを取得するリクエストを投げる。その情報を受け取り保持しておく

https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol
Metadata API
  このメタデータは下記の応答を返す
    トピックは存在するか
    トピックにはどれくらいのパーティションが存在するのか
    トピックのパーティションのリーダーはどのブローカーか
    (すべての？トピックに関係する？)ブローカーのホストとポート

トピックの指定がなければ全てのトピック情報を返す

Topic Metadata Request
Metadata Response

# 接続できない場合は、違うブローカーから取得

コンシューマー起動時に渡したブローカのなかから

# ダメだった場合、もっかい成功するまで最初からやるか終了

# 自身のコンシューマーグループのコーディネーターとなるブローカー情報を取得して、保持

コーディネーターとは、コンシューマーグループのオフセットを管理する特定のブローカーをそう呼ぶ

Offset Commit/Fetch API
0.8.2から出来た新しいAPI

GroupCoordinatorRequest => GroupId

GroupCoordinatorResponse => ErrorCode CoordinatorId CoordinatorHost CoordinatorPort

オフセットはこのブローカーに投げる

# コーディネーターに接続する



-------- 以下、rebalance処理

# コーディネーターに対して自身を登録する(JoinGroupRequest)

JoinGroupRequest => GroupId SessionTimeout MemberId ProtocolType GroupProtocols

メンバーIDは初回接続時はからでよいが、rebalance等で再接続時には発行されたメンバーIDで接続すること

JoinGroupResponse => ErrorCode GenerationId GroupProtocol LeaderId MemberId Members

# グループリーダーに対してステートの同期をおこなう？(SyncGroup Request)

メンバーのアサインと、担当パーティション情報をグループ全体で同期する
リーダーだけが全員のアサイン情報を計算して送信するっぽい？

コーディネーターはリーダーからのsyncgroupが来るまで rebalance is in progress. のエラーを返すようになるらしい。リーダーが投げるタイミングはどうやって決める？

もしかして、リーダーからリクエストが来るまでリーダーじゃないコンシューマーのレスポンスをブロックしてる？

joinは全consumerが揃うまでblock
 ・generationIDが同じconsumerを待ってるっぽい(タイムアウトした後にjoinを投げたら、generationIDが古いってでた)
syncはリーダーがアサイン情報を投げてくるまでその他はブロックって言う流れだった

つまりjoinGroupを通り過ぎた後のjoinは全てrebalance対象


JoinGroup時のプロトコルタイプがconsumerの場合だけMemberAssignmentが有効らしい

SyncGroupRequest => GroupId GenerationId MemberId GroupAssignment

SyncGroupResponse => ErrorCode MemberAssignment

# 担当パーティションのオフセットを取得する

OffsetFetchRequest => ConsumerGroup [TopicName [Partition]]

OffsetFetchResponse => [TopicName [Partition Offset Metadata ErrorCode]]

# 担当パーティションの指定オフセットから取得する


--------- 以下 heartbeat

# 一定間隔でコーディネーターにハートビートを投げる

HeartbeatRequest => GroupId GenerationId MemberId

設定に指定している時間(session timeout)ハートビートがなければそのコンシューマーはkickされる

誰かが抜けたとか、ブローカーの数が変わったとかだと多分ハートビートがREBALANCE_IN_PROGRESSのエラーコードを返すから、そこでリバランスを実行するんだと思う
