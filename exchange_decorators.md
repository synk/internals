# Exchange Decorators #

Exchange decorators are modules implemented as behaviours that can let
you extend existing exchanges. For example, you might want to perform
some actions only when the exchange is created, or deleted, but leave
alone the whole routing logic to the underlying exchange.

Exchange decoratorは既存のexchangeを拡張するビヘイビアとして実装されたモジュールである．
たとえばexchangeが作られたり削除されたときにだけいくつかのアクションを実行しつつ，
元となるexchangeのルーティングロジックはそのままにしておきたいかもしれない．

Decorators are usually associated with exchanges via policies.

Decoratorは通常はポリシーを通じたexchangeと関連している．

See the `active_for/1`
[callback](https://github.com/rabbitmq/rabbitmq-common/blob/master/src/rabbit_exchange_decorator.erl#L70)
to understand which functions on the exchange would be decorated.

`active_for/1`を参照してexchangeのどのfunctionが拡張できるかを理解しよう．

Take a look at the
[Sharding Plugin](https://github.com/rabbitmq/rabbitmq-sharding/blob/master/src/rabbit_sharding_exchange_decorator.erl)
and the
[Federation Plugin](https://github.com/rabbitmq/rabbitmq-federation/blob/master/src/rabbit_federation_exchange.erl)
to see how exchange decorators are implemented.

Sharding PluginとFederation Pluginは，exchange decoratorがどのように実装されるかを
見ることができる．