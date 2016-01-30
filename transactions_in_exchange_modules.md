# What are those transactions inside the exchange callback modules? #

Many callbacks inside the `rabbit_exchange_type` behaviour expect a
`tx()` parameter which is defined as follows:

rabbit_exchange_typeビヘイビアの中の多数のコールバックは下記のように定義された
tx()パラメータを要求する．

```erlang
-type(tx() :: 'transaction' | 'none').
```

Then for example create is defined like:

そして例としてcreateは以下のように定義される

```erlang
 %% called after declaration and recovery
-callback create(tx(), rabbit_types:exchange()) -> 'ok'.
```

The question is, what's the purpose of that transaction parameter?

ここでの疑問は，このトランザクションパラメータの目的はなんだろうか

This is related to how RabbitMQ runs Mnesia transactions for its
internal bookkeeping:

これはRabbitMQが内部予約のためにMnesiaトランザクションをどうやって
実行しているかに関連している．

[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_misc.erl#L546](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_misc.erl#L546)

As you can see in that code there's this PrePostCommitFun which is
called in Mnesia transaction context, and after the transaction has
run.

このコードからわかるように，Mnesiaトランザクションコンテキストから呼び出される
PrePostCommitFunがある．そしてトランザクションの後にも実行される．

So here for example:
[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange.erl#L176](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange.erl#L176)
the create callback from the exchange is called inside a Mnesia
transaction, and outside of afterwards.

これが例である：
exchangeからコールバックを作成するのはMnesiaトランザクションの中から,
後ほど外から呼ばれる

You can see this in action/understand the usefulness of it when
considering an exchange like the topic exchange which keeps track of
its own data structures:

実際の様子を見ることができ，それ自身のデータ構造を追跡するtopic exchange
のようなexchangeを考えたとき，その利便性を理解することができよう．

[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L54](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L54)
[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L64](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L64)
[https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L69](https://github.com/rabbitmq/rabbitmq-server/blob/master/src/rabbit_exchange_type_topic.erl#L69)

Deleting the exchange, adding or removing bindings, are all done
inside a Mnesia transaction for consistency reasons.

exchangeを削除する，追加する，バインディングを削除する，これらは並行性の理由から
すべてMnesiaトランザクションのなかで行われる．
