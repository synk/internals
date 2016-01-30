# Publishing Messages into RabbitMQ #

One of the best ways to cover the various parts of RabbitMQ's
architecture is to see what happens when a message gets published into
the broker. In this document we are going to visit the different
subsystems a message crosses inside the broker. Let's start by
`rabbit_reader`.

The `rabbit_reader` process is the one that takes care of reading data
from the network and forwarding it to the respective channel
process. Messages get into the channel when the reader calls the
function `rabbit_channel:do_flow/3`, this function will call the
`credit_flow` module to track that a message was received from the
reader, so it could eventually throttle down the reader in case the
message publisher is sending more messages in than the amount the
broker can handle at a particular time. Read more about Credit Flow
[here](./credit_flow.md). More information about the Reader process
can be found in the
[Networking and Connections guide](./networking_and_connections.md#rabbit_reader).

## Arriving into the Channel Process ##

Once Credit Flow is accounted for, then the `do_flow/3` function will
issue an asynchronous `gen_server:cast/2` into the channel process
passing in this Erlang message: `{method, Method, Content,
flow}`. There we have the AMQP `Method`, then method `Content`, and
the atom `flow` indicating the channel that credit flow is in use.

ひとたびCredit Flowが処理されると、`do_flow/3`はチャンネルプロセスに対して
Erlang message:`{method, Method, Content, flow}`と共に非同期的に
`gen_server:cast/2`を呼び出す。
これらは、AMQPの`Method`、及びその`Content`、そしてチャンネルがcredit flowを
使用中かどうかを示すatomである`flow`である。

When the cast reaches the `handle_cast/2` function inside the channel
module, we are finally inside the channel process memory and execution
path. If `flow` was in use, as is the case here, then the channel will
issue a `credit_flow:ack/1` to the reader process. Then the AMQP
method that's being processed will be passed to the
[Interceptor](./interceptors.md) defined for the channel, in case
there are any. After the Interceptors are done processing the AMQP
method, then the channel process will continue processing the method,
in our case the function `handle_method/3` will be called, with a
`basic.publish` record.

このキャストがチャンネルモジュールの`handle_cast/2`に到達すると、
チャンネルプロセスメモリと実行パスの中にいることになる。
`flow`が使用中の場合、チャンネルは`credit_flow:ack/1`をreaderプロセスに対して発行する。
そして処理中のAMQPメソッドが、そのチャンネルに定義されたInterceptorに渡される。
Interceptorの処理が終わったら、チャンネルプロセスは処理を続け、
ここでは`handle_method/3`が`basic.publish`レコードと共に呼ばれる。

## Inside basic.publish ##

basic.publish works by receiving an AMQP message, an Exchange and a
Routing Key, and it will use the exchange to route the message to one
or various queues, based on the routing key. Let's see how's that
accomplished.

basic.publishはAMQP message、Exchange、Routing Keyを受け取ることによって動作する。
exchangeは、routing keyに基づいてメッセージを一つから複数のキューにルートするために使われる。
これがどのように実現されるか見ていく。

The first the function does is to check the size of the message since
RabbitMQ has an upper limit of 2GB for messages.

この関数が最初に行うのはメッセージのサイズをチェックすることである。
RabbitMQはメッセージについて2GBの上限を設けているからである。

Then the function needs to build the resource record for the
Exchange. Exchanges and Queues are represented internally with a
resource record that keeps track of the name, and the vhost where the
exchange or queue was declared. The type declaration record looks like
this:

```erlang
#resource{virtual_host :: VirtualHost,
          kind         :: Kind,
          name         :: Name}
```

そして、Exchangeに対してresource recordをつくる。
ExchangeとQueueは内部的にはresouce recordとして表現される。
これらは名前と、そのexchangeまたはqueueが定義されたvhostを保持する。

So if a message was published to the default vhost to an exchange
called `"my_exchange"`, we will end up with the following record:

メッセージがデフォルトのvhostの`my_exchange`にpublishされた場合、
次のようなレコードになる

```erlang
#resource{virtual_host = <<"/">>
          kind         = exchange,
          name         = <<"my_exchange">>}
```

Resources like that one are used everywhere in RabbitMQ, so it's a
good idea to study their parts in the
[rabbit_types](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_types.erl)
module where this declarations are defined.

そのようなリソースはRabbitMQのいたるところで使用される。

Once we have the exchange record, `basic.publish` will use it to see
if the user publishing the message has write permissions to this
particular exchange by calling the function
`check_write_permitted/2`. Read more about the different kind of
permissions here:
[access-control](https://www.rabbitmq.com/access-control.html)

Exchangeレコードを用意したら、`basic.puhlish`はそれを使って`check_write_permitted/2`
を呼び、ユーザによるパブリッシュがそののexchangeに対して書き込み権限を持っているか確認する。

If the user does have permission to publish messages to this exchange,
then the channel will query the Mnesia database trying to find out if
the exchange actually exists, so the function
`rabbit_exchange:lookup_or_die/1` is called in order to retrieve the
actual exchange record from the database, if the exchange is not found,
then a channel error is raised by `lookup_or_die/1`. Keep in mind that
one thing is the exchange resource we mentioned above, and another
much different is the exchange record stored in mnesia. The latter
holds up much more information about the actual exchange, like it's
type for example (direct, fanout, topic, etc). Here's the exchange
record definition from `rabbit.hrl`:

ユーザがそのexchangeに対してメッセージを発行する権限を持っている場合、
exchangeが実際に存在するか確認するためにMnesiaデータベースにクエリを行う。
データベースから実際のexchangeレコードを取り出すために`rabbit_exchange:lookup_or_die/1`が呼び出される。
もしexchangeが存在しなければ、`lookup_or_die/1`からチャンネルエラーが発生する。
なお前述のexchange resourceとmnesiaに格納されたexchange resourceは大きく異なる。
後者は実際のexchangeについての情報（direct, fanout, topic, etc)をより多く持つ.
以下は`rabbit.hrl`に定義されたexchange recordの定義である。

```erlang
%% fields described as 'transient' here are cleared when writing to
%% rabbit_durable_<thing>
-record(exchange, {
          name, type, durable, auto_delete, internal, arguments, %% immutable
          scratches,    %% durable, explicitly updated via update_scratch/3
          policy,       %% durable, implicitly updated when policy changes
          decorators}). %% transient, recalculated in store/1 (i.e. recovery)
```

Then we need to check that the record returned by Mnesia happens
is not an internal exchange, otherwise an error will be raised and the
publish will fail.

そしてMnesiaから返されたレコードが内部exchangeでないことを確認する必要がある。
もしそうでなければエラーを発行してpublishが失敗する。

The next thing to do is to validate the user id provided with the
basic publish, if any. If provided, this user id has to validated
against the user that created the channel where the message is being
published. More details
[here](https://www.rabbitmq.com/validated-user-id.html)

次に、basic publishに指定されたuser idを検証する。
そのメッセージがpublishされているチャンネルを作成したユーザーに対してuser idが検証される。

Then we need to validate if the message expiration header that the
user provided is correct. More info about the Per-Message-TTL

そしてユーザが指定したメッセージのexpirationヘッダが正しいことを確認する。
Per-Message-TTLについては以下を参照。

[here](https://www.rabbitmq.com/ttl.html#per-message-ttl)

Then it's time to check if the message was published as _Mandatory_ or
if the channel is in _Transaction_ or _Confirm Mode_. If this is the
case, then the `publish_seqno` field on the channel state will be
incremented to account for the new publish that's being handled. This
Message Sequence Number will be later used to reply back to the
publisher in case the message was Mandatory and/or the channel was in
[Confirm Mode](https://www.rabbitmq.com/confirms.html). See also the
document [Delivering Messages to Queues](./deliver_to_queues.md).

そしていよいよメッセージが _Mandatory_ としてpublishされたか、あるいは
channelが _Transaction_ または _Confirm Mode_ であるかを確認する。
その場合は、新しく処理されるメッセージのためにそのチャンネルの`publish_seqno`を
インクリメントする。
このメッセージシーケンス番号は、messageがMandatoryであるか、チャンネルが
Confirm Modeである場合にpublisherに対して返答するために後ほど使われる。

After all these steps have been completed, it's time to route the AMQP
message, but in order to do that we need to wrap the message first
into a `#basic_message` record, and then pass it to the exchange and
queues as a `#delivery{}` record:

これらのステップが完了したあと、AMQPメッセージをrouteするが、
そのためにはまずメッセージを`#basic_message`レコードにラップし、
exchangeに渡し、`#delivery{}`レコードとしてキューイングする。

```erlang
-record(basic_message,
        {exchange_name,     %% The exchange where the message was received
         routing_keys = [], %% Routing keys used during publish
         content,           %% The message content
         id,                %% A `rabbit_guid:gen()` generated id
         is_persistent}).   %% Whether the message was published as persistent

-record(delivery,
        {mandatory,  %% Whether the message was published as mandatory
         confirm,    %% Whether the message needs confirming
         sender,     %% The pid of the process that created the delivery
         message,    %% The #basic_message record
         msg_seq_no, %% Msg Sequence Number from the channel publish_seqno field
         flow}).     %% Should flow control be used for this delivery
```

## Message Routing ##

The `#delivery` we just created on the previous step is now passed to
the exchange via the function `rabbit_exchange:route/2`. If the
exchange name used during `basic.publish` is the empty string
`<<"">>`, then the `default` exchange is assumed, and the `route/2`
will just return the queue name associated with the routing key, per
AMQP spec. If that's not the case, then the delivery will be processed
first by the [exchange decorators](./exchange_decorators.md) that are
configured to the exchange that's handling the routing. The decorators
will send back a list of _destinations_. At this point, delivery will
finally reach the exchange, where the routing algorithm implemented by
the exchange will take place. This process will return a new list of
_destinations_ which will be merged and deduplicated with the list
returned before by the decorators. At this point, all the destinations
proposed by the
[Exchange To Exchange](https://www.rabbitmq.com/e2e.html) bindings are
also included in the list of destinations that will be returned to the
channel.

前のステップで作成した`#delivery`レコードは`rabbit_exchange:route/2`経由で
exchangeに渡される。exchange名が空の`<<"">>`の場合、`default`exchangeとみなされ、
`route/2`は単にrouting keyに関連するqueue名を返す。
そうで無い場合、その送信はまず、ルーティングを担当するexchange用に構成された
exchange decoratorsによって処理される。
decoratorsは _distinations_のリストを送り返す。
この時点で、送信はexchangeに到達し、exchangeによって実装されたroutingアルゴリズムが実行される。
この処理は前にdecoratorsによって返されたリストとマージされて重複除去される新しい _distinations_ を返す。
この時点で、exchange to exchangeバインディングで提案される全てのdistinations
もチャンネルに返されるリストの中に含まれることになる。

## Processing Routing Results ##

Now the channel has a list of queues to which it should deliver the
messages. Before doing that, we need to see if the channel is in
transaction mode, if that's the case, then the `#delivery` and the
list of queues are enqueued for later until the transaction is
committed. Keep in mind that transaction support in RabbitMQ are a
very simple form of
[message batching](https://www.rabbitmq.com/semantics.html). If the
channel is not in transaction mode, then the message will be delivered
to the queues returned by the routing function.

今チャンネルはメッセージが送信されるべきキューのリストを持っている。
それを行う前に、チャンネルがtransaction modeかどうかを確認する必要がある。
もしそうなら、`#delivery`とキューのリストはトランザクションがコミットされるまで
キューイングされる。なおRabbitMQのトランザクションはmessage batchingの
とてもシンプルな実装であることに留意すること。
もしチャンネルがtransaction modeでなければ、
ルーティング関数によって返されたキューにメッセージが送信される。

## Summary ##

We saw in this guide that messages arrive via the network into the
`rabbit_reader` process. This process forwards commands to
`rabbit_channel` processes who take care of processing the various
AMQP methods. In this case, we are seeing what happens when a message
is published into RabbitMQ. Once credit flow has been acked back to
the reader process, then it's time to take care of handling the
message. First it will go to the interceptors, who might modify or
augment the AMQP method received from the _reader_. Then the channel
must make sure the message complies to the size limits set at the
broker side. Once that's done, we need to see if the user has
permission to publish message to the selected exchange. If that's fine
and the `user_id` and `expiration` headers of the message are
validated, then it's time to route the message. The exchange who
handles the message will return back a list of queues to which the
message must be delivered to. At this point we are done with the
message and the channel is ready to keep processing commands.

ここではメッセージがネットワークから`rabbit_reader`プロセスへ到達するのを見た。
このプロセスは、AMQPの様々なメソッドを取り扱う`rabbit_channel`プロセスにコマンドを送る。
この場合、メッセージがRabbitMQにpublishされた時に何が起きるかを見た。
一度credit flowがreader processにackすると、メッセージを扱い始める。
最初に、readerから受けたAMQPメソッドを修正したり増幅したりするinterceptorsに行き、
そしてチャンネルはメッセージがブローカー側で設定されたサイズに収まっているのを確認する。
それが終わると、ユーザーが選択されたexchangeにメッセージを送信する権限があるかどうか確認する。
権限が問題なく、`user_id`と`expiration`ヘッダが問題なければ、
メッセージをルーティングする段である。そのメッセージを扱うexchangeが、そのメッセージが送信されるべき
キューのリストを返し、この時メッセージの処理が完了し、チャンネルは別のコマンドを処理できる状態になる。

Now we can continue with the next guide and see what happens when
messages are delivered to queues:
[Delivering Messages to Queues](./deliver_to_queues.md)

次に我々はメッセージがキューに届いてから何が起きるかを見ていく。