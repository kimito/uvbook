ファイルシステム
==========

単純なファイルシステムの読み取り/書き込みには ``uv_fs_*`` 関数群と ``uv_fs_t`` 構造体を用います。

.. note::

    libuvのファイル操作は :doc:`ソケット操作<networking>` とは異なります。ソケット操作はOSから提供されたノンブロッキング操作を用いています。ファイルシステム操作は内部的にはブロッキング関数を用いていますが、これらの関数はスレッドプールの内部で呼び出され、アプリケーションとのやりとりが必要になるときにイベントループに登録されたウォッチャに通知されます。

全てのファイルシステムは2つの形態を持ちます - *同期的* と *非同期的* です。

*同期的*  な形態はコールバックが指定されない場合に自動的に呼び出され(、そして **ブロック** され)ます。関数の戻り値はUnixの戻り値と同様です(通常は成功時に0、エラー時に-1)。

*非同期*  な形態はコールパックが渡された時に呼び出され、戻り値は0となります。

ファイルを読み書きする
---------------------

ファイルディスクリプタは下記の関数を用いることにより取得されます。

.. code-block:: c

    int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path, int flags, int mode, uv_fs_cb cb)

``flags`` と ``mode`` は標準的な `Unixのflags <http://man7.org/linux/man-pages/man2/open.2.html>`_ です。libuvはWindowsにおけるflagsに変換するように配慮しています。

ファイルディスクリプタは下記の関数を用いることによりクローズされます。

.. code-block:: c

    int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file, uv_fs_cb cb)

ファイルシステム操作のコールバックは下記のシグネチャ(パラメータと戻り値)を持ちます。

.. code-block:: c

    void callback(uv_fs_t* req);

``cat`` の単純な実装を見てみましょう。ファイルがオープンされる時のコールバックを登録することから始めます。

.. rubric:: uvcat/main.c - opening a file
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 39-48
    :emphasize-lines: 2
    
.. code-block:: c

    void on_open(uv_fs_t *req) {
        if (req->result != -1) {
            uv_fs_read(uv_default_loop(), &read_req, req->result,
                       buffer, sizeof(buffer), -1, on_read);
        }
        else {
            fprintf(stderr, "error opening file: %d\n", req->errorno);
        }
        uv_fs_req_cleanup(req);
    }

``uv_fs_t`` 構造体の ``result`` フィールドは ``uv_fs_open`` コールバックの場合にはファイルディスクリプタを表します。もし正常にオープンされた場合、読み取りを開始します。

.. warning::

    ``uv_fs_req_cleanup()`` 関数はlibuv内部のメモリ割り当てを開放するために呼び出す必要があります。

.. rubric:: uvcat/main.c - read callback
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 24-37
    :emphasize-lines: 6,9,12

.. code-block:: c

    void on_read(uv_fs_t *req) {
        uv_fs_req_cleanup(req);
        if (req->result < 0) {
            fprintf(stderr, "Read error: %s\n", uv_strerror(uv_last_error(uv_default_loop())));
        }
        else if (req->result == 0) {
            uv_fs_t close_req;
            // synchronous
            uv_fs_close(uv_default_loop(), &close_req, open_req.result, NULL);
        }
        else {
            uv_fs_write(uv_default_loop(), &write_req, 1, buffer, req->result, -1, on_write);
        }
    }

readの呼び出しの場合、readのコールバックが呼び出される前にデータが格納された *初期化された* バッファを渡さなければなりません。(訳注: on_read中のuv_fs_write()に渡すバッファのことだと思われます)

readコールバックの中の ``result`` フィールドはEOFの場合は0が、エラーの場合は-1が、成功時には読み込んだバイト数が格納されます。

非同期のプログラムを記述するときの共通のパターンを見てみましょう。 ``uv_fs_close()`` の呼び出しは同期的に処理されます。 *通常、一回きりのタスク、もしくは起動時・終了時の処理の一部として処理されるタスクは同期的に処理されます。なぜならプログラムが最も重要なタスクを処理し、複数のI/Oソースを扱うときに高速なI/Oを期待するからです。* 単独のタスクに対してはパフォーマンスの違いは無視できるほどであり、単純なコードのほうがよいと思われるからです。

もとのシステムコールの実際の戻り値は ``uv_fs_t.result`` に格納されるというパターンを見出すことができます。

ファイルシステムへの書き込みは同様に ``uv_fs_write()`` を用いる単純なものです。 *コールバックは書き込みが終了した後に呼び出されます。* このケースでは、コールバックは単純に次の読み取りのきっかけを作るだけです。従って、読み取りと書き込みはコールバックを通じて密接に処理されます。

.. rubric:: uvcat/main.c - write callback
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 14-22
    :emphasize-lines: 7

.. code-block:: c

    void on_write(uv_fs_t *req) {
        uv_fs_req_cleanup(req);
        if (req->result < 0) {
            fprintf(stderr, "Write error: %s\n", uv_strerror(uv_last_error(uv_default_loop())));
        }
        else {
            uv_fs_read(uv_default_loop(), &read_req, open_req.result, buffer, sizeof(buffer), -1, on_read);
        }
    }

.. note::

    通常 `errno<http://man7.org/linux/man-pages/man3/errno.3.html>`_ に格納されるエラーには ``uv_fs_t.errorno`` を通じてアクセスすることができますが、標準的な ``UV_*`` のエラーコードに変換されます。現在は ``errorno`` フィールドからのエラーメッセージ文字列を直接抽出することはできません。

.. warning::
    
    ファイルシステムとディスクが性能のために設定されている方法に従い、 '成功'した書き込みはまだディスクにコミットされていない可能性があります。 これを強く保証するための方法は ``uv_fs_fsync`` を参照してください。

上記の連鎖的な動作の繰り返しを ``main()`` で設定します。

.. rubric:: uvcat/main.c
.. literalinclude:: ../code/uvcat/main.c
    :linenos:
    :lines: 50-54
    :emphasize-lines: 2

.. code-block:: c

    int main(int argc, char **argv) {
        uv_fs_open(uv_default_loop(), &open_req, argv[1], O_RDONLY, 0, on_open);
        uv_run(uv_default_loop(), UV_RUN_DEFAULT);
        return 0;
    }

ファイルシステムの操作
---------------------

``unlink`` 、 ``rmdir`` 、 ``stat`` のような全ての標準的なファイルシステム操作は非同期的な処理のサポートがあり、直感的な引数の順番を持ちます。これらはread/write/openの呼び出しと同様のパターンに従い、 ``uv_fs_result`` フィールドに結果を格納してリターンします。

.. rubric:: Filesystem operations
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1523-1581

.. code-block:: c

    typedef enum {
      UV_FS_UNKNOWN = -1,
      UV_FS_CUSTOM,
      UV_FS_OPEN,
      UV_FS_CLOSE,
      UV_FS_READ,
      UV_FS_WRITE,
      UV_FS_SENDFILE,
      UV_FS_STAT,
      UV_FS_LSTAT,
      UV_FS_FSTAT,
      UV_FS_FTRUNCATE,
      UV_FS_UTIME,
      UV_FS_FUTIME,
      UV_FS_CHMOD,
      UV_FS_FCHMOD,
      UV_FS_FSYNC,
      UV_FS_FDATASYNC,
      UV_FS_UNLINK,
      UV_FS_RMDIR,
      UV_FS_MKDIR,
      UV_FS_RENAME,
      UV_FS_READDIR,
      UV_FS_LINK,
      UV_FS_SYMLINK,
      UV_FS_READLINK,
      UV_FS_CHOWN,
      UV_FS_FCHOWN
    } uv_fs_type;

    /* uv_fs_t is a subclass of uv_req_t */
    struct uv_fs_s {
      UV_REQ_FIELDS
      uv_fs_type fs_type;
      uv_loop_t* loop;
      uv_fs_cb cb;
      ssize_t result;
      void* ptr;
      const char* path;
      uv_err_code errorno;
      uv_stat_t statbuf;  /* Stores the result of uv_fs_stat and uv_fs_fstat. */
      UV_FS_PRIVATE_FIELDS
    };

    UV_EXTERN void uv_fs_req_cleanup(uv_fs_t* req);

    UV_EXTERN int uv_fs_close(uv_loop_t* loop, uv_fs_t* req, uv_file file,
        uv_fs_cb cb);

    UV_EXTERN int uv_fs_open(uv_loop_t* loop, uv_fs_t* req, const char* path,
        int flags, int mode, uv_fs_cb cb);

    UV_EXTERN int uv_fs_read(uv_loop_t* loop, uv_fs_t* req, uv_file file,
        void* buf, size_t length, int64_t offset, uv_fs_cb cb);

    UV_EXTERN int uv_fs_unlink(uv_loop_t* loop, uv_fs_t* req, const char* path,
        uv_fs_cb cb);

コールバックは ``uv_fs_req_cleanup()`` を用いて ``uv_fs_t`` 引数を開放する必要があります。

.. _buffers-and-streams:

バッファとストリーム
-------------------

libuvに含まれる基本的なI/O機能はストリーム( ``uv_stream_t`` )です。TCPやUDPのソケット、ファイルI/OのためのパイプとIPCはストリームのサブクラスとして統一的に扱われます。

ストリームはそれぞれのサブクラスのためのカスタム化された関数を用いて初期化され、下記のの関数により操作されます。

.. code-block:: c

    int uv_read_start(uv_stream_t*, uv_alloc_cb alloc_cb, uv_read_cb read_cb);
    int uv_read_stop(uv_stream_t*);
    int uv_write(uv_write_t* req, uv_stream_t* handle,
                uv_buf_t bufs[], int bufcnt, uv_write_cb cb);

これらのストリームベースの関数はファイルシステムのものより簡単に使え、libuvは 一度 ``uv_read_start()`` がコールされると ``uv_read_stop()`` がコールされるまで自動的にストリームから読み出しを継続します。

データの個々のユニットはバッファ -- ``uv_buf_t`` です。これは単純にバイト列( ``uv_buf_t.base`` )とデータの長さ( ``uv_buf_t.len`` )へのポインタです。 ``uv_buf_t`` は軽量であり値によって引き回されます。管理が必要とされるものは実際のバイト列であり、アプリケーションによって確保・開放される必要があります。

ストリームを実演する前に ``uv_pipe_t`` を使う必要があります。これはローカルのファイル [#]_ のストリーミングを可能にするものです。libuvを用いた単純なteeユーティリティを考えてみましょう。全ての操作を非同期で処理することでイベント駆動I/Oの威力を見ることができます。2つの書き込みは互いにブロックすることはありませんが、実際に書き込まれるまでバッファが開放されないようにバッファデータのコピーに注意する必要があります。

このプログラムは下記のように実行されます。

    ./uvtee <output_file>

必要とするファイルのパイプをオープンすることから始めましょう。ファイルに対するlibuvパイプはデフォルトでは双方向にオープンされます。

.. rubric:: uvtee/main.c - read on pipes
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 62-81
    :emphasize-lines: 4,5,15

.. code-block:: c

    int main(int argc, char **argv) {
        loop = uv_default_loop();

        uv_pipe_init(loop, &stdin_pipe, 0);
        uv_pipe_open(&stdin_pipe, 0);

        uv_pipe_init(loop, &stdout_pipe, 0);
        uv_pipe_open(&stdout_pipe, 1);
    
        uv_fs_t file_req;
        int fd = uv_fs_open(loop, &file_req, argv[1], O_CREAT | O_RDWR, 0644, NULL);
        uv_pipe_init(loop, &file_pipe, 0);
        uv_pipe_open(&file_pipe, fd);

        uv_read_start((uv_stream_t*)&stdin_pipe, alloc_buffer, read_stdin);

        uv_run(loop, UV_RUN_DEFAULT);
        return 0;
    }

``uv_pipe_init()`` の第三引数は名前付きパイプを用いてIPCを行うためには1を設定する必要があります。これには :doc:`processes` で言及しています。``uv_pipe_open()`` の呼び出しはファイルディスクリプタとファイルを関連付けます。

次に ``stdin`` を監視し始めます。 ``alloc_buffer`` コールバックは入力されたデータを保持するために必要な新しいバッファとして呼び出されます。 ``read_stdin`` はこれらのバッファと共に呼び出されます。

.. rubric:: uvtee/main.c - reading buffers
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 19-22,44-60

.. code-block:: c
    
    uv_buf_t alloc_buffer(uv_handle_t *handle, size_t suggested_size) {
        return uv_buf_init((char*) malloc(suggested_size), suggested_size);
    }

    void read_stdin(uv_stream_t *stream, ssize_t nread, uv_buf_t buf) {
        if (nread == -1) {
            if (uv_last_error(loop).code == UV_EOF) {
                uv_close((uv_handle_t*)&stdin_pipe, NULL);
                uv_close((uv_handle_t*)&stdout_pipe, NULL);
                uv_close((uv_handle_t*)&file_pipe, NULL);
            }
        }
        else {
            if (nread > 0) {
                write_data((uv_stream_t*)&stdout_pipe, nread, buf, on_stdout_write);
                write_data((uv_stream_t*)&file_pipe, nread, buf, on_file_write);
            }
        }
        if (buf.base)
            free(buf.base);
    }

ここでは標準的な ``malloc`` で十分ですが、任意のメモリ確保の方法が使えます。例えば、node.jsは自身のV8オブジェクトとバッファを関連付けるスラブアロケータを用いています。

readコールバックの ``nread`` パラメータはエラー時には-1となります。このエラーは、内部の方に基づいたハンドルを処理する汎用のクローズ関数 ``uv_close()`` を用いて、関連する全てのストリームをクローズした場合、エラーはEOFである可能性があります。それ以外の場合、 ``nread`` は非負の数値であり、出力ストリームへのデータ書き込みを試行することができます。最後にバッファの確保と廃棄はアプリケーションの責務であることを忘れないようにして、データを開放します。

.. rubric:: uvtee/main.c - Write to pipe
.. literalinclude:: ../code/uvtee/main.c
    :linenos:
    :lines: 9-13,23-42

.. code-block:: c

    typedef struct {
        uv_write_t req;
        uv_buf_t buf;
    } write_req_t;

    void free_write_req(uv_write_t *req) {
        write_req_t *wr = (write_req_t*) req;
        free(wr->buf.base);
        free(wr);
    }

    void on_stdout_write(uv_write_t *req, int status) {
        free_write_req(req);
    }

    void on_file_write(uv_write_t *req, int status) {
        free_write_req(req);
    }

    void write_data(uv_stream_t *dest, size_t size, uv_buf_t buf, uv_write_cb callback) {
        write_req_t *req = (write_req_t*) malloc(sizeof(write_req_t));
        req->buf = uv_buf_init((char*) malloc(size), size);
        memcpy(req->buf.base, buf.base, size);
        uv_write((uv_write_t*) req, (uv_stream_t*)dest, &req->buf, 1, callback);
    }

``write_data()`` はread関数から獲得したバッファのコピーを作成します。言い換えると、バッファは書き込みの完了によって呼び出されるコールバックにそのまま渡されるわけではありません。これを行うために、書き込みのリクエストとバッファを ``write_req_t`` 内にラップし、コールバックの中でこのラップをほどいています。

.. WARNING::

    もしプログラムが他のプログラムと同時に使用されるのなら、意識しているかどうかに関わらずパイプに書き込みがされます。このことでプログラムは `SIGPIPEを受信することによりアボート`_ されやすくなってしまいます。下記をアプリケーションの初期化時に挿入しておくのがよい考えでしょう。

        signal(SIGPIPE, SIG_IGN)

.. _SIGPIPEを受信することによりアボート: http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#The_special_problem_of_SIGPIPE

ファイルの変更イベント
------------------

現代のOSは個々のファイルもしくはディレクトリを監視し、ファイルが変更されたときにこれを知らせるためのAPIを提供しています。libuvはこのようなファイル変更通知ライブラリ  [#fsnotify]_ をラップしています。この部分はlibuvの中で比較的一貫性のない部分です。ファイルが変更通知システム自体がきわめてプラットフォームごとに多様であるため、全てのことがどこでも動作するというのは困難です。試しに監視したファイルのどれかが変更されたらコマンドを実行する簡単なユーティリティを作成してみます::

    ./onchange <command> <file1> [file2] ...

ファイル更新通知は ``uv_fs_event_init()`` を用いて開始されます:

.. rubric:: onchange/main.c - The setup
.. literalinclude:: ../code/onchange/main.c
    :linenos:
    :lines: 29-32
    :emphasize-lines: 3

.. code-block:: c
    
    while (argc-- > 2) {
        fprintf(stderr, "Adding watch on %s\n", argv[argc]);
        uv_fs_event_init(loop, (uv_fs_event_t*) malloc(sizeof(uv_fs_event_t)), argv[argc], run_command, 0);
    }

第三引数は監視する実際のファイルもしくはディレクトリです。最後の引数である ``flags`` は以下の値のいずれかになります:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 1728,1737,1744

.. code-block:: c
   
    UV_FS_EVENT_WATCH_ENTRY = 1,
    UV_FS_EVENT_STAT = 2,
    UV_FS_EVENT_RECURSIVE = 3

``UV_FS_EVENT_WATCH_ENTRY`` と ``UV_FS_EVENT_STAT`` は(まだ)何もしません。
``UV_FS_EVENT_RECURSIVE`` はサポートされているプラットフォームではサブディレクトリの監視を開始します。

このコールバックは下記の引数を受け取ります:

  #. ``uv_fs_event_t *handle`` - ウォッチャ。 ウォッチャの ``filename`` フィールドは監視されるファイルです。
  #. ``const char *filename`` - もしディレクトリが監視されていた場合、ここは変更されたファイルになります。LinuxとWindowsの場合のみ ``null``以外の値が入ります。他のプラットフォームでは ``null`` になるかも知れません。
  #. ``int flags`` - ``UV_RENAME`` か ``UV_CHANGE``のいずれかです。
  #. ``int status`` - 現在は常に0です。

この例では単純に引数をprintして ``system()`` を利用してコマンドを実行しています。

.. rubric:: onchange/main.c - file change notification callback
.. literalinclude:: ../code/onchange/main.c
    :linenos:
    :lines: 9-18

.. code-block:: c
    
        void run_command(uv_fs_event_t *handle, const char *filename, int events, int status) {
        fprintf(stderr, "Change detected in %s: ", handle->filename);
        if (events == UV_RENAME)
            fprintf(stderr, "renamed");
        if (events == UV_CHANGE)
            fprintf(stderr, "changed");

        fprintf(stderr, " %s\n", filename ? filename : "");
        system(command);
    }

----

.. [#fsnotify] Linuxではinotify、 DarwinではFSEvents、BSD系ではkqueue、WindowsではReadDirectoryChangesW、 Solarisではevent ports、Cygwinではサポートされていません。

.. [#] :ref:`pipes` を参照のこと
