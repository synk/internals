# Queue Decorators #

Queue decorators are modules implemented as behaviours that can let
you extend queues. A decorator will have a set of callbacks that will
be called from the queue process in response to some events that might
happen at the queue, like when messages are delivered to consumers,
which might cause the list of active consumers to be updated.

Queueデコレータはキューを拡張するためのビヘイビアとして実装されたモジュールである．
デコレータは，キューに発生する様々なイベントに対してレスポンスするキュープロセス
から呼び出される各種コールバックを持つ．たとえばメッセージがコンシューマにデリバリ
されたとき，アクティブなコンシューマのリストをアップデートするかもしれない．

They were added to the broker as a way to handle
[consumer priorities](https://www.rabbitmq.com/consumer-priority.html)
or by the federation plugin, to know when to move messages across
[federated queues](https://www.rabbitmq.com/federated-queues.html).

これらはconsumer prioeritiesを制御する方法としてや，federation pluginが
メッセージを転送するタイミングを知るために，ブローカーに追加された．

Decorators need to implement the `rabbit_queue_decorator`
[behaviour](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_queue_decorator.erl)
and are usually associated with queues via policies.

Decoratorsは`rabbit_queue_decorator`ビヘイビアを実装する必要がある
また，通常はポリシーによるキューに関連している．

A Queue decorator can receive notifications of the following events:

Queueデコレータは以下のようなイベントの通知を受け取ることができる．

- Queue Startup
- Queue Shutdown
- Consumer State Changed (active consumers)
- Queue Policy Changed
