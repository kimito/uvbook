libuvの基礎
===============

libuvは **非同期**、 **イベント駆動** のプログラミングスタイルを強制します。 libuvの中心的な機能はイベントループとI/Oと他の活動(activity)の通知をベースにしたコールバックを提供することです。libuvはタイマのようなユーティリティ、ノンブロッキングのネットワークのサポート、非同期のファイルシステムへのアクセス、子プロセス等を提供します。

イベントループ
-----------

イベント駆動のプログラミングにおいては、アプリケーションは特定のイベントに対する興味を表明し、そのイベントが発生した時にこれらに反応します。OSか他の発生源からイベントを収集する責任はlibuvによって取り扱われ、ユーザはイベントが発生した時に実行されるコールバックを登録することができます。イベントループは通常 *永久に* 実行され続けます。擬似コードは以下のようになります:

.. code-block:: python

    while there are still events to process:
        e = get the next event
        if there is a callback associated with e:
            call the callback

イベントの例には以下のようなものがあります 

* ファイルの書き込み準備ができている
* ソケットが読み取り可能なデータを持っている
* タイマが設定したタイミングに達した

このイベントループは ``uv_run()`` -- libuvを用いたときに結局実行される関数 -- によって隠蔽されています。

システムプログラミングのもっとも共通な処理は、複雑な計算というよりもむしろ入力と出力を取り扱うことです。一般的な入力/出力関数(``read``, ``fprintf``, etc.)を用いる問題点は、これらが **ブロッキング** であるということです。ハードディスクに対する実際の書き込み、もしくはネットワークからのデータ読み出しは、プロセッサのスピードに比較して絶望的に長い時間を必要とします。これらの関数は処理が完了するまで制御が戻らないため、その間プログラムは何も行うことができません。高いパフォーマンスを必要とするプログラムにとって、これは他の処理や他のI/O操作が待たされ続けるため重要な障害となります。

標準的な解決策の一つはスレッドを用いることです。各ブロッキングI/O操作は分離されたスレッド(もしくはスレッドプール)の中で開始されます。スレッド内でブロッキング関数が実行されたとき、プロセッサは実際にCPU処理を必要としている実行すべき他のスレッドをスケジューリングすることができます。

libuvで採用されているアプローチはこれとは違うもので、**非同期、ノンブロッキング**なスタイルを用いています。ほとんどのモダンなOSはイベント通知のためのサブシステムを提供しています。例えば、ソケットに対する通常の ``read`` 呼び出しは送信側が何かを送信するまでブロックする可能性があります。それに対して、アプリケーションはOSにソケットを監視することを要求し、キューにイベントの通知を投入してもらうことができます。アプリケーションは都合の良い時(おそらくプロセッサを最大限に使う前にいくつかをガリガリ処理する)にイベントを解析しデータを把握します。これはアプリケーションがある時点で興味を示し、(時間的/空間的に)別の時点でデータを利用するために**非同期**です。また、これはアプリケーションは他の処理を行うことが自由であるために**ノンブロッキング**です。これはOSのイベントが単にlibuvイベントの別種として扱えるために、libuvのイベントループによるアプローチとよく合っていると言えます。

.. NOTE::
    
    どうI/Oがバックグラウンドで処理されるかというのは我々の感心事ではないが、コンピュータのハードウェアの処理方法と、プロセッサの基本単位としてのスレッドの関係のために、libuvと各種のOSは通常バックグラウンド/ワーカスレッドと、処理を行うためのノンブロックキングなポーリングを選択もしくは併用している。

libuvのコア開発者の一人であるBert Belderはlibuvのアーキテクチャとその背景を短い動画にしている。もしlibuvもしくはlibevに関する経験がなければ、素早く理解するために有用なものになります。

.. raw:: html

    <iframe width="560" height="315"
    src="https://www.youtube-nocookie.com/embed/nGn60vDSxQ4" frameborder="0"
    allowfullscreen></iframe>

Hello World
-----------

With the basics out of the way, lets write our first libuv program. It does
nothing, except start a loop which will exit immediately.

.. rubric:: helloworld/main.c
.. literalinclude:: ../code/helloworld/main.c
    :linenos:

This program quits immediately because it has no events to process. A libuv
event loop has to be told to watch out for events using the various API
functions.

Default loop
++++++++++++

A default loop is provided by libuv and can be accessed using
``uv_default_loop()``. You should use this loop if you only want a single
loop.

.. note::

    node.js uses the default loop as its main loop. If you are writing bindings
    you should be aware of this.

Error handling
--------------

libuv functions which may fail return ``-1`` on error. The error code itself is
set on the event loop as ``last_err``. Use ``uv_last_error(loop)`` to get
a ``uv_err_t`` which has a ``code`` member with the error code. ``code`` is an
enumeration of ``UV_*`` as defined here:

.. rubric:: libuv error codes
.. literalinclude:: ../libuv/include/uv.h
    :lines: 69-127

You can use the ``uv_strerror(uv_err_t)`` and ``uv_err_name(uv_err_t)`` functions
to get a ``const char *`` describing the error or the error name respectively.

Async callbacks have a ``status`` argument as the last argument. Use this instead
of the return value.

Watchers
--------

Watchers are how users of libuv express interest in particular events. Watchers
are opaque structs named as ``uv_TYPE_t`` where type signifies what the watcher
is used for. A full list of watchers supported by libuv is:

.. rubric:: libuv watchers
.. literalinclude:: ../libuv/include/uv.h
    :lines: 190-207

.. note::

    All watcher structs are subclasses of ``uv_handle_t`` and often referred to
    as **handles** in libuv and in this text.

Watchers are setup by a corresponding::

    uv_TYPE_init(uv_TYPE_t*)

function.

.. note::

    Some watcher initialization functions require the loop as a first argument.

A watcher is set to actually listen for events by invoking::

    uv_TYPE_start(uv_TYPE_t*, callback)

and stopped by calling the corresponding::

    uv_TYPE_stop(uv_TYPE_t*)

Callbacks are functions which are called by libuv whenever an event the watcher
is interested in has taken place. Application specific logic will usually be
implemented in the callback. For example, an IO watcher's callback will receive
the data read from a file, a timer callback will be triggered on timeout and so
on.

Idling
++++++

Here is an example of using a watcher. An idle watcher's callback is repeatedly
called. There are some deeper semantics, discussed in :doc:`utilities`, but
we'll ignore them for now. Let's just use an idle watcher to look at the
watcher life cycle and see how ``uv_run()`` will now block because a watcher is
present. The idle watcher is stopped when the count is reached and ``uv_run()``
exits since no event watchers are active.

.. rubric:: idle-basic/main.c
.. literalinclude:: ../code/idle-basic/main.c
    :emphasize-lines: 6,10,14-17

void \*data pattern

note about not necessarily creating type structs on the stack

----

.. [#] Depending on the capacity of the hardware of course.
