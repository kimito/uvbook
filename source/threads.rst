スレッド
=======

ちょっと待って下さい? なぜスレッドの話をするのですか? イベントループは *web-scaleプログラミング* を行うための *手段* だったのではないのですか? いいえ、違います。スレッドはまだプロセッサが処理を行う際の手段であり、同期プリミティブを苦労して使う必要があるとしても、スレッドは時々かなり有用です。

スレッドは全てのシステムコールを非同期の性質を持つかのように装わせるために内部的に用いられています。libuvはまた、スレッドを起動してタスクが終了した時に結果を収集することにより、実際はブロックするタスクを非同期的に実行するためにスレッドを用いています。

現在、２つの有力なスレッドライブラリが存在します。Windowsのスレッド実装と `pthreads`_ です。libuvのスレッドAPIはpthread APIによく似ており、しばしば同様の部分があります。

libuvのスレッド機能の特筆すべき点は、これがlibuvそのものに含まれている(self contained)点です。他の機能がイベントループやコールバックと密接に関係しているのに対して、スレッドは完全に隠蔽されており、戻り値を直接経由したシグナルエラーを必要に応じてブロックし、 :ref:`最初の例 <thread-create-example>` で見たようにイベントループを実行する必要すらありません。

libuvのスレッドAPIはスレッドの意味や文法がプラットフォームごとに全て異なり、完全さの店でもレベルが異なるのでとても制限されています。

この章では以下の仮定を行います: **一つのイベントループだけが存在し、単一(メイン)のスレッド上で動作する** （ ``uv_async_send`` を用いた場合を除き）イベントループは他のスレッドと関連することはありません。 :doc:`multiple` は異なる複数のスレッドでイベントループを実行し、これらを管理します。

コアとなるスレッド操作
----------------------

たくさんあるわけではありませんが、 ``uv_thread_create()`` を用いてスレッドを開始して ``uv_thread_join()`` を用いて終了を待ち合わせることができます。

.. _thread-create-example:

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 26-37
    :emphasize-lines: 3-7

.. code-block:: c
    
   int main() {
        int tracklen = 10;
        uv_thread_t hare_id;
        uv_thread_t tortoise_id;
        uv_thread_create(&hare_id, hare, &tracklen);
        uv_thread_create(&tortoise_id, tortoise, &tracklen);

        uv_thread_join(&hare_id);
        uv_thread_join(&tortoise_id);
        return 0;
    }    

.. tip::

    ``uv_thread_t`` はUnixにおいては単なる ``pthread_t`` の別名ですが、これは実装の詳細であり、常に成り立つことに依存することは避けてください。

第二引数はスレッドのエントリポイントとなる関数で、最後の引数はスレッドに渡すために用いられる ``void*`` のカスタム引数です。 ``hare``関数は分離されたスレッドで実行され、OSによってプリエンプティブにスケジューリングされます。

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 6-14
    :emphasize-lines: 2
    
.. code-block:: c
    
    void hare(void *arg) {
        int tracklen = *((int *) arg);
        while (tracklen) {
            tracklen--;
            sleep(1);
            fprintf(stderr, "Hare ran another step\n");
        }
        fprintf(stderr, "Hare done running!\n");
    }

第二引数を用いて起動したスレッドから呼び出し側のスレッドに値を戻すことが可能な ``pthread_join()`` とは異なり、 ``uv_thread_join()`` はこれができません。値を送信するには :ref:`スレッド間通信` を用います。

同期プリミティブ
--------------------------

この項はあえてスパルタ式になっています。この書籍はスレッドに関するものではないので、ここに上げるlibuvのAPIには特に驚くものは列挙されていません。pthreadの `man pages<pthreads>`_ を参照してください。

ミューテックス
~~~~~~~

ミューテックス関数はpthreadにおける同等のものと **直接** マッピングされています。

.. rubric:: libuv mutex functions
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1834-1838

.. code-block:: c

    UV_EXTERN int uv_mutex_init(uv_mutex_t* handle);
    UV_EXTERN void uv_mutex_destroy(uv_mutex_t* handle);
    UV_EXTERN void uv_mutex_lock(uv_mutex_t* handle);
    UV_EXTERN int uv_mutex_trylock(uv_mutex_t* handle);
    UV_EXTERN void uv_mutex_unlock(uv_mutex_t* handle);

``uv_mutex_init()`` と ``uv_mutex_trylock()`` 関数は成功時には0、エラー時にはエラーコードの代わりに-1を返却します。

`libuv` がデバッグモードでコンパイルされた場合、 ``uv_mutex_destroy()`` 、 ``uv_mutex_lock()`` と ``uv_mutex_unlock()`` はエラー時に ``abort()`` します。同様に、 ``uv_mutex_trylock()`` は ``EAGAIN`` 以外の何らかのエラー時にアボートします。

再帰ミューテックスはいくつかのプラットフォームでサポートされていますが、それらに頼るべきではありません。BSDミューテックスの実装はミューテックスがロックされたスレッドがもう一度ロックしようとするとエラーを発生させます。例えば、このように使用した場合で、 ::

    uv_mutex_lock(a_mutex);
    uv_thread_create(thread_id, entry, (void *)a_mutex);
    uv_mutex_lock(a_mutex);
    // more things here

というように用いることで他のスレッドが何か行って ``a_mutex`` をアンロックするまで待つことができますが、これはデバッグモードであればプログラムがクラッシュし、もしくは二回目の ``uv_mutex_lock()`` でエラーを返却するでしょう。

.. note::

    linuxにおけるミューテックスは再帰ミューテックスのための属性をサポートしていますが、libuv経由では公開されていません。

ロック
~~~~~

リードライトロックはより荒い粒度のアクセス制御機構です。2つのリーダ(reader)が同時に共有メモリアクセスすることができます。ライタ(writer)はリーダが保持している間はロックを獲得することができません。リーダもしくはライタはライタがロックを保持している間は獲得することができません。リードライトロックはたびたびデータベースで用いられます。以下は簡単なサンプルになります。

.. rubric:: locks/main.c - simple rwlocks
.. literalinclude:: ../code/locks/main.c
    :linenos:
    :emphasize-lines: 13,16,27,31,42,55

.. code-block:: c

    #include <stdio.h>
    #include <uv.h>

    uv_barrier_t blocker;
    uv_rwlock_t numlock;
    int shared_num;

    void reader(void *n)
    {
        int num = *(int *)n;
        int i;
        for (i = 0; i < 20; i++) {
            uv_rwlock_rdlock(&numlock);
            printf("Reader %d: acquired lock\n", num);
            printf("Reader %d: shared num = %d\n", num, shared_num);
            uv_rwlock_rdunlock(&numlock);
            printf("Reader %d: released lock\n", num);
        }
        uv_barrier_wait(&blocker);
    }

    void writer(void *n)
    {
        int num = *(int *)n;
        int i;
        for (i = 0; i < 20; i++) {
            uv_rwlock_wrlock(&numlock);
            printf("Writer %d: acquired lock\n", num);
            shared_num++;
            printf("Writer %d: incremented shared num = %d\n", num, shared_num);
            uv_rwlock_wrunlock(&numlock);
            printf("Writer %d: released lock\n", num);
        }
        uv_barrier_wait(&blocker);
    }

    int main()
    {
        uv_barrier_init(&blocker, 4);

        shared_num = 0;
        uv_rwlock_init(&numlock);

        uv_thread_t threads[3];

        int thread_nums[] = {1, 2, 1};
        uv_thread_create(&threads[0], reader, &thread_nums[0]);
        uv_thread_create(&threads[1], reader, &thread_nums[1]);

        uv_thread_create(&threads[2], writer, &thread_nums[2]);

        uv_barrier_wait(&blocker);
        uv_barrier_destroy(&blocker);

        uv_rwlock_destroy(&numlock);
        return 0;
    }

実行し、複数のリーダがどのように時々オーバラップするかを確認してみてください。複数のライタの場合、スケジューラはライタに高い優先度を与えます。ですので、もし２つのライタを加えた場合、リーダが再び機会を得る前に両方のライタが最初に終了するという現象となることを確認することができます。

その他
~~~~~~

libuvは `セマフォ`_ 、 `状態変数`_ と `バリア`_ をpthreadと類似のAPIでサポートしています。 

状態変数の場合、libuvはウエイトのタイムアウトをプラットフォーム依存のちょっと変わった形でで実装しています。 [#]_

.. _セマフォ: http://en.wikipedia.org/wiki/Semaphore_(programming)
.. _状態変数: http://en.wikipedia.org/wiki/Condition_variable#Waiting_and_signaling
.. _バリア: http://en.wikipedia.org/wiki/Barrier_(computer_science)

加えて、libuvは ``uv_once()`` という便利な関数を提供しています。 複数のスレッドがガードと関数ポインタを与えて ``uv_once()`` の実行を試すことができます。 **最初のスレッドだけでこれは成功し、関数はただ一度だけ実行されます。**::

    /* Initialize guard */
    static uv_once_t once_only = UV_ONCE_INIT;

    int i = 0;

    void increment() {
        i++;
    }

    void thread1() {
        /* ... work */
        uv_once(once_only, increment);
    }

    void thread2() {
        /* ... work */
        uv_once(once_only, increment);
    }

    int main() {
        /* ... spawn threads */
    }

全てのスレッドが実行し終わったあと、 ``i==1`` となります。

.. _libuv-work-queue:

libuvワークキュー
----------------

``uv_queue_work()`` は別のスレッド同士でタスクを実行するための便利な関数で、タスクが完了したらコールバックが呼ばれます。外見上シンプルな関数で、 ``uv_queue_work()`` が魅力的なのは任意の第三者のライブラリをイベントループのスタイルで用いることができる点です。イベントループを使用した場合、 *I/Oを実行するときや、CPUを酷使するときにループスレッド内で定期的に実行される関数がブロックしないことを確実にすることが必須条件となります* 。なぜなら、これが満たされない場合ループの実行頻度が落ちてしまい、最大の能力でイベントをさばくことができなくなってしまうからです。

しかし、多数の既存のコード(例えば内部的にI/Oを実行する処理)はスレッド内で用いるブロッキング関数の形で提供され、(一つのクライアントに対して一つのサーバが存在するような典型的な)応答性が求められる場合にはこのコードはスレッド上で使用され、分離したスレッドの中でタスクを実行するシステムに含まれるイベントループライブラリ中でこのコードを実行することになります。libuvはこのために便利な抽象化を提供します。

下記は `node.js is cancer`_ にインスパイアされたシンプルな例です。実行中に少しスリープしながらフィボナッチ数を計算していきますが、ブロッキングやCPU律速のタスクがイベントループの他の仕事を妨げないように計算を別々のスレッドで実行します。

.. rubric:: queue-work/main.c - lazy fibonacci
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 17-29

.. code-block:: c

    void fib(uv_work_t *req) {
        int n = *(int *) req->data;
        if (random() % 2)
            sleep(1);
        else
            sleep(3);
        long fib = fib_(n);
        fprintf(stderr, "%dth fibonacci is %lu\n", n, fib);
    }

    void after_fib(uv_work_t *req, int status) {
        fprintf(stderr, "Done calculating %dth fibonacci\n", *(int *) req->data);
    }

実際のタスク関数はシンプルでこれが分離されたスレッドで実行されること以外に特筆すべき点はありません。 ``uv_work_t`` 構造体が手がかりとなります。 ``void* data`` フィールドを用いて任意のデータを渡すことができ、スレッド間で通信をするために使用することができます。しかし、実行中の両方のスレッドでデータを変更する場合は適切なロックを使用するようにしてください。

きっかけは ``uv_queue_work`` です:

.. rubric:: queue-work/main.c
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 31-44
    :emphasize-lines: 40

.. code-block:: c

    int main() {
        loop = uv_default_loop();

        int data[FIB_UNTIL];
        uv_work_t req[FIB_UNTIL];
        int i;
        for (i = 0; i < FIB_UNTIL; i++) {
            data[i] = i;
            req[i].data = (void *) &data[i];
            uv_queue_work(loop, &req[i], fib, after_fib);
        }

        return uv_run(loop, UV_RUN_DEFAULT);
    }

スレッド関数は分離されたスレッドの中で起動され、 ``uv_work_t`` 構造体が渡されて、関数が終了したら同じ構造体と共に *after* 関数がコールされます。

ブロッキングのライブラリに対するラッパを記述するために、共通の :ref:`パターン <baton>` がデータを交換するためにバトン(baton)を使用します。

libuvのバージョン `0.9.4` からは追加の関数である ``uv_cancel()`` が使用できます。この関数はlibuvのワークキュー上のタスクをキャンセルすることを可能にします。 *まだ開始されていない* タスクだけがキャンセルできます。もしタスクが *既に開始されている、もしくは既に実行し終わった* 場合、 ``uv_cancel()`` は **失敗します。**

.. WARNING::

    ``uv_cancel()`` はUnixでのみ使用可能です!

``uv_cancel()`` はユーザの要求終了の場合に待機中のタスクを後始末するために有用です。例えば、音楽プレーヤは音楽ファイルをスキャンするために複数のディレクトリを探索する必要があります。もしユーザがプログラムを終了した場合、即座に終了し、待機中の要求が実行されるまで待つことはありません。

それでは ``uv_cancel()`` を試すためにフィボナッチの例を改造してみましょう。最初に終了のシグナルハンドラを準備します。

.. rubric:: queue-cancel/main.c
.. literalinclude:: ../code/queue-cancel/main.c
    :linenos:
    :lines: 43-

.. code-block:: c

    int main() {
        loop = uv_default_loop();

        int data[FIB_UNTIL];
        int i;
        for (i = 0; i < FIB_UNTIL; i++) {
            data[i] = i;
            fib_reqs[i].data = (void *) &data[i];
            uv_queue_work(loop, &fib_reqs[i], fib, after_fib);
        }

        uv_signal_t sig;
        uv_signal_init(loop, &sig);
        uv_signal_start(&sig, signal_handler, SIGINT);

        return uv_run(loop, UV_RUN_DEFAULT);
    }

ユーザが ``Ctrl+C`` を入力することによってシグナルを発生させた時、 ``uv_cancel()`` が全てのワーカに送信されます。 ``uv_cancel()``は既に実行されているもしくは終了したものに対しては ``-1`` を返却します。

.. rubric:: queue-cancel/main.c
.. literalinclude:: ../code/queue-cancel/main.c
    :linenos:
    :lines: 33-41
    :emphasize-lines: 6

.. code-block:: c

    void signal_handler(uv_signal_t *req, int signum)
    {
        printf("Signal received!\n");
        int i;
        for (i = 0; i < FIB_UNTIL; i++) {
            uv_cancel((uv_req_t*) &fib_reqs[i]);
        }
        uv_signal_stop(req);
    }

キャンセルが成功したタスクのために、 *終了後*  の関数が ``status`` に ``-1`` を設定して呼び出され、ループのエラーコードは ``UV_ECANCELED`` が設定されます。

.. rubric:: queue-cancel/main.c
.. literalinclude:: ../code/queue-cancel/main.c
    :linenos:
    :lines: 28-31
    :emphasize-lines: 2

.. code-block:: c

    void after_fib(uv_work_t *req, int status) {
        if (status == -1 && uv_last_error(loop).code == UV_ECANCELED)
            fprintf(stderr, "Calculation of %d cancelled.\n", *(int *) req->data);
    }

``uv_cancel()`` は ``uv_fs_t`` と ``uv_getaddrinfo_t`` リクエストと組み合わせて使用することもできます。ファイルシステム系のAPIに対しては、 ``uv_fs_t.errono`` が ``UV_ECANCELED`` に設定されます。

.. TIP::

    よく設計されたプログラムは既に実行されている長時間実行されるワーカを狩猟させるための手段を持ちます。このようなワーカはメインプロセスだけが設定可能な終了を知らせる変数を定期的にチェックすることになります。

.. _inter-thread-communication:

スレッド間通信
--------------------------

時々様々なスレッドで *実行中に* 互いにメッセージを送信したいことがあります。例えば分離されたスレッドで(おそらく ``uv_queue_work`` を用いて)何かの長期間実行するタスクを走らせるが、メインスレッドに経過を通知したいことがあるとします。これは実行中のダウンロードの状態ををユーザに知らせるダウンロードマネージャの例です。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 7-8,34-
    :emphasize-lines: 2,11

.. code-block:: c

    uv_loop_t *loop;
    uv_async_t async;

    int main() {
        loop = uv_default_loop();

        uv_work_t req;
        int size = 10240;
        req.data = (void*) &size;

        uv_async_init(loop, &async, print_progress);
        uv_queue_work(loop, &req, fake_download, after);

        return uv_run(loop, UV_RUN_DEFAULT);
    }

asyncのスレッド通信は *ループ上で* 動作しますがどんなスレッドもメッセージの送信者になることができ、libuvループのスレッド(というよりループ)だけが受信者になることができます。libuvはメッセージを受信した時はいつでもasyncウォッチャと共にコールバック( `print_progress` )を起動します。

.. warning::
    
    メッセージ送信は *async* であることを意識することは重要であり、コールバックは別のスレッドで ``uv_async_send`` が呼び出されると即座(もしくは以後のいつか)に起動されます。libuvは複数の ``uv_async_send`` を結合し、コールバックを一度だけ呼ぶ場合があります。libuvが行う保証は、コールバックは ``uv_async_send`` が呼び出された後、 *最低一回* 呼び出されることだけです。もし待機中の ``uv_async_send`` がない場合、コールバックが呼び出されることはありません。もし複数回の呼び出しを行い、libuvがコールバックをまだ実装する機会を得ていない場合、 複数回の ``uv_sync_send`` に対して *ただ一度* コールバックが呼び出される *可能性が高いです。* コールバックはひとつのイベントに対して二度呼び出されることはありません。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 10-23
    :emphasize-lines: 7-8

.. code-block:: c

    void fake_download(uv_work_t *req) {
        int size = *((int*) req->data);
        int downloaded = 0;
        double percentage;
        while (downloaded < size) {
            percentage = downloaded*100.0/size;
            async.data = (void*) &percentage;
            uv_async_send(&async);

            sleep(1);
            downloaded += (200+random())%1000; // can only download max 1000bytes/sec,
                                               // but at least a 200;
        }
    }

download関数では、進行を表すインジケータを操作し、 `uv_async_send` によりメッセージをキューに積みます。 ``uv_async_send`` はノンブロッキングであり、即座に制御が戻ることを忘れないで下さい。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 30-33

.. code-block:: c

    void print_progress(uv_async_t *handle, int status /*UNUSED*/) {
        double percentage = *((double*) handle->data);
        fprintf(stderr, "Downloaded %.2f%%\n", percentage);
    }

コールバックはlibuvの標準パターンであり、ウォッチャからデータを取り出します。

最終的にウォッチャを後片付けすることを忘れないことが重要です。

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 25-28
    :emphasize-lines: 3

.. code-block:: c

    void after(uv_work_t *req, int status) {
        fprintf(stderr, "Download complete\n");
        uv_close((uv_handle_t*) &async, NULL);
    }

``data`` フィールドの乱用を観察するこの例のあと、 bnoordhuis_ が示したように ``data`` フィールドの使用はスレッドセーフではなく、 ``uv_async_send()`` は実質的にイベントループを起動することだけを意味しています。 アクセスが正しい順序行われることを保証するためにミューテックスもしくはリードライトロックを使用してください。

.. warning::

    ミューテックスとリードライトロックは ``uv_async_send`` がこれを行っているのに対してシグナルハンドラ内部では動作しません。

``uv_async_send`` が必要とされる一つのユースケースは、スレッドアフィニティが必要なライブラリを機能性のために操作するときです。例えばnode.jsでは、v8エンジンのインスタンス、コンテキストとオブジェクト群はv8インスタンスが開始されるスレッドと分けられています。異なるスレッドからのv8データ構造と関わりを持つことは未定義の結果を生み出します。第三者のライブラリと結合しているnode.jsモジュールについて考えてみましょう。そのようなモジュールはおそらくこのようになるでしょう:

1. nodeでは、第三者のライブラリは追加の情報のために呼び出されるJavaScriptコールバックとともに準備されます::

    var lib = require('lib');
    lib.on_progress(function() {
        console.log("Progress");
    });

    lib.do();

    // do other stuff

2. ``lib.do`` はノンブロッキングであるが第三者のライブラリはブロッキングであると思われるため、 ``uv_queue_work`` を使用します。

3. 分離されたスレッドで実行される実際の処理はprogressコールバックを実行したいのですが、JavaScriptでやりとりするv8の中で直接呼び出すことができません。そのため ``uv_async_send`` を使用します。

4. 非同期のコールバックがメインループのスレッド上で実行されます。これはv8スレッドであり、JavaScriptコールバックを起動するためにv8とやりとりを行います。

.. _pthreads: http://man7.org/linux/man-pages/man7/pthreads.7.html

----

.. _node.js is cancer: https://raw.github.com/teddziuba/teddziuba.github.com/master/_posts/2011-10-01-node-js-is-cancer.html
.. _bnoordhuis: https://github.com/bnoordhuis

.. [#] https://github.com/joyent/libuv/blob/master/include/uv.h#L1853
