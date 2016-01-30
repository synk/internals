# Backing Queue #

RabbitMQ supports plugable backing queues by modules implementing the
`rabbit_backing_queue` behaviour.

RabbitMQは`rabbit_backing_queue`ビヘイビアを実装したモジュールによる
プラッガブルなbacking queueをサポートする。

The backing queue `init/3` callback expects an `async_callback()`
parameter which is a `fun` callback which takes the backing queue
state, and returns a new state. Keep reading to understand what all
this callback mumbo-jumbo means.

backing queueの`init/3`コールバックは、backing queueステートを引数にとり、
新しいステートを返すfunである`async_callback()`パラメータを要求する。

**TL;DR:** due to the two problems explained below, this callback
  takes care of executing certain functions in the context of a
  particular Erlang process.

以下に説明する2つの問題により、このコールバックは特定のErlangプロセスの
コンテキストの中のあるfunctionの実行に対処する。

## Process Dictionary Problem ##

Understanding how this callback works is vital since the persistence
layer of the backing queue does heavy use of the _process dictionary_
and the use of `self()` to track who opened which file handle. What
this means is that even tho the backing queue behaviour callbacks
seem to have the referential transparent property, they do not. Behind
the scenes, some of the backing queue behaviour callbacks will `put/get`
values to/from the process dictionary, but if one of said callbacks is
executed in a different process context, then those values won't be
found on the process dictionary, and everything else breaks havoc.

このコールバックがどのように動くかを理解するのは重要である、なぜなら、
backing queueの永続化レイヤは、ファイルハンドルをだれが開いたかをトラックするために
process dictionaryと`self()`を多用するからである。
これが意味するのは、backing queueビヘイビアコールバックは参照透過のようだが、
そうではない。
その裏は、backing queueビヘイビアコールバックのいくつかはprocess dictionary
から値を出し入れするが、コールバックのうちひとつが異なるプロセスコンテキストで
実行されると、それらの値はprocess dictionaryに無いため、
他のものは破壊する

## File Handle Cache Problem ##

The same applies for the `file_handle_cache` tracking who owns which
file handle by calling `self()` inside its functions implementations
instead of expecting a `Pid` as parameter for example. The call of
`self()` again violates referential transparency. The function
behaviour now depends on the process context on which it's
called. This means that closing file handles must be done from the
same caller that issued the file open.

同様に`file_handle_cache`は，どの関数がPidをパラメータとして取る代わりに
`self()`を呼んでファイルハンドルを所有しているかをトラッキングする．
`self()`の呼び出しは参照透過性を侵害する．
このfuncitonはそれを呼び出したプロセスコンテキストに依存した振る舞いを取る．
これはファイルハンドルを閉じるのはファイルを開いたのと同じ呼び出し元から
行われなければならないことを意味する．

## How Things Work ##

The function `rabbit_amqqueue_process:bq_init/3` takes care of
initializing the backing queue implementation, whether it is the
`rabbit_variable_queue`, the `rabbit_mirror_queue_master`, the
`rabbit_priority_queue` or your own backing queue behaviour
implementation.

`rabbit_amqqueue_process:bq_init/3`はbacking queueの実装
（`rabbit_variable_queue`, `rabbit_mirror_queue_master`,
 `rabbit_priority_queue`またはあなた自身のbacking queueビヘイビア実装）
の初期化を扱う．

The async callback passed into `BQ:init` is defined as:

`BQ:init`に渡される非同期コールバックの定義は；

```erlang
fun (Mod, Fun) ->
        rabbit_amqqueue:run_backing_queue(Self, Mod, Fun)
end
```

This `fun` will take a module argument, which is usually an atom
referring to the backing queue module being used, for example
`rabbit_variable_queue`, or `rabbit_mirror_queue_master`. The second
argument expected by this callback is a `fun` that will be passed
along to `rabbit_amqqueue:run_backing_queue/3`. Now lets see what
`rabbit_amqqueue:run_backing_queue` does.

この`fun`は，`rabbit_variable_queue`, `rabbit_mirror_queue_master`など，
通常は使用されるbacking queueを参照するatomをモジュール引数にとる．
二番目の引数は`rabbit_amqqueue:run_backing_queue/3`に伝達される`fun`である．
では`rabbit_amqqueue:run_backing_queue`がなにをするか見てみよう．

### rabbit_amqqueue:run_backing_queue ###

The function body is like this:

このfunction本体はこのようである．

```erlang
run_backing_queue(QPid, Mod, Fun) ->
    gen_server2:cast(QPid, {run_backing_queue, Mod, Fun}).
```

It sends a `{run_backing_queue, Mod, Fun}` message to whatever process
was provided as `QPid`. **This is important**, since that process'
context is the one which will get its process dictionary modified
indirectly, and at the same time will own file handles when they are
opened by the _msg\_store_ for example.

これは`{run_backing_queue, Mod, Fun}`のメッセージをQPidとして与えられた
プロセスに送る．これは（ここが重要）このプロセスのコンテキストは間接的にそのプロセス
dictionaryを更新されるのと，また同時にたとえばmsg\storeから開かれたときに
ファイルハンドルを持つため．

Back to `rabbit_amqqueue_process` we will see that this module has a
callback for the message mentioned above:

`rabbit_amqqueue_process`に戻って，このモジュールは先に述べたメッセージのための
コールバックを持つ．

```erlang
handle_cast({run_backing_queue, Mod, Fun},
            State = #q{backing_queue = BQ, backing_queue_state = BQS}) ->
    noreply(State#q{backing_queue_state = BQ:invoke(Mod, Fun, BQS)});
```

This function takes care of extracting the current `backing_queue`
module and `backing_queue_state` from its own process state, and then
calling `BQ:invoke(Mod, Fun, BQS)`.

このfunctionは現在の`backing_queue`モジュールと`backing_queue_state`を
自分のプロセスのステートから取り出し，`BQ:invoke(Mod, Fun, BQS)`を呼ぶ．

This is what `BQ:invoke/3` does:

これが`BQ:invoke/3`が何をしているかである：

```erlang
invoke(?MODULE, Fun, State) -> Fun(?MODULE, State);
invoke(      _,   _, State) -> State.
```

Invoke's implementation is pretty simple, if the `Mod` argument
provided to it matches the current module, in this example
`rabbit_variable_queue`, then the `Fun` will be executed with
`rabbit_variable_queue` as first parameter and the backing queue
`State` as the second argument. To reiterate, what's important to
understand is that `Fun` will be executed in the context of whatever
`QPid` was referring to above. In the case we are analyzing so far,
this is the `rabbit_amqqueue_process` pid.

Invokeの実装はとても単純で，もし`Mod`引数が現在のモジュールにマッチすれば，
この例では`rabbit_variable_queue`だが，`Fun`は`rabbit_variable_queue`
を最初の引数として，backing queueの`State`を第二の引数として，実行される．
繰り返しになるが，理解しておくべきことは，`Fun`がなんであれ`QPid`が参照する
コンテキストの中で実行されるということである．
今の場合は`rabbit_amqqueue_process`のpidがそれである．

## What Fun ##

Now let's try to find out what `Fun` actually is. To get to this we
need to see how `rabbit_variable_queue` is initialized`.

では`Fun`が実際になんであるかを確認しよう．このためには`rabbit_variable_queue`
がどのように初期化されるかを見る必要がある．

`rabbit_variable_queue:init/6` will call into
`msg_store_client_init/3` passing our initial callback as the third
parameter (`msg_store_client_init/3` then expands into
`msg_store_client_init/4`). Let's refresh what that callback was:

`rabbit_variable_queue:init/6`は`msg_store_client_init/3`
（これはさらに`msg_store_client_init/4`につながる）に
最初のコールバックを第三の引数として渡して呼び出す．
そのコールバックがなんであったかを更新しよう．

```erlang
fun (Mod, Fun) ->
        rabbit_amqqueue:run_backing_queue(Self, Mod, Fun)
end
```

That callback will be now wrapped into yet another `fun` like this:

そのコールバックはこのようにまた別の`fun`にラップされる：

```erlang
fun () -> Callback(?MODULE, CloseFDsFun) end
```

To see that in context:

文脈の中でみると：

```erlang
msg_store_client_init(MsgStore, Ref, MsgOnDiskFun, Callback) ->
    CloseFDsFun = msg_store_close_fds_fun(MsgStore =:= ?PERSISTENT_MSG_STORE),
    rabbit_msg_store:client_init(MsgStore, Ref, MsgOnDiskFun,
                                 fun () -> Callback(?MODULE, CloseFDsFun) end).
```

So now we have a clue of what the `Fun` passed into our callback might
be. It is whatever `msg_store_close_fds_fun` returned as
`CloseFDsFun`. Let's check:
そして今我々のコールバックに渡される`Fun`がなんであるかのヒントを得た．
それは`msg_store_close_fds_fun`が`CloseFDsFun`として返すものである．

```erlang
msg_store_close_fds_fun(IsPersistent) ->
    fun (?MODULE, State = #vqstate { msg_store_clients = MSCState }) ->
            {ok, MSCState1} = msg_store_close_fds(MSCState, IsPersistent),
            State #vqstate { msg_store_clients = MSCState1 }
    end.
```

We get a `fun` that will only be executed if the `Mod` argument
matches, in this case `rabbit_variable_queue`. That `fun` takes as
second argument our `rabbit_variable_queue` state.

`Mod`引数がマッチしたとき（この場合は`rabbit_variable_queue`)
のみ実行される`fun`を得る．

On `msg_store_client_init/4` above we said that our initial callback
gets wrapped like this:

上の`msg_store_client_init/4`で我々の最初のコールバックはこのようにラップされる．

```erlang
fun () -> Callback(?MODULE, CloseFDsFun) end
```

This means inside the msg_store, at various places, that `fun` closure
gets called without arguments which in turn calls our callback with
the `CloseFDsFun`. We end up with something like what's below after
some expansions:

これは，msg_storeの様々なところで，その`fun`クロージャは我々のコールバックと
`CloseFDsFun`を呼び出す引数なしで呼び出される．いくらかの拡張のあと，
以下のようになる：

```erlang
fun (rabbit_variable_queue, Fun) ->
        rabbit_amqqueue:run_backing_queue(QPid, rabbit_variable_queue,
            fun (?MODULE, State = #vqstate { msg_store_clients = MSCState }) ->
                {ok, MSCState1} = msg_store_close_fds(MSCState, IsPersistent),
                State #vqstate { msg_store_clients = MSCState1 }
            end)
end
```

So our `rabbit_amqqueue_process` will ask the backing queue module
to invoke that expanded fun in the context of the
`rabbit_amqqueue_process` Pid:

したがって我々の`rabbit_amqqueue_process`はbacking queue moduleに
`rabbit_amqqueue_process` Pidのコンテキストのなかでその拡張したfunを
呼び出すように依頼する：

```erlang
handle_cast({run_backing_queue, Mod, Fun},
            State = #q{backing_queue = BQ, backing_queue_state = BQS}) ->
    noreply(State#q{backing_queue_state = BQ:invoke(Mod, Fun, BQS)});
```

This very same technique is used on `rabbit_variable_queue:init/3` to
setup the functions that will write messages to disk (see
`rabbit_variable_queue:msgs_written_to_disk/3`) and the ones that will
write the message indexes to disk (see
`rabbit_variable_queue:msg_indices_written_to_disk/2`).

これはディスクにメッセージを書き込むためのfunctionや
(`rabbit_variable_queue:msgs_written_to_disk/3`をみよ)
message indexをディスクに書き込むためのfunctions
(`rabbit_variable_queue:msg_indices_written_to_disk/2`をみよ)
をセットアップするために
`rabbit_variable_queue:init/3`で使われるのとほとんど同じテクニックである．

## It's About Context ##

From all these layers of indirection, what's important to understand
is that the `Pid` passed into `rabbit_amqqueue:run_backing_queue/3`
determines the context on which all the functions implementing message
persistence will be run. Unless your `rabbit_backing_queue` behaviour
implementation is just a proxy like that of `rabbit_priority_queue`,
you must take that `Pid` context into account, since it will hold file
handles references and its process dictionary will be the one where
the `file_handle_cache` will store its information.

これらの間接的な層から，理解しなければならないのは，`rabbit_amqqueue:run_backing_queue/3`
に渡される`Pid`が，メッセージの永続化を実装するすべてのfunctionsが実行される
コンテキストを決定するということである．

If you want a second example of what we outlined above, take a look at
`rabbit_mirror_queue_slave:bq_init/3` where the Pid provided to
`run_backing_queue/3` in this case is the _slave_ Pid. The slave
process implements it's own `handle_cast({run_backing_queue, Mod,
Fun}, State)` function clause, on which funs from
`rabbit_variable_queue` like `msg_store_close_fds_fun`,
`msgs_written_to_disk`, `msg_indices_written_to_disk` and
`msgs_and_indices_written_to_disk` will be run.

以上に解説した件についての別の事例として，`rabbit_mirror_queue_slave:bq_init/3`
では，`run_backing_queue/3`にslaveのPidが渡される．
スレーブのプロセスは自分の`handle_cast({run_backing_queue, Mod, Fun}, State)`
関数クロージャを実装し，この中で`msg_store_close_fds_fun`,
`msgs_written_to_disk`, `msg_indices_written_to_disk`
そして`msgs_and_indices_written_to_disk`のような，
`rabbit_variable_queue`からの関数が実行される