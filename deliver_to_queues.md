# Deliver To Queues #

In this document we will be going over quite a few RabbitMQ modules,
since a message crosses all of these once it enters the broker:

この文章ではたくさんのRabbitMQモジュールを駆け抜ける。
なぜなら、メッセージが一度ブローカーに入るとそれら全てを横断するためである。

```
rabbit_reader -> rabbit_channel -> rabbit_amqqueue -> delegate -> rabbit_amqqueue_process -> rabbit_backing_queue
```

Let's see this process in more detail.

このプロセスをより詳細に見ていこう。

The process of delivering messages to queues start during
`basic.publish`, right after the channel receives the result from
calling `rabbit_exchange:route/2`.

キューへのメッセージ送信のプロセスは`basic.publish`でチャンネルが
`rabbit_exchange:route/2`の結果を受け取ってから始まる。

First we need to lookup the list of `#amqqueue` records based on the
destinations obtained from `route/2`. These records will be passed to
the function `rabbit_amqqueue:deliver/2` where they will be used to
obtain the _pids_ of the queue process where the message is going to
be delivered. Once the master and slave pids have been obtained, then
the message can start its way to be delivered to a queue process,
which consists of two parts: accounting for credit flow, and casting
the message into the queue process.

最初に`route/2`による送信先に基づいて`#amqpqueue`レコードのリストを照会する必要がある。
これらのレコードは、メッセージが送信されるqueueプロセスのpidを取得するために
`rabbit_amqqueue:deliver/2`に渡される。
一度masterとslaveのpidsが取得されると、メッセージはキュープロセスに送信可能になる。
これは二つの部分からなる：credit flowをaccountingする、queueプロセスに
メッセージを投げる。

If the message delivery arrived with `flow = true`, then `credit_flow`
must be accounted for this message. One credit for each master Pid
where the message should arrive, plus one credit for each slave pid
that receives the message.

もしメッセージデリバリが`flow = true`の場合、`credit_flow`はこのメッセージを扱わなければならない。
メッセージが到達すべきそれぞれのmaster PIDにつき一つのcreditと、メッセージを
受診する各slave pidにつき一つのcredit

Then the message delivery will be sent to master pids and slave pids,
via the `delegate` framework. The Erlang message will have this shape:

メッセージデリバリは、`delegate`フレームワークによって、master pidとslave pidに送信される。
Erlang messageは次の形をとる：

```erlang
{deliver,            %% message tag
 Delivery,           %% The Delivery record
 SlaveWhenPublished} %% The Pid that received the message, was it a
                     %% slave when the deliver was published? This is
                     %% used in case of slave promotion
```

You can learn more about the delegate framework
delegate frameworkについては以下でさらに学ぶことができる。
[here](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/delegate.erl#L19).

## AMQQueue Process Message Handling ##

At this point the message delivery will finally arrive at the queue
process, implemented as a `gen_server2` callback inside the
`rabbit_amqqueue_process` module. The message from the delegate
framework will be received by the `handle_cast/2` callback. This
callback will ack the `credit_flow` issued in above, and it will
monitor the message sender. The message sender is usually the
`rabbit_channel` that received the process. This pid is tracked using
the
[pmon module](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/pmon.erl). The
state is kept as part of the `senders` field in the gen_server state
record. Once the message sender is accounted for the delivery is
passed to the function `deliver_or_enqueue/3`. There is where the
message will either be sent to a consumer or enqueued into the backing
queue.

ここでメッセージデリバリは、`rabbit_amqqueue_process`モジュールの中で
`gen_server2`コールバックとして実装されたキュープロセスに到着する。
delegate frameworkからきたメッセージは`handle_cast/2`コールバックに
受診される。このcallbackは前に発行された`credit_flow`にackする、
そしてメッセージ送信者を監視する。メッセージ送信者は普通、プロセスを受けた`rabbit_channel`に成る。
このpidはpmonモジュールを使って追跡される。
そのステートはgen_serverステートレコードの`senders`フィールドの一部として保持される。
一度メッセージ送信元がそのデリバリにaccountされると、`deliver_or_enqueue/3`に渡される。
ここでメッセージはコンシューマに送られるか、backing queueに入れられる。

### Mandatory Message Handling ###

The first thing `deliver_or_enqueue/3` does is to account for the
mandatory flag of the delivery. If the message was published as
mandatory, then at this point the queue process will consider the
message as routed to the queue. To that effect, the queue process will
cast the message `{mandatory_received, MsgSeqNo}` to the channel pid
that received the delivery. The channel process will the proceed to
forget the message, since from the point of view mandatory message
handling, there isn't anything left to do for that particular
delivery.

`deliver_or_enqueue/3`で最初に行われるのは、そのデリバリのmandatoryフラグ
をaccountする。メッセージがmandatoryとして発行された場合、キュープロセスは
キューにルートされたものとして考慮する。
この影響で、キュープロセスはメッセージ`{mandatory_received, MsgSeqNo}`
を、デリバリを受けたチャンネルpidに投げる。
そのチャンネルプロセスはそのメッセージを忘れる。mandatoryメッセージの扱いという
点から、そのデリバリに対してすることが残っていないからである。

Take a look at the
[mandatory message handling guide](./mandatory_message_handling.md) for
more info.

### Message Confirm Handling ###

When handling confirms we need to take into account two things: is the
queue durable, and was the message published as persistent. If that's
the case, then the queue process will keep track of the `MsgId` in
order to confirm the message back later to the channel that received
it from a producer. To achieve that, the queue process keeps track of
a dictionary in the process state, using `msg_id_to_channel` record
field to hold it. As the name of the field implies, this dictionary
maps _msg ids_ to _channels_. When a message is finally persisted to
disk by the backing queue, then the BQ will notify the queue process,
which will send the confirm back to the channel using the
`msg_id_to_channel` dictionary just mentioned.

Confirmを扱う時、2つのことを考慮する必要がある：
queueがdurableであるか、そしてmessageはpersistentとしてpublishされたか。
もしそうなら、キュープロセスは、producerからメッセージを受信した
チャンネルにconfirmを送り返すために`MsgID`を追跡する。
これを達成するために、`msg_id_to_channel`レコードフィールドを使って、
プロセスステートの中にディクショナリーを維持する。
このフィールドの名前が示唆するように、このディクショナリーはmsg idsとchannels
をマッピングする。メッセージがbacking queueによって最終的にディスクに永続化された時、
先ほどのディクショナリーを使って、BQはキュープロセスに通知する。

If the queue was non durable, or the message was published as
transient, then the queue process will proceed to issue a confirm back
to the channel that sent the message in.

キューがdurableでない、またはメッセージがtransientとして送信された場合、
confirmを送信元のチャンネルに送り返す。

The function `rabbit_misc:confirm_to_sender/2` is the one taking care
of sending confirms back to channels.

関数`rabbit_misc:confirm_to_sender/2`はチャンネルにconfirmを送り返すものの
一つである。

Take a look at the
[publisher confirm handling guide](./publisher_confirms.md) for more info.

### Check for Message Duplicates ###

The next step is to check if the message has been seen by the queue
before. If the backing queue responds that the message is a duplicate,
then processing stops right here, since there's anything left to do
for this delivery, so `deliver_or_enqueue/3` simply returns.

次のステップは、メッセージがキューに置いて以前見られたかどうかを
確認することである。もしbacking queueがそのメッセージをduplicateとして
判断した場合、ここで処理を止め、`deliver_or_enqueue/3`は単にreturnする。

### Attempt to Deliver the Message to a Consumer ###

To try to send the message delivery to a consumer, the function
`attempt_delivery/4` is called. This function will in turn call
`rabbit_queue_consumers:delivery/3` which takes a `FetchFun`, the
`QueueName`, and the `Consumers State` for this particular queue. The
Fetch Fun will return the message that will be delivered to the
consumer (if a consumer is available). This function deals with
message acknowledgment from the point of view of the queue. If the
consumer is in `ackmode = true`, then the message will be
`publish_delivered` into the backing queue, otherwise the message will
be discarded.

コンシューマにメッセージデリバリを送信するために、`attempt_delivery/4`
が呼び出される。この関数は`rabbit_queue_consumers:delivery/3`を呼び出す。
これは各キューに対して`FetchFun`, `QueueName`, `Consumers State`を引数に取る。
Fetch Funはコンシューマに（もしコンシューマが有効なら）届けられるメッセージを返す。
この関数はメッセージのacknowledgementをキューの観点から扱う。
もしコンシューマが`ackmode = true`なら、そのメッセージはbacking queueに`publish_delivered`
され、そうでなければメッセージは捨てられる。

Discarding a message involves confirming the message, in case that's
required for this particular delivery, and telling the backing queue
to discard it as well.

メッセージの破棄はメッセージのconfirmingに関係する。
あるデリバリにconfirmingが求められる場合、backing queueにそれを
破棄することを伝える。

Once the queue attempted to deliver the message straight to a
consumer, it will call the function `maybe_notify_decorators/2` which
takes care of telling the queue decorators that the consumer state
might have changed. See the [queue decorators](./queue_decorators.md)
guide for more information on how decorators work.

一度キューがメッセージをコンシューマに届けることを試みると、
`maybe_notify_decorators/2`を呼び出し、キューデコレータにコンシューマの
ステートが変わったかもしれないことを通知する。
queue decoratorsのガイドをみよ。

The `attempt_delivery/4` will return back to the
`deliver_or_enqueue/3` function telling it if the message was
`delivered` or if it is still `undelivered`. If the message was
delivered to a consumer, then there's nothing else to do, and
`deliver_or_enqueue/3` will simply return. Otherwise there's still
more to do.

この`attempt_delivery/4`は`deliver_or_enqueue/3`に対して
メッセージが`delivered`あるいはまだ`undelivered`かを返す。
もしメッセージがコンシューマに届けられたら、何もせずreturnする。
そうでなければまだやることがある。

### Handling Undelivered Messages ###

When handling undelivering messages, there's a special case that can
be considered an optimization. If the queue has a
[TTL](https://www.rabbitmq.com/ttl.html) of 0, and no
[DLX](https://www.rabbitmq.com/dlx.html) has been set up, then there
is no point in queueing this message, so it can be discarded in the
same way as explained above.

届けられなかったメッセージのハンドリングは、最適化が可能な特別なケースである。
もしキューのTTLが0でDLXがセットされていない場合、キューイングする場所はないので、
上記と同じように破棄することができる。

If a message cannot be discarded, then it has to be enqueued, so the
queue process will `publish` the message into the backing queue. After
the message has been published, we need to enforce the various
policies that might apply to this queue, like `max-length` for
example. This means we need to see if the queue head has to be
dropped. Once that's enforced, then we also have to check if we need
to drop expired messages. Both these functions work in conjunction
with the DLX feature mentioned above. At this point
`deliver_or_enqueue/3` returns.

もしメッセージが破棄できない場合は、キューイングする必要がある。
キュープロセスはメッセージをbacking queueに`publish`する。
メッセージがpublishされると、様々なポリシーをこのキューに適用する必要がある。
例えば`max-length`など。これはキューの先頭をdropする必要があるか
確認する必要があることを意味する。
一度それが適用されると、expireしたメッセージを破棄する必要があるかも
確認する必要がある。これらの機能は前述のDLX機能と連動して作用する。
ここで`deliver_or_enqueue/3`はreturnする。

## Bookkeeping ##

Even if we are done with the delivery after this was handled by the
respective queue processes where it was sent, we still need to perform
some bookkeeping on the channel side. The `rabbit_amqqueue:deliver/2`
function will return a list of `QPids` that received the
messages. This list of pids will be used now for bookkeeping.

送信されたそれぞれのキュープロセスによって扱われるデリバリが終わったとしても、
まだチャンネルサイドで予約をする必要がある。
`rabbit_amqqueue:deliver/2`はQPidsのリストを返す。
このリストは予約のために使われる。

### Queue Monitoring ###

The first thing to do is to monitor the queue pids to which the
message was delivered. This is done among other things, to account for
credit flow in case the queue goes down. We don't want to block the
channel forever if a queue that's blocking it is actually down.

最初にやることは、メッセージが届けられたキューのpidsをモニタすることである。
これは他のものによって行われる。credit flowでキューがダウンした場合のために。
もしブロックしたキューが実際にダウンしていた場合に、永久にチャンネルをブロックしたくない。

Take a look at the `handle_info` channel callback for the case when a
`DOWN` message is
[received](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_channel.erl#L578).

### Process Mandatory Messages ###

Here if the message wasn't delivered to any queue, then it's time to
issue `basic.return`s back to the publisher that sent them. If the
message was delivered to queues, then those `QPids` will be kept into
a dictionary for later processing.

ここでもしメッセージがどのキューにもデリバリされなかった場合、送信元のpublisherに
`basic.return`を発行する。もしメッセージがデリバリされたら、QPidsが
後続の処理のためにディクショナリに保持される。

As explained above, once the queue process receives the message
delivery, then it will take care of updating the `mandatory`
dictionary on the channel's state.

ここまでに説明されたように、一度キュープロセスがメッセージを受診すると、
チャンネルステートのmandatoryディクショナリーを更新する。

### Process Confirms ###

Similar as with mandatory messages, if the message wasn't routed to
any queue, then it's time to record the message as confirmed. If the
message was delivered to some queues, then it will be tracked as
unconfirmed until the queue updates the message status.

mandatoryメッセージと同様、もしメッセージがどこにもルートされなかった時、
メッセージをconfirmedと記録する。もしメッセージがどこかのキューにデリバリされたら、
キューがメッセージのステートを更新するまで、unconfirmedとして追跡される。

### Stats Update ###

The final step for the channel is to account for stats, so it will
update the exchange stats, indicating that a message has been routed,
and then it will also update the queue stats, to indicate that a
message was delivered to this or that queue.

チャンネルの最後のステップは、statsをaccountすることであり、exchangeのstats
を更新してメッセージがルートされたことを示し、queue statsも更新して
メッセージがキューにデリバリされたことを示す。

## Summary ##

Delivering a message to a RabbitMQ queue is quite an involved process,
and we didn't even touch on queue mirroring! The main things to
account for when handling a delivery are mandatory messages and
message confirms. Both have to be handled accordingly, and the whole
process is coordinated between the channel process and the queue
process that receives the message. Other than that, the queue needs to
see if the message can be delivered to a consumer or if it has to be
enqueued for later. Once this is handled, the queue needs to enforce
the various policies that can be applied to it, like TTLs, or
max-lengths.

RabbitMQのキューにメッセージをデリバリすることは複雑なプロセスであり、
キューのミラーリングについてはまだ触れてもいない。
デリバリのハンドリングにおける主な事項はmandatoryメッセージとメッセージconfirmである。
両者は適宜扱われる必要がある、そして全体のプロセスはメッセージを受信した
チャンネルプロセスとキュープロセスの間で調整される。
他にも、キューは、メッセージがコンシューマに送信できるかどうか、後でキューイング
されるべきかどうかを確認する必要がある。
一度これが行われると、キューは様々なポリシーをキューに適用する必要がある。
TTLやmax-lengthなど。

To understand what happens once a message arrives to a queue, take
look at the [variable queue](./variable_queue.md). guide.

メッセージがキューに到着した時に何が起きるか理解するにはvariable queueをみよ。