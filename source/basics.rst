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
    
    どうI/Oがバックグラウンドで処理されるかというのは我々の感心事ではありませんが、コンピュータのハードウェアの処理方法と、プロセッサの基本単位としてのスレッドの関係のために、libuvと各種のOSは通常バックグラウンド/ワーカスレッドと、処理を行うためのノンブロックキングなポーリングを選択もしくは併用しています。

libuvのコア開発者の一人であるBert Belderはlibuvのアーキテクチャとその背景を短い動画にしている。もしlibuvもしくはlibevに関する経験がなければ、素早く理解するために有用なものになります。

.. raw:: html

    <iframe width="560" height="315"
    src="https://www.youtube-nocookie.com/embed/nGn60vDSxQ4" frameborder="0"
    allowfullscreen></iframe>

Hello World
-----------

基本はさておき、最初のlibuvプログラムを書いてみましょう。すぐにexitするループを開始する以外には何もありません。

.. rubric:: helloworld/main.c
.. literalinclude:: ../code/helloworld/main.c
    :linenos:
.. code-block:: C

    #include <stdio.h>
    #include <uv.h>

    int main() {
        uv_loop_t *loop = uv_loop_new();

        printf("Now quitting.\n");
        uv_run(loop, UV_RUN_DEFAULT);

        return 0;
    }

プログラムは処理すべきイベントがないため即座に終了します。libuvのイベントループは様々な関数を用いることによって、イベントを監視するように指示される必要があります。

デフォルトループ
++++++++++++

デフォルトループはlibuvによって提供され、 ``uv_default_loop()`` を用いてアクセスすることができます。ループを一つだけ欲しい場合はこのループを用いるべきです。。

.. note::

    node.jsはメインループとしてこのデフォルトループを用います。もしバインディングを作成する場合はこのことを気に留めておく必要があります。

エラーハンドリング
--------------

失敗する可能性があるlibuvの関数はエラー時に ``-1`` を返却します。エラーコードそのものはイベントループ上で ``last_err`` としてセットされます。エラーコードとして ``code`` メンバを持つ ``uv_err_t`` 構造体を取得するために ``uv_last_error(loop)`` を使いましょう。 ``code`` はここに定義されるように ``UV_*`` の列挙です。

.. rubric:: libuv error codes
.. literalinclude:: ../libuv/include/uv.h
    :lines: 69-127

libuvのエラーコード

.. code-block:: C

    XX( -1, UNKNOWN, "unknown error")                                           \
    XX(  0, OK, "success")                                                      \
    XX(  1, EOF, "end of file")                                                 \
    XX(  2, EADDRINFO, "getaddrinfo error")                                     \  
    XX(  3, EACCES, "permission denied")                                        \
    XX(  4, EAGAIN, "resource temporarily unavailable")                         \
    XX(  5, EADDRINUSE, "address already in use")                               \
    XX(  6, EADDRNOTAVAIL, "address not available")                             \
    XX(  7, EAFNOSUPPORT, "address family not supported")                       \
    XX(  8, EALREADY, "connection already in progress")                         \
    XX(  9, EBADF, "bad file descriptor")                                       \
    XX( 10, EBUSY, "resource busy or locked")                                   \
    XX( 11, ECONNABORTED, "software caused connection abort")                   \
    XX( 12, ECONNREFUSED, "connection refused")                                 \
    XX( 13, ECONNRESET, "connection reset by peer")                             \
    XX( 14, EDESTADDRREQ, "destination address required")                       \
    XX( 15, EFAULT, "bad address in system call argument")                      \
    XX( 16, EHOSTUNREACH, "host is unreachable")                                \
    XX( 17, EINTR, "interrupted system call")                                   \
    XX( 18, EINVAL, "invalid argument")                                         \
    XX( 19, EISCONN, "socket is already connected")                             \
    XX( 20, EMFILE, "too many open files")                                      \
    XX( 21, EMSGSIZE, "message too long")                                       \
    XX( 22, ENETDOWN, "network is down")                                        \
    XX( 23, ENETUNREACH, "network is unreachable")                              \
    XX( 24, ENFILE, "file table overflow")                                      \
    XX( 25, ENOBUFS, "no buffer space available")                               \
    XX( 26, ENOMEM, "not enough memory")                                        \
    XX( 27, ENOTDIR, "not a directory")                                         \
    XX( 28, EISDIR, "illegal operation on a directory")                         \
    XX( 29, ENONET, "machine is not on the network")                            \
    XX( 31, ENOTCONN, "socket is not connected")                                \
    XX( 32, ENOTSOCK, "socket operation on non-socket")                         \
    XX( 33, ENOTSUP, "operation not supported on socket")                       \
    XX( 34, ENOENT, "no such file or directory")                                \
    XX( 35, ENOSYS, "function not implemented")                                 \
    XX( 36, EPIPE, "broken pipe")                                               \
    XX( 37, EPROTO, "protocol error")                                           \
    XX( 38, EPROTONOSUPPORT, "protocol not supported")                          \
    XX( 39, EPROTOTYPE, "protocol wrong type for socket")                       \
    XX( 40, ETIMEDOUT, "connection timed out")                                  \
    XX( 41, ECHARSET, "invalid Unicode character")                              \
    XX( 42, EAIFAMNOSUPPORT, "address family for hostname not supported")       \
    XX( 44, EAISERVICE, "servname not supported for ai_socktype")               \
    XX( 45, EAISOCKTYPE, "ai_socktype not supported")                           \
    XX( 46, ESHUTDOWN, "cannot send after transport endpoint shutdown")         \
    XX( 47, EEXIST, "file already exists")                                      \
    XX( 48, ESRCH, "no such process")                                           \
    XX( 49, ENAMETOOLONG, "name too long")                                      \
    XX( 50, EPERM, "operation not permitted")                                   \
    XX( 51, ELOOP, "too many symbolic links encountered")                       \
    XX( 52, EXDEV, "cross-device link not permitted")                           \
    XX( 53, ENOTEMPTY, "directory not empty")                                   \
    XX( 54, ENOSPC, "no space left on device")                                  \
    XX( 55, EIO, "i/o error")                                                   \
    XX( 56, EROFS, "read-only file system")                                     \
    XX( 57, ENODEV, "no such device")                                           \
    XX( 58, ESPIPE, "invalid seek")                                             \
    XX( 59, ECANCELED, "operation canceled")                                    \

``uv_strerror(uv_err_t)`` と ``uv_err_name(uv_err_t)`` 関数を用いると、エラーやエラーの名前が格納された ``const char*`` を取得することができます。

非同期コールバックは最後の引数として ``status`` 引数を持ちます。これを戻り値の代わりに使ってください。

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
