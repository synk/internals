# Variable Queue #

## Publishing messages ##

When a message is published to the queue, the first thing we have to
do is to determine if the message is persistent. We track this
information in the `msg_status` record:

メッセージがキューにpublishされるとき、最初にやるべきことはメッセージがpersistent
かどうかを決定することである。この情報をmsg_statusレコードでトラックする。


```erlang
-record(msg_status,
        { seq_id,
          msg_id,
          msg,
          is_persistent,
          is_delivered,
          msg_in_store,
          index_on_disk,
          persist_to,
          msg_props
        }).
```

Message statuses are kept in a record where the `is_persistent` field
is set to `true` if the queue is durable and the message was published
as persistent:

もしキューがdurableでそのメッセージがpersistentのとき、メッセージステータスは
is_persistentフィールドをtrueにセットされる：

```erlang
is_persistent = IsDurable andalso IsPersistent
```

If it was determined that the message needs persistence, then it will
be immediately written to disk, either to the message store or the 
queue index, depending on the message size (see
`queue_index_embed_msgs_below`).

メッセージがpersistenceである必要があると決定された場合、それは直ちに
ディスクか、メッセージストア、queue indexに、メッセージサイズに応じて書き込まれる。


Internally the `variable_queue` keeps messages on four `queue` data
structures. They are a variation of erlang's _queue_ module, but which
some extensions that allow getting the queue length in constant
time. These four queues are identified on the variable queue state as
`q1`, `q2`, `q3` and `q4`. The need for these four queues becomes
apparent once disk paging is taken into account.

内部的にはvariable_queueはメッセージを4つのqueueデータ構造に保持する。
それらはerlangのqueueモジュールのバリエーションであるが、定時間でキューの長さを
得ることができる拡張である。それらの4つのキューはvariable queueステートの
q1, q2, q3, q4として識別される。4キューの必要性はディスクページングが考慮される
状況において発揮される。

`q4` keeps track of the oldest messages, that is, those at the front
of the queue, those that will be delivered earlier to consumers.

q4は最も古いメッセージを記録する。つまり、キューの先頭で最も先にコンシューマに
送られるメッセージである。

`q3` only has messages when there has been some disk paging due to
memory pressure, or if we have a queue that has recovered contents
from disk, due to a broker restart for instance. This means that
messages that once were in `q4` only, now have had their content
pushed to disk, and their references are now kept in `q3`. So when a
message arrives into the variable queue, we need to determine if the
message needs to be inserted at the back of `q4`, or somewhere else.

q3はメモリ圧迫のためにディスクページングが起きたときか、または、ブローカーの再起動
のためにディスクから復旧したキューがある場合にのみメッセージを保持する。
q4にのみあったメッセージがディスクにプッシュされてそれらの参照がq3に保持される。
従ってメッセージがvariable queueに到達したとき、メッセージをq4の最後か、
あるいは他のどこかに入れるかを決定する必要がある。

If `q3` is empty, this means we haven't paged queue contents to disk,
so the messages at the front of the queue are still in `q4`, and the
last message arriving to the queue is still in `q4` as well. So new
messages can be inserted at the back of `q4`. Now, if `q3` has
messages in it, this means at some point we have paged to disk, so
some messages that were at the rear of `q4` are in `q3` now. This
means a new message _can't_ be inserted into `q4`, otherwise we will
lose message ordering; therefore, if `q3` has messages, new messages
go into `q1`.

q3が空のとき、これはキューの内容をディスクにページングしておらず、キューの先頭は
まだq4にあり、最後に届いたメッセージもq4にあることを意味する。
従って新しいメッセージはq4の最後に追加することができる。
q3がメッセージを持つとき、これはどこかの時点でディスクにページングし、
q4より後方のメッセージがq3にあることを意味する。
この場合はメッセージをq4に入れるとメッセージの順序を失ってしまう。
そのため、q3がメッセージを持つときは、新しいメッセージをq1に入れる。

```erlang
case ?QUEUE:is_empty(Q3) of
    false -> State1 #vqstate { q1 = ?QUEUE:in(m(MsgStatus1), Q1) };
    true  -> State1 #vqstate { q4 = ?QUEUE:in(m(MsgStatus1), Q4) }
end,
```

## Fetching Messages ##

Messages are fetched by calling `queue_out/1`, which retrieves
messages from `q4` or, if `q4` is empty then they are retrieved from
`q3`.

メッセージのフェッチはqueue_out/1により行う。これはメッセージをq4から、または
q4が空の場合はq3からメッセージを取得する。

For `q3` to have messages, it means that at some point messages were
paged to disk due to memory pressure which led to
`push_alphas_to_betas/2` being called. Another way for `q3` to get
messages is when we load messages from disk into memory, for example
when `maybe_deltas_to_betas/1` is called; this can happen when we are
recovering queue contents during queue initialization, or also when we
try to load more messages from disk so they can be delivered to
clients.

q3がメッセージを持つとき、それはどこかの時点でメモリ圧迫のために
push_alphas_to_betas/2呼び出しによってメッセージがディスクにページされたことを
意味する。q3からメッセージを取得する他の状況としては、ディスクからメモリに
メッセージをロードしたときで、たとえばmaybe_deltas_to_betas/1が呼び出されたとき
：これはキュー初期化の間にキューの内容をリカバリしているときや、さらにメッセージを
ディスクから読み込もうとしてクライアントに送信できるときにも起き得る。

When there are no more messages in `q4`, `fetch_from_q3/1` is called
trying to obtain messages from `q3`. If `q3` is empty, then the queue
must be empty. Remember that if `q3` wasn't empty, then new messages
arriving into the queue were put into `q1`.

q4にメッセージが無いときは、fetch_from_q3/1が呼ばれてq3からメッセージを取得を
試みる。q3が空の時、キューは空のはずである。q3が空でないときは新しいメッセージは
q1に入るのだった。

If `q3` wasn't empty, but the message fetched was the last one there,
we must see if we need to load more messages from disk, or if we need
to migrate `q1` messages into `q4`.

q3が空で無いとき、しかしフェッチされたメッセージは最後の一つだった場合、
ディスクからさらにメッセージをロードする必要があるかどうか、あるいは
q1のメッセージをq4に移動する必要があるかを確認しなければならない。

So let's say we fetched the last message from `q3` and we know there
are no more messages on disk (delta count = 0). If there were new
publishes while `q3` had messages, those messages are in `q4`, so we
need to move messages from `q1` into `q4`. Why? Remember that during
publishing messages are queued into `q4` when `q3` is empty, otherwise
they go into `q1`. Imagine that we were publishing messages into `q4`,
then at some point we had to `push_alphas_to_betas`, which means some
`q4` messages were moved into `q3`. Now that `q3` has some messages,
_new_ messages are put into `q1`, but from the point of view for an
external user of the backing queue, `q3` messages come first than
those in `q1`, i.e.: they are at the front of the queue. So when there
are no more messages in `q3`, we can start consuming those in `q1`,
but since `queue_out/1` only fetches messages from `q4`, we move `q1`
contents there.

q3から最後のメッセージをフェッチし、そしてディスクにもそれ以上のメッセージが
存在しないとわかっているとしよう（delta count = 0）。
q3にメッセージがある状態で新しいpublishがあると、それらはq4にあり、したがって
q1からq4にメッセージを移す必要がある。なぜか？q3が空の時はq4にキューイングされるのを
思いだそう。そうでなければq1にキューイングされる。想像しよう、今q4にキューイングした。
そしていつかpush_alphas_to_betasする必要がある。これはいくつかq4からq3に
メッセージが移動する。いまq3はいくつかのメッセージを持っていて、新しいメッセージを
q1に入る。しかし外部のbacking queueのユーザーの視点からすると、q3のメッセージは
q1にあるメッセージより先である。つまり：それらのメッセージはキューの先頭にいる。
したがってq3にメッセージがないとき、q1からコンシュームを始めることができるが、
queue_out/1はq4からしかフェッチをしないため、q1からq4へメッセージを移すのである。

Now let's say there were remaining messages on disk, instead of moving
messages from `q1` into `q4`, we have to load messages from disk into
`q3`. This is accomplished by calling `maybe_deltas_to_betas/1`.

（q3から最後のメッセージをフェッチしたときに）ディスクにメッセージが残っているとしよう、
q1からq4にメッセージを移動する代わりに、ディスクからq3にメッセージをロードする必要がある。
これはmaybe_deltas_to_betas/1によって実行される。

`maybe_deltas_to_betas/1` reads the index and then depending on what
it finds there, it loads messages at the rear of `q3`. If there are no
more messages on disk, then those messages that are in `q2` are moved
to the rear of `q3`. When we look into message paging we will see why
only when there are no more paged messages, we move `q2` into `q3`,
and why `q2` messages go to the back of `q3`. For now keep in mind
that all this message shuffling is done to ensure message ordering
from an external observer's point of view. `q2` only has messages if
`q1` had messages before, and `q1` pushes messages to `q2` only when
some messages have been paged before, so whatever is on disk, comes
before than whatever is on `q2`.

maybe_deltas_to_betas/1はindexを読み込み、メッセージがあればq3の後ろにそれをロードする。
もしディスクになにもなければ、q2にあるメッセージはq3の後ろに移される。
後ほどmessage pagingについて詳しくみるときになぜページングされたメッセージが無いときにだけ
q2からq3にメッセージを移すのか、そしてなぜq2メッセージをq3の後方に移すのか
が理解できるだろう。このようなメッセージの移動は外部からみた場合のメッセージ順序を
保証するために行われる。
q2はq1がそれより前にメッセージを持っていたときにのみメッセージを持つ、
そしてq1はメッセージがページングされたときにのみq2にメッセージを送る、
したがって、ディスクにあるものはq2よりも先に送られる。

Remember, messages in `q1` are recently published messages that went
there because `q3` had messages, so those are the last ones we should
deliver.

q1にあるメッセージは、q3にメッセージがあるために最近パブリッシュされたメッセージであり、
最後に送るべきメッセージである。


## Paging messages to disk ##

Disk paging starts when the function `reduce_memory_use/1` is
called. This function calls `push_alphas_to_betas/2` to start sending
messages to disk. _Alpha_ messages, are those messages where the
contents and the queue position are held on RAM, and we are trying to
convert them to _betas_, i.e.: messages where we still keep the
position of the message in RAM, but we send the contents to disk.

ディスクへのページングはreduce_memory_use/1が呼ばれたときに始まる。
これはさらにpush_alphas_to_betas/2を呼び出し、メッセージがディスクに書き込まれる。
AlphaメッセージはRAMに保持されたメッセージで、betasに送ろうとしているメッセージである。
つまり：RAMにあるがディスクに書き込もうとしているメッセージである。

When paging to disk we try to first page those messages that are going
to be delivered later, those that from a client point of view are at
the rear of the queue, so we start with `q1`'s contents. If there are
messages on disk (because we have paged out queue contents already, or
because we didn't load all the queue contents in memory), then
messages are moved from `q1` into `q2`, otherwise they go into
`q3`. Then we move messages from `q4` into `q3`.

ディスクにページングするとき、なるべく最後にデリバリされることになるメッセージを先に
ページするよう試みる。クライアントの視点からは、それはキューの後方であり、よってq1の
コンテンツから始める。もしディスクにメッセージがあるなら（キューのコンテンツをすでに
ページングしていたり、まだすべてのメッセージをメモリにロードしていない場合）、
メッセージはq1からq2に移動する。そうでなければq3に移す。そしてq4のメッセージをq3へ移す。

Keep in mind that we move messages based on a _quota_ that's
calculated taking into account how messages are in ram vs how many
messages `set_ram_duration_target/2` decided that need to be paged out
to disk. If after moving messages from _alpha_ into _beta_ this quota
hasn't been consumed entirely, we have to then push messages from
_beta_ to _delta_. _Delta_ messages are those messages where the
message content and their position are only held on disk.

メッセージの移動は、ramにどれほどのメッセージがあるか、そしてどれほどのメッセージが
set_ram_duration_target/2によってディスクにページアウトされるべきと決定されるか
によって計算されるquotaに基づいて実行される。
もしメッセージをalphaからbetaに移動してもこのquotaに到達していない場合、
betaからdeltaにメッセージを移動する必要がある。
deltaのメッセージというのは、ディスクにのみ保持されたメッセージである。

To page to disk those messages whose position in the queue was still
on RAM we call `push_betas_to_deltas/2`. We first page messages from
`q3` into disk, and then we page messages from `q2`, but there's a
catch. Keep in mind that we might not want to page every single
message out to disk. `q3` holds messages that are in front of the
queue compared to those in `q2`, so `q3` messages are paged in reverse
order, that is, those messages at the rear of `q3` are sent to disk
first, with the idea that if the quota of messages that need paging is
reached, then we will keep in RAM messages that will be sent soon to
clients.

まだキューにおける位置がまだRAMの中にあるメッセージをディスクにページするには
push_betas_to_deltas/2を呼ぶ。まずq3のメッセージをページし、次にq2からページする。
しかし問題がある。我々は個々のメッセージをページアウトしたいわけではない。
q3はq2に比べて先頭にあるメッセージを持っているため、q3のメッセージは
逆順にページされる。つまり、q3の後方にあるメッセージが先にディスクに送られ、
ページすべきメッセージの量に達したら、すぐにクライアントに送られることになる
メッセージはRAMに残しておく。

## Example ##

**publish msgs `[1, 2, 3, 4, 5, 6, 7, 8, 9]`**:

```
Q4: [1, 2, 3, 4, 5, 6, 7, 8, 9]

Q3: []

Q2: []

Q1: []

Delta: []
```

**publish msgs `[10]`**:

```
q4: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

Q3: []

Q2: []

Q1: []

Delta: []
```

**push_alphas_to_betas**:

```
Q4: [1, 2, 3, 4, 5, 6, 7]

Q3: [8, 9, 10]

Q2: []

Q1: []

Delta: []
```

**publish msgs `[11, 12, 13, 14, 15]`**:

```
Q4: [1, 2, 3, 4, 5, 6, 7]

Q3: [8, 9, 10]

Q2: []

Q1: [11, 12, 13, 14, 15]

Delta: []
```

**push_alphas_to_betas**:

```
Q4: [1, 2, 3, 4]

Q3: [5, 6, 7, 8, 9, 10, 11, 12, 13]

Q2: []

Q1: [14, 15]

Delta: []
```

**push_betas_to_deltas**:

```
Q4: [1, 2, 3, 4]

Q3: [5, 6, 7, 8, 9]

Q2: []

Q1: [14, 15]

Delta: [10, 11, 12, 13]
```

**publish msgs `[16, 17, 18, 19, 20]`**:

```
Q4: [1, 2, 3, 4]

Q3: [5, 6, 7, 8, 9]

Q2: []

Q1: [14, 15, 16, 17, 18, 19, 20]

Delta: [10, 11, 12, 13]
```

**push_alphas_to_betas**:

```
Q4: [1]

Q3: [2, 3, 4, 5, 6, 7, 8, 9]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**fetch 3 messages**:

```
Q4: []

Q3: [4, 5, 6, 7, 8, 9]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**fetch 5 messages**:

```
Q4: []

Q3: [9]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**fetch 1 message**:

```
Q4: []

Q3: []

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: [10, 11, 12, 13]
```

**Q3 became empty, but we have msgs on disk/delta, so we call
maybe_deltas_to_betas to load messages from delta into Q3**:

```
Q4: []

Q3: [10, 11, 12, 13]

Q2: [14, 15, 16, 17]

Q1: [18, 19, 20]

Delta: []
```

**maybe_deltas_to_betas saw that now there are no more messages on
disk, so it joins Q3 with Q2, Q2 messages going to the rear of Q3**:

```
Q4: []

Q3: [10, 11, 12, 13, 14, 15, 16, 17]

Q2: []

Q1: [18, 19, 20]

Delta: []
```

**publish msgs `[21, 22, 23, 24, 25]`**:

```
Q4: []

Q3: [10, 11, 12, 13, 14, 15, 16, 17]

Q2: []

Q1: [18, 19, 20, 21, 22, 23, 24, 25]

Delta: []
```

**fetch 8 messages**:

```
Q4: []

Q3: []

Q2: []

Q1: [18, 19, 20, 21, 22, 23, 24, 25]

Delta: []
```

**Q3 is empty and Delta is empty as well, time to move Q1 messages into
Q4**:

```
Q4: [18, 19, 20, 21, 22, 23, 24, 25]

Q3: []

Q2: []

Q1: []

Delta: []
```

**fetch 1 message**:

```
Q4: [19, 20, 21, 22, 23, 24, 25]

Q3: []

Q2: []

Q1: []

Delta: []
```

and so on.
