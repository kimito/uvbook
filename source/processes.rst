プロセス
=========

libuvはかなりの量の子プロセス管理機能を提供しており、プラットフォーム間の差異を抽象化し、ストリームや名前付きパイプを用いて子プロセスと通信することを可能にしています。

Unixにおける共通のイディオムは全てのプロセスが一つのことを良好に行えることです。このようなケースでは、プロセスはタスクを完遂するために複数の子プロセスを使用します(シェルがパイプを用いるように)。メッセージを用いるマルチプロセスモデルはスレッドと共有メモリを用いるモデルに比較して簡単になるでしょう。

イベントベースのプログラムに対する共通のマイナス要因は現代のコンピュータにおいて複数のコアを持つ利点を活用しきれない点です。マルチスレッドプログラムではカーネルはスケジューリングを行い、異なるスレッドに異なるコアを割り当てることができます。しかし、イベントループは単一のスレッドです。これに対する回避策は代わりに複数のプロセスを起動することであり、各プロセスはイベントループを処理し、各プロセスは別々のコアを割り当てられます。

子プロセスの起動
------------------------

最も単純なケースはプロセスを起動し、いつプロセスが終了したかを知ることです。これは ``uv_spawn`` を用いることで達成されます。

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 5-7,13-
    :emphasize-lines: 11,13-17

.. NOTE::

    ``options`` はグローバル変数であるため、明示的に0で初期化されています。もし ``options`` とローカル変数に変える場合、使用しない全てのフィールドをnullに初期化することを忘れないでください。

        uv_process_options_t options = {0};

``uv_process_t`` 構造体はウォッチャとしてのみ動作し、全てのオプションは ``uv_process_options_t`` を通じて設定します。プロセスの起動を簡単にするために、 ``file`` と ``args`` フィールドのみを設定する必要があります。 ``file`` は実行するプログラムです。 ``uv_spawn`` は内部的に execvp_ を内部的に使用するため、フルパスを渡す必要はありません。最後に、基本的な規則として、 **引数の配列は引数の数より一つ大きく、最後の要素はNULLである必要があります。**

.. _execvp: http://www.kernel.org/doc/man-pages/online/pages/man3/exec.3.html

``uv_spawn`` の呼び出しのあとに、 ``uv_process_t.pid`` には子プロセスのプロセスIDが格納されています。

exitコールバックは *終了ステータス* と exitの原因となった *signal* の種類とともに呼び出されます。

.. rubric:: spawn/main.c
.. literalinclude:: ../code/spawn/main.c
    :linenos:
    :lines: 9-12
    :emphasize-lines: 3

プロセスが終了した後にプロセスのウォッチャをクローズすることが **必要です。**

プロセスのパラメータ変更
---------------------------

子プロセスが起動する前に ``uv_process_options_t`` 内のフィールドを用いて実行環境を制御することができます。

実行ディレクトリの変更
++++++++++++++++++++++++++

``uv_process_options_t.cwd`` を対応するディレクトリに設定します。

環境変数の設定
+++++++++++++++++++++++++

``uv_process_options_t.env`` は文字列のヌル終端された配列であり、 各 ``VAR=VALUE`` 形式はプロセスの環境変数を設定するために用いられます。これを ``NULL`` にすることにより親プロセスからからの環境変数を受け継ぎます。

オプションフラグ
++++++++++++

``uv_process_options_t.flags`` に下記のビット単位のORフラグを設定することにより、子プロセスの動作を修正します。

* ``UV_PROCESS_SETUID`` - これは子の実行ユーザIDを ``uv_process_options_t.uid`` に設定します。
 ``UV_PROCESS_SETGID`` - これは子の実行グループIDを ``uv_process_options_t.gid`` に設定します。

UID/GIDの変更はUnixのみでサポートされており、 ``uv_spawn`` はWindowsでは ``UV_ENOTSUP`` で失敗します。

* ``UV_PROCESS_WINDOWS_VERBATIM_ARGUMENTS`` - ``uv_process_options_t.args`` にクオートやエスケープを付与しません。Unixでは無視されます。
* ``UV_PROCESS_DETACHED`` - 新しいセッションで子プロセスを開始し、親プロセスが終了した後も処理を続けます。下記の例を参照してください。

プロセスをデタッチする
-------------------

``UV_PROCESS_DETACHED`` フラグを渡すことで、親の終了が影響しないデーモン、もしくは親と独立した子プロセスを起動することができます。

.. rubric:: detach/main.c
.. literalinclude:: ../code/detach/main.c
    :linenos:
    :lines: 9-30
    :emphasize-lines: 12,19

ただ、ウォッチャはまだ子を監視しているため、プログラムは終了しないことをを忘れないでください。もっと *発動したら忘れる* ことを求めるなら、 ``uv_unref()`` を使用してください。

プロセスへのシグナル送信
----------------------------

libuvはUnix標準の ``kill(2)`` システムコール、及びWindowsにおいて同様の機能性を実装していますが、一つ警告があります。 ``SIGTERM`` 、 ``SIGINT`` と ``SIGKILL`` はプロセスを終了させます。 ``uv_kill`` のシグネチャは以下です ::

    uv_err_t uv_kill(int pid, int signum);

libuvを用いて開始されたプロセスのために、 ``uv_process_kill`` を代わりに用いることができます。これは pidの代わりに ``uv_process_t`` ウォッチャを第一引数として受け取ります。この場合、ウォッチャに対して ``uv_close`` を **呼び出すことを忘れないでください。**

シグナル
-------

TODO: update based on https://github.com/joyent/libuv/issues/668

libuvは `some Windows support
<https://github.com/joyent/libuv/blob/node-v0.9.4/include/uv.h#L1659>`_ によってUnixシグナルに対するラッパも提供します。

libuvにより ’よく動作する' シグナルを生成するために、あるAPIが *全ての実行中のイベントループ上の全てのハンドラ* に対してシグナルを送信します! ハンドラを初期化し、ループに関連付けるために``uv_signal_init()`` を使用してください。特定のシグナルを待ち受けるために、ハンドラ関数とともに ``uv_signal_start()`` を使用してください。 各ハンドラは一つのシグナル番号に対してのみ関連付け、 ``uv_signal_start()`` を続けて呼び出すことで、以前の関連付けを上書きすることができます。監視を中止するためには ``uv_signal_stop()`` を使用してください。以下はいろいろな使い方を示した小さな例です :

.. rubric:: signal/main.c
.. literalinclude:: ../code/signal/main.c
    :linenos:
    :emphasize-lines: 8,17-18

.. NOTE::

    ``uv_run(loop, UV_RUN_NOWAIT)`` is similar to ``uv_run(loop, UV_RUN_ONCE)``
    in that it will process only one event. UV_RUN_ONCE blocks if there are no
    pending events, while UV_RUN_NOWAIT will return immediately. We use NOWAIT
    so that one of the loops isn't starved because the other one has no pending
    activity.
    
    ``uv_run(loop, UV_RUN_NOWAIT)`` は これがたった一つのイベントしか処理しない点で ``uv_run(loop, UV_RUN_ONCE)`` と同様です。UV_RUN_NOWAITがすぐに制御を戻すのに対し、 UV_RUN_ONCEは保留されたイベントがない場合にブロックします。NOWAITは他のループが保留された処理を持っていないという理由でループがロック(starved)しないようにするために用いることができます。

``SIGUSR1`` をプロセスに送信すると、各 ``uv_signal_t`` で１つずつ、4回ハンドラが起動していることを確認することができます。ハンドラはプログラムを終了するために各ハンドルを停止するだけです。全てのハンドラに対してのこの種の送信はとても有用です。単純に各イベントループに ``SIGINT`` のためのウォッチャを追加するだけで、複数のイベントループを使用するサーバは終了する前にデータが安全に保存されることを保証できます。

Child Process I/O
-----------------

通常の、新規に起動したプロセスは自分自身のファイルディスクリプタのセットを持っており、0、1、2、はそれぞれ ``stdin`` 、 ``stdout`` 、 ``stderr`` に対応しています。ときどきファイルディスクリプタを子プロセスと共有したい場合があります。例えば、アプリケーションはサブコマンドを起動し、発生したエラーをログファイルに書きたいが ``stdout`` は無視したいかもしれません。このために子プロセスの ``stderr`` を表示したいとします。この場合、libuvはファイルディスクリプタの *継承*  をサポートしています。この例では、下記のようなテストプログラムを起動します:

.. rubric:: proc-streams/test.c
.. literalinclude:: ../code/proc-streams/test.c

実際のプログラムである ``proc-streams`` は ``stderr`` のみを継承して実行されます。 ``uv_process_options_t`` の ``stdio`` フィールドを使用することで子プロセスのファイルディスクリプタは設定されます。最初に ``stdio_count`` フィールドに設定するファイルディスクリプタの数を設定します。 ``uv_process_options_t.stdio`` は ``uv_stdio_container_t`` の配列であり、下記のようになります:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1263-1270

ここで、フラグはいくつかの値を持つことができます。 これが使われない場合に ``UV_IGNORE`` を用いてください。もし最初の3つの ``stdio`` フィールドが ``UV_IGNORE`` に設定された場合、これらは ``/dev/null`` にリダイレクトされます。

既存のディスクリプタに渡したいため、 ``UV_INHERIT_FD`` を使用します。それから ``fd`` を ``stderr`` に設定します。

.. rubric:: proc-streams/main.c
.. literalinclude:: ../code/proc-streams/main.c
    :linenos:
    :lines: 15-17,27-
    :emphasize-lines: 6,10,11,12

``proc-stream`` を実行すると、"This is stderr" という行だけが表示されます。 ``stdout`` を継承するように設定して出力を確認してください。

これはこのようなストリームのリダイレクションを適用したあまり感動のない単純なものです。 ``flags`` を ``UV_INHERIT_STREAM`` に設定し、 ``data.stream`` を親プロセスのストリームに設定することにより、子プロセスはそのプロセスを標準I/Oとして扱うことができます。これは CGI_ のようなものを実装するのに使用できます。

.. _CGI: http://en.wikipedia.org/wiki/Common_Gateway_Interface

CGI スクリプト/実行ファイルの例は以下になります:

.. rubric:: cgi/tick.c
.. literalinclude:: ../code/cgi/tick.c

このCGIサーバは、この章のコンセプトと :doc:`ネットワーク` を結びつけており、全てのクライアントは接続が閉じられるまでに10回"tick"を受信します。

.. rubric:: cgi/main.c
.. literalinclude:: ../code/cgi/main.c
    :linenos:
    :lines: 47,53-62
    :emphasize-lines: 5

ここでTCP接続を受け付け、 ``invoke_cgi_script`` にソケット (*stream*) を渡します。

.. rubric:: cgi/main.c
.. literalinclude:: ../code/cgi/main.c
    :linenos:
    :lines: 16, 25-45
    :emphasize-lines: 8-9,17-18

CGIスクリプトの ``stdout`` はtickスクリプトが出力する全ての内容をクライアントに送信させるためにソケットに設定されています。プロセスを用いることにより、read/writeのバッファリングをOSに任せることができ、これは簡単さの観点では素晴らしいことです。ただ、プロセスの生成はコストがかかる処理であることは肝に命じてください。

.. _pipes:

パイプ
-----

libuv's ``uv_pipe_t`` structure is slightly confusing to Unix programmers,
because it immediately conjures up ``|`` and `pipe(7)`_. But ``uv_pipe_t`` is
not related to anonymous pipes, rather it has two uses:

libuvの ``uv_pipe_t`` 構造体はUnixプログラマーを少し混乱させるもので、なぜならこれはすぐに ``|`` と `pipe(7)`_ を呼び出すからです。しかし ``uv_pipe_t`` は無名パイプとは関連がなく、これは2つを用いています:

#. ストリームAPI - これは ``uv_stream_t`` の具体的な実装として実行されるものです。FIFOを提供するためのAPIであり、ローカルファイルのI/Oに対するストリーミングインターフェイスです。これは :ref:`buffers-and-streams` で言及される ``uv_pipe_open`` を用いて処理されます。これをTCP/UDPのためにも用いることができますが、この目的のためには既に便利な関数と構造体が用意されています。

#. IPC機構 - ``uv_pipe_t`` は `Unixドメインソケット`_ か `Windowsの名前付きパイプ` を複数のプロセスが通信できるように利用することができます。

.. _pipe(7): http://www.kernel.org/doc/man-pages/online/pages/man7/pipe.7.html
.. _Unixドメインソケットt: http://www.kernel.org/doc/man-pages/online/pages/man7/unix.7.html
.. _Windowsの名前付きパイプ: http://msdn.microsoft.com/en-us/library/windows/desktop/aa365590(v=vs.85).aspx

親-子のIPC
++++++++++++++++

親と子はパイプを通じた一方向もしくは双方向の通信を行うことができ、パイプは ``uv_stdio_container_t.flags`` にビット単位の``UV_CREATE_PIPE`` と ``UV_READABLE_PIPE`` もしくは ``UV_WRITABLE_PIPE`` の組み合わせを設定することによって作成することができます。read/writeフラグは子プロセス側の観点のフラグです。

任意のプロセスのIPC
+++++++++++++++++++++

ドメインソケットはファイルシステム内にウェルノウンネームとロケーションを持つことができるため関連のないプロセス間でIPCを使用することができます。 オープンソースのデスクトップ環境で使用されている D-BUS_ システムはイベント通知のためにドメインソケットを使用しています。さまざまなアプリケーションがオンライン上のコンタクトが来た時や新しいハードウェアが検知された時に反応することができます。MySQLサーバもクライアントとやりとりするドメインソケットを実行します。

.. _D-BUS: http://www.freedesktop.org/wiki/Software/dbus

ドメインソケットを使用するとき、クライアント-サーバパターンが使用され、ソケットの作成者/オーナがサーバとして動作します。初期化の後は、通信の方法はTCPと変わりがありませんので、エコーサーバの例をもう一度持ち出します。

.. rubric:: pipe-echo-server/main.c
.. literalinclude:: ../code/pipe-echo-server/main.c
    :linenos:
    :lines: 56-
    :emphasize-lines: 5,9,13

ソケットにはローカルディレクトリに作成されるという意味で ``echo.sock`` と名前をつけました。このソケットはストリームAPIが関係する限りはTCPソケットと変わらない振る舞いをします。このサーバは `netcat`_ を用いてサーバをテストすることができます::

    $ nc -U /path/to/echo.sock

ドメインソケットに接続したいクライアントは以下を使用します::

    void uv_pipe_connect(uv_connect_t *req, uv_pipe_t *handle, const char *name, uv_connect_cb cb);

ここで ``name`` は ``echo.sock`` もしくは同様のものです。

.. _netcat: http://netcat.sf.net

Sending file descriptors over pipes
+++++++++++++++++++++++++++++++++++

ドメインソケットのクールな点はドメインソケットを介して送信することによりファイルディスクリプタを交換できる点です。これはプロセスに他のプロセス間でI/Oを受け渡すことを可能にします。ロードバランスを行うサーバ、ワーカプロセスや他の手法を含むアプリケーションは最適なCPU利用を行うことができます。

.. warning::

    Windows上ではTCPソケットを表すファイルディスクリプタだけを受け渡しすることができます。

実演するために、ラウンドロビン方式でクライアントをワーカプロセスに引き渡すエコーサーバの実装を見てみましょう。このプログラムは少し複雑で、書籍内には少しのスニペットしかないので、本当に理解するには全てのコードを読むことをおすすめします。

ファイルディスクリプタがマスターから渡されるので、ワーカプロセスは本当に単純です。

.. rubric:: multi-echo-server/worker.c
.. literalinclude:: ../code/multi-echo-server/worker.c
    :linenos:
    :lines: 7-9,53-
    :emphasize-lines: 7-9

一方、 ``queue`` は　マスタープロセスに接続されたパイプであり、送信された新しいファイルディスクリプタです。 ``read2`` 関数を用いてファイルディスクリプタに対する興味(interest)を表現します。　``uv_pipe_init()`` の ``ipc`` 引数を1に設定することが重要で、これはパイプがプロセス間通信に使用されることを表します! マスターはファイルハンドルをワーカの標準入力に書き込むため、パイプを ``uv_pipe_open`` を用いて ``stdin`` に接続します。

.. rubric:: multi-echo-server/worker.c
.. literalinclude:: ../code/multi-echo-server/worker.c
    :linenos:
    :lines: 36-52
    :emphasize-lines: 9

``accept``はこのコード内で奇妙に思えますが、これは的を得ているものです。 ``accept`` がしていることは、伝統的に他のファイルディスクリプタ(リスニングソケット)からファイルディスクリプタ(クライアント)を得ることだからです。これはまさにここでしていることです。 ``queue`` からファイルディスクリプタ (``client``) を取得してください。ここからはワーカは標準的なエコーサーバの内容を行います。

マスターに戻って、ロードバランスを可能にするためにワーカがどのように起動されるかを見てみましょう。

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 6-13

``child_worker`` 構造体はプロセスと、マスターと各プロセスの間のパイプをラップしています。

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 49,61-93
    :emphasize-lines: 15,18-19

ワーカの準備の中で、 ``uv_cpu_info`` というちょっとかっこいい関数を使っており、これはCPUの数を取得することができるのでワーカの数をこれと同じにすることができます。再度第三引数を1にしてパイプをIPCとして初期化することの重要性を述べておきます。それから子プロセスの ``stdin`` を(子プロセスの観点から)読み取り可能なパイプであることを指定します。ここまで全て正攻法です。ワーカは起動し、パイプにファイルディスクリプタが書き込まれるを待ちます。

``on_new_connection`` (TCP周りは ``main()`` で初期化されています)で、クライアントソケットを待ち受け、ラウンドロビンの中の次のワーカにソケットを渡します。

.. rubric:: multi-echo-server/main.c
.. literalinclude:: ../code/multi-echo-server/main.c
    :linenos:
    :lines: 29-47
    :emphasize-lines: 9,12-13

もう一度、 ``uv_write2`` が全ての抽象化を制御し、正しい引数としてファイルディスクリプタを渡すことが問題であることを述べておきます。これでマルチプロセスのエコーサーバの出来上がりです。

TODO what do the write2/read2 functions do with the buffers?

----

.. [#] 同様にこの章ではドメインソケットはWindows上では名前付きパイプを用いて実装されています。