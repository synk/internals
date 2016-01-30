This file attempts to document the overall structure of a queue, and
how persistence works.

このファイルはキューの全体的な構造，永続化がいかに働くかを文書化することを試みる．

Each queue is a gen_server2 process. The usual pattern of the API and
implementation being in one file is not applied; rabbit_amqqueue is
the API and rabbit_amqqueue_process is the implementation.

個々のキューはgen_server2プロセスである．通常のAPIのパターンである1ファイル1実装
は適用されない．rabbit_amqqueueがAPIでrabbit_amqqueue_processは実装になっている．

Startup
-------

The queue's supervisor initially starts the process as
rabbit_prequeue. This is a gen_server which determines whether the
process is a an HA slave or a regular queue or master (see HA
documentation), and if so whether it is starting afresh or needs to
recover. This then uses the gen_server2 "become" mechanism to become
the correct sort of process - for this document we'll deal with
rabbit_amqqueue_process for regular queues.

キューのスーパーバイザは最初にrabbit_prequeueのプロセスとしてスタートする．
これはgen_serverであり，プロセスがHAスレーブか，通常のキューか，あるいは
master（HAの文書を見よ）を決定し，新たに開始するか，リカバリすべきか．
これはgen_server2の"become"機能を使って適切な種類のプロセスに成る．
なおこの文章ではrabbit_amqqueue_processを通常のキューとして扱う．

The queue process decides for itself what it should be (rather than
having some library function that starts different types of processes)
so that it can do the right thing if it crashes and is restarted by
the supervisor - it might have been started as a master but need to
restart as a slave after crashing, for example. Or vice-versa.

そのキュープロセスはそれ自身が何になるべきかを決定する（別のタイプのプロセスを
開始するなんらかのライブラリ関数を持つのではなく）
もしそれがクラッシュしてスーパーバイザに再起動されたときに，正しい挙動をするために，
たとえば，それはマスターとして起動したがクラッシュ後はスレーブとして再起動しなければ
ならないかもしれない．逆もまたしかり

Sub-modules
-----------

The queue process probably has the most code running in it of any
process; the rabbit_amqqueue_process has had various subsystems broken
out of it into separate modules over the years. The most major such
break-out is the queue implementation API, described by the
rabbit_backing_queue behaviour.

キュープロセスは他のプロセスとくらべて最も多いコードを持っているかもしれない．
rabbit_amqqueue_processは何年もかけて分離されたモジュールに分解された様々な
サブシステムを持っている．最も主要なものは，rabbit_backing_queueビヘイビアとして
記述されたqueue implementation APIである．

The aim of the code within rabbit_amqqueue_process is therefore mainly
to take the abstract queue implementation and make it support AMQPish
features, by handling consumers, implementing features like TTL and max
length in terms of lower level APIs, and coordinating everything.

rabbit_amqqueue_processのコードの狙いは，抽象的なキューの実装とAMQP風の
機能をサポートすること，コンシューマを扱うことによって，ローレベルAPIの立場から
TTLやmax lengthなどの機能を実装し，すべてを調整することである．

Recently all the consumer-handling code was moved into
rabbit_queue_consumers.

最近すべてのコンシューマハンドリングに関するコードがrabbit_queue_consumersに
移された．

rabbit_backing_queue
--------------------

The behaviour rabbit_backing_queue (BQ) implements a Rabbit-ish queue
with persistence and so on. The module rabbit_variable_queue (VQ) is
the major implementation of this behaviour.

rabbit_backing_queue(BQ)ビヘイビアは永続化可能なRabbit風キューを実装する．
rabbit_variable_queue(VQ)はこのビヘイビアの主要な実装である．

This split was introduced with the "new" persister in 2.0.0. At the
time this was done so the old persister could be offered as a backup
(rabbit_invariable_queue) if serious bugs were found in the new
implementation. rabbit_invariable_queue is long gone but the mechanism
to configure an alternate module is still there. At various times
there have been proposals to provide alternate queue implementations
(using Mnesia, SQL etc) but this never came to anything. (One
rationale for optional e.g. SQL-based queues is that they would make
queue-browsing, atomic transactions and so on trivial, at the cost of
performance.)

この分離は2.0.0の新しい永続化機構とともに導入された．このときは，新しい実装に
深刻なバグが存在した場合に備えて古い永続化機構をバックアップ（rabbit_invariable_queue）
として提供できるようにするため．rabbit_invariable_queueは取り払われて久しいが，
代替モジュールを設定する機構はまだそのままである．なんどか代替のキュー実装が
提案されたが（MnesiaやSQLを使ったものなど）これらは実現しなかった．
(任意の1つの根拠は，SQLベースのキューは，パフォーマンスのコストを犠牲にして，
キューのブラウジングや，アトミックなトランザクションを実現する)

The BQ behaviour had a secondary use that has turned out to be
important - it provides an API where we can insert a proxy to modify
how the queue behaves by intercepting calls and deferring to
VQ. Currently there are two such proxies: rabbit_mirror_queue_master
(see HA documentation) and rabbit_priority_queue (which implements
priority queues by providing one BQ implemented in terms of several
BQs.

BQビヘイビアはには，重要なことが判明した第二の用途があった．それはcallに割り込む
ことによってキューの振る舞いを変更し，VQに委譲するためのプロキシを挿入できるよう
にするAPIを提供する．このようなプロキシにはrabbit_mirror_queue_master(HAの文書をみよ)
とrabbit_priority_queue(いくつかのBQによって実装されたBQによってpriority queue
を実装する)

rabbit_variable_queue
---------------------

So this is the meat of the queue implementation. This implements a
queue in terms of various sub-queues, with various degrees of
paged-out-ness.

そしてこれがキュー実装の根幹である．これはさまざまなサブツリーと様々なページアウト
の度合いを持つ

publish -> [q1 -> q2 -> delta -> q3 -> q4] -> consumer

q1 and q4 contain "alpha" messages, meaning messages are entirely
within RAM. q2 and q3 contain "beta" and "gamma" messages, meaning
they have metadata in RAM (message ID, position etc) and contents on
disk. Finally, delta messages are on disk only. Many of the subqueues
can be empty so that messages do not need to pass through all states
if the queue is short.

q1とq4はalphaメッセージを持つ，これらは完全にRAM上にあるメッセージである．
q2とq3はbetaとgammaメッセージであり，metadata（message ID, positionなど）
はRAM上にあって内容がディスクにある．最後に，deltaメッセージはディスクにしかない
メッセージである．サブキューは空になることがあるので，キューが短いときは
すべてのキューを通過する必要はない．

The essay at the top of rabbit_variable_queue goes into a great deal
more detail on this.

rabbt_variable_queueの冒頭の小論はこれについての詳細について述べている．

Most of the complexity of VQ deals with moving messages between the
various queues in an optimised way. The actual persistence is handled
by rabbit_queue_index (QI) and rabbit_msg_store.

VQの複雑さの大半は様々なキューの間の最適化されたメッセージの移動による．
実際の永続化はrabbit_queue_index(QI)とrabbit_msg_storeによる．

rabbit_queue_index
------------------

QI contains metadata that needs to be held per queue even if one
message is published to multiple queues - publication records with a
small amount of metadata, and delivery / acknowledgement record. In
3.5.0 the QI was extended to directly handle persistence of tiny
messages to improve performance by reducing the number of I/O ops we
do. The QI exists as "segment" files containing a log of the actions
which have taken place for an ordered segment (i.e. part) of the
queue, and an out of order journal which we write to any time anything
happens. Again, see the module for much more detail.

QIは．あるメッセージが複数のキューにパブリッシュした場合でもキューごとに保持されるべきmetadataを含む，
小さなメタデータが付随する発行記録（publication records）と，delivery/acknowledgementレコード
3.5.0ではQIはI/Oオペレーションの数を減らしてパフォーマンスを改善するために，
小さなメッセージの永続化を直接扱うように拡張された
QIは順序付けされたキューのセグメントにより起こったアクションのログを含むセグメントファイルとして存在する．
様々なタイミングで書き込んだ破損したジャーナル


Note that everything as far as this part is within the main queue
process.

この章に記載したすべてはメインキュープロセスのなかで発生する．

rabbit_msg_store
----------------

There are also two msg_store processes per broker - one for transient
messages and one for persistent ones (mainly so the transient one can
just be deleted at startup).

ブローカーには2つのmsg_storeプロセスが存在する．ひとつは揮発性のメッセージ用で，
もうひとつは永続化メッセージ用である．（揮発性のものはすぐに消すことができる）

The msg_store is a disk-based reference-counting key-value store,
storing messages in log-structured files. Again, see its module for
more details.

msg_storeはディスクベースの参照カウント，キーバリューストア，log-structuredファイルにメッセージを書き込む．
詳細はモジュールのコードをみよ

If one message is published to multiple queues, they will all submit
it to the message store, and the store will detect the non-first
requests to record the message and just increment the reference count.

もし1つのメッセージが複数のキューに書き込まれた場合，それらはすべて
メッセージストアにかきこまれる．そして最初の1回以降の書き込みリクエスト以外が
検知されて，参照カウンタだけを上げる

The message store is designed to allow clients (i.e. queues) to read
from store files directly without calling into the message store
process. Only writes go via the process. There are a number of shared
ETS tables to coordinate what's going on.

メッセージストアはメッセージストアプロセスを呼ぶことなしにクライアント（キューなど）
が直接ファイルを読むことができるように設計されている．書き込みだけがこのプロセスを
経由する．起きていることを調整するためにかなりの共有ETSテーブルが存在する．

We go to some effort to avoid unnecessary work. For example, the
message store maintains a table of "flying writes" - writes which have
been submitted by queues but not yet actioned. If a request to delete
a message is enqueued before the message is actually written, the
write is cancelled.

我々は不要な作業を避けるためにさまざまな努力をする．たとえば，メッセージストア
は"flying writes"（キューからサブミットされたがまだ実行されていない書き込み）
のテーブルを管理する．もしメッセージを削除するリクエストがメッセージが書き込まれる前に
キューイングされたら，書き込みはキャンセルされる．

The message store needs an index, from message-id to {file, offset,
etc}. This is also pluggable. The default index is implemented in ETS
and so each message has an in-memory cost.

メッセージストアは，message-idからファイル，オフセット，その他の対応づけのために
indexを必要とする．これはプラッガブルである．デフォルトのindexはETS実装され，
各メッセージはメモリ消費のコストがかかる．

The message store also needs to be garbage collected. There's an extra
process for GC (so that GC can lock some files and the message store
can concurrently serve from the rest). Within the message store, "GC"
boils down to combining together two files, both of which are known to
have over 50% messages where the ref count has gone to 0. See
rabbit_message_store_gc for more details on how that works.

メッセージストアはガベージコレクトを必要とする．GCのための特別なプロセスがある
(GCがファイルをロックして，メッセージストアが他のプロセスから守られるように)
メッセージストアの中では，GCは，参照カウントが0になったメッセージが50%を超えた
ファイルを2つ結合して集約する．
これはどう働くかについて詳細はrabbit_message_store_gcをみよ
