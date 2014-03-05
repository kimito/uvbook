ネットワーク処理
==========

libuvにおけるネットワーク処理は直接BSDソケットを用いた場合のそう変わりませんが、いくつかのことは簡単になっており、全てはノンブロッキングであり、コンセプトは同じです。加えてlibuvはBSDソケットを用いたソケットの準備、DNSルックアップ、色々なソケットパラメータの調整のようなやっかいで何度も必要となる低レベルの処理を抽象化するためのユーティリティ関数を提供します。

``uv_tcp_t`` と ``uv_udp_t`` 構造体はネットワークI/Oのために用いられます。

TCP
---

TCPはコネクション指向のストリームプロトコルであるため、libuvストリーム基盤の上に構築されています。

サーバ
++++++

サーバソケットの処理は以下のように行われます:

1. ``uv_tcp_init`` でTCPウォッチャを初期化します。
2. それを ``uv_tcp_bind`` します。
3. クライアントにより新しい接続が行われた(established)時に呼び出されるコールバックを設定するためにウォッチャに対して ``uv_listen`` を呼び出します。
4. 接続を受け入れるために ``uv_accept`` を使用します。
5.  クライアントと通信するために :ref:`stream operations <buffers-and-streams>` を使用します。

以下は簡単なエコーサーバです。

.. rubric:: tcp-echo-server/main.c - The listen socket
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 50-
    :emphasize-lines: 4-5,7-9

.. code-block:: c

     int main() {
         loop = uv_default_loop();
 
         uv_tcp_t server;
         uv_tcp_init(loop, &server);
 
         struct sockaddr_in bind_addr = uv_ip4_addr("0.0.0.0", 7000);
         uv_tcp_bind(&server, bind_addr);
         int r = uv_listen((uv_stream_t*) &server, 128, on_new_connection);
         if (r) {
             fprintf(stderr, "Listen error %s\n", uv_err_name(uv_last_error(loop)));
             return 1;
         }
         return uv_run(loop, UV_RUN_DEFAULT);
     }

BSDソケットAPIで必要となる、人間が読めるIPアドレスとポートの組からsockaddr_in構造体への変換に用いられる ``uv_ip4_addr`` というユーティリティ関数があります。この逆の操作が ``uv_ip4_name`` によって得られます。

.. NOTE::

    それほど明確なケースではありませんが、ip4関数と相似の ``uv_ip6_*`` があります。

大部分のセットアップ関数はCPUバウンド(訳注: CPUの処理が大部分を占める)の処理であるため通常の関数となっています。 ``uv_listen`` libuvのコールバックに戻る場所です。第二引数はバックログのキューです -- キューイングされた接続の最大の数です。

接続がクライアントにより初期化された時、クライアントソケットのためのウォッチャを準備し ``uv_accept`` を用いてウォッチャに関連付けるためのコールバックが必要となります。この場合ストリームからデータを読み出す処理を開始します。

.. rubric:: tcp-echo-server/main.c - Accepting the client
.. literalinclude:: ../code/tcp-echo-server/main.c
    :linenos:
    :lines: 34-48
    :emphasize-lines: 9-10

.. code-block:: c

    void on_new_connection(uv_stream_t *server, int status) {
        if (status == -1) {
            // error!
            return;
        }

        uv_tcp_t *client = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
        uv_tcp_init(loop, client);
        if (uv_accept(server, (uv_stream_t*) client) == 0) {
            uv_read_start((uv_stream_t*) client, alloc_buffer, echo_read);
        }
        else {
            uv_close((uv_handle_t*) client, NULL);
        }
    }

残りの関数群はストリームの例とかなりよく似ており、これらのコード中から探しだすことができます。 ただ、ソケットが必要なくなった時に ``uv_close`` を呼び出すことを忘れないでください。これは接続の受付が必要なくなったら ``uv_listen`` コールバックの中で行うこともできます。

クライアント
++++++

bind/listen/acceptを行う箇所では、クライアント側では単純に ``uv_tcp_connect`` を呼び出す処理となります。 ``uv_listen``の同様の ``uv_connect_cb`` スタイルのコールバックが ``uv_listen`` によって使用されます。下記のようになります::


    uv_tcp_t* socket = (uv_tcp_t*)malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, socket);

    uv_connect_t* connect = (uv_connect_t*)malloc(sizeof(uv_connect_t));

    struct sockaddr_in dest = uv_ip4_addr("127.0.0.1", 80);

    uv_tcp_connect(connect, socket, dest, on_connect);

ここで ``on_connect`` は接続が行われた(established)後に呼び出されます。

UDP
---

`User Datagram Protocol`_　はコネクション指向ではない、信頼性のないネットワーク通信を提供します。ですのでlibuvはストリームを提供しない代わりに(受信のために) `uv_udp_t` と(送信のために)　`uv_udp_send_t` 構造体と関連する関数経由でノンブロッキングのUDP機能を提供しています。これは、実際の読み書きのためのAPIは通常のストリームと非常に似ているということです。UDPの使用方法を理解するために `DHCP`_ サーバからIPアドレスを取得する(DHCP探索)の第一歩を例として見てみましょう。 

.. note::

    1024番以下のウェルノウンポートをを使用するため、 `udp-dhcp`は **root** で実行する必要があります。

.. rubric:: udp-dhcp/main.c - Setup and send UDP packets
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 7-10,104-
    :emphasize-lines: 8,10-11,14,15,21

.. code-block:: c

    uv_loop_t *loop;
    uv_udp_t send_socket;
    uv_udp_t recv_socket;

    int main() {
        loop = uv_default_loop();

        uv_udp_init(loop, &recv_socket);
        struct sockaddr_in recv_addr = uv_ip4_addr("0.0.0.0", 68);
        uv_udp_bind(&recv_socket, recv_addr, 0);
        uv_udp_recv_start(&recv_socket, alloc_buffer, on_read);

        uv_udp_init(loop, &send_socket);
        uv_udp_bind(&send_socket, uv_ip4_addr("0.0.0.0", 0), 0);
        uv_udp_set_broadcast(&send_socket, 1);

        uv_udp_send_t send_req;
        uv_buf_t discover_msg = make_discover_msg(&send_req);

        struct sockaddr_in send_addr = uv_ip4_addr("255.255.255.255", 67);
        uv_udp_send(&send_req, &send_socket, &discover_msg, 1, send_addr, on_send);

        return uv_run(loop, UV_RUN_DEFAULT);
    }


.. note::

    ``0.0.0.0`` というIP アドレスは全てのインターフェイスにバインドするために用いられます。 ``255.255.255.255`` というIPアドレスはサブネット上の全てのインターフェイスにパケットを送信するブロードキャストアドレスです。 ``0``というポートはOSがランダムにポートを割り当てることを意味します。

最初に、ポート68番(DHCPクライアント)上で全てのインターフェイスにバインドするための受信ソケットを準備し、読み取り用のウォッチャを設定します。続いて、同様に送信ソケットを準備して、 ``uv_udp_send`` を用いて 67番ポート(DHCPサーバ)に *ブロードキャストメッセージ* を送信します。

ブロードキャストフラグを設定することが **必要不可欠** であり、そうしないと ``EACCES`` エラー [#]_ が発生します。送信した正確なメッセージは本書には無関係ですが、興味があるならコードから読み解くことができます。通常と同じように、readとwriteのコールバックはなにか不具合がある場合は-1のステータスコードを受け取ります。

UDPソケットは特定のピア(相手)と接続されているわけではないので、readのコールバックはパケットの送信者に関する追加パラメータを受け取ります。 ``flags``パラメータはアロケータによって提供されたバッファがデータを保持するのに十分でない場合には ``UV_UDP_PARTIAL`` になる可能性があります。 *この場合、OSは残りのデータを破棄するでしょう。*  (それがUDPです!)

.. rubric:: udp-dhcp/main.c - Reading packets
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 15-27,38-41
    :emphasize-lines: 1,16
    
.. code-block:: c
    
    void on_read(uv_udp_t *req, ssize_t nread, uv_buf_t buf, struct sockaddr *addr, unsigned flags) {
        if (nread == -1) {
            fprintf(stderr, "Read error %s\n", uv_err_name(uv_last_error(loop)));
            uv_close((uv_handle_t*) req, NULL);
            free(buf.base);
            return;
        }

        char sender[17] = { 0 };
        uv_ip4_name((struct sockaddr_in*) addr, sender, 16);
        fprintf(stderr, "Recv from %s\n", sender);

        // ... DHCP specific code

        free(buf.base);
        uv_udp_recv_stop(req);
    }

UDPのオプション
+++++++++++

Time-to-live
~~~~~~~~~~~~

ソケットで送信されるパケットのTTLは ``uv_udp_set_ttl`` を用いて変更することができます。

IPv6限定
~~~~~~~~~~~~~~~

IPv6ソケットはIPv4とIPv6通信の両方に使用することができます。もしIPv6限定にソケットを限定したい場合は、 ``uv_udp_bind6`` [#]_ に ``UV_UDP_IPV6ONLY`` フラグを渡してください。

マルチキャスト
~~~~~~~~~

A socket can (un)subscribe to a multicast group using:

ソケットは以下を用いてマルチキャストグループを購読(解除)することができます:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 796-798
    
.. code-block:: c
    
    UV_EXTERN int uv_udp_set_membership(uv_udp_t* handle,
        const char* multicast_addr, const char* interface_addr,
        uv_membership membership);

ここで、 ``membership``は ``UV_JOIN_GROUP`` か ``UV_LEAVE_GROUP`` です。

マルチキャストのローカルループバックはデフォルト [#]_ で有効になっています。 無効にするには ``uv_udp_set_multicast_loop`` を使用します。

マルチキャストパケットのTTLは ``uv_udp_set_multicast_ttl`` を使用して変更することができます。

Querying DNS
------------

libuv provides asynchronous DNS resolution. For this it provides its own
``getaddrinfo`` replacement [#]_. In the callback you can
perform normal socket operations on the retrieved addresses. Let's connect to
Freenode to see an example of DNS resolution.

libuvは非同期DNS解決を提供します。このため、libuvは自身が使う ``getaddrinfo`` の代替 [#]_ を提供します。コールバックの中で、抽出したアドレスに大して通常のソケット操作を行うことができます。それではDNS解決の例としてFreenode(irc.freenode.net)に接続してみましょう。

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 61-
    :emphasize-lines: 12

.. code-block:: c

    int main() {
        loop = uv_default_loop();

        struct addrinfo hints;
        hints.ai_family = PF_INET;
        hints.ai_socktype = SOCK_STREAM;
        hints.ai_protocol = IPPROTO_TCP;
        hints.ai_flags = 0;

        uv_getaddrinfo_t resolver;
        fprintf(stderr, "irc.freenode.net is... ");
        int r = uv_getaddrinfo(loop, &resolver, on_resolved, "irc.freenode.net", "6667", &hints);

        if (r) {
            fprintf(stderr, "getaddrinfo call error %s\n", uv_err_name(uv_last_error(loop)));
            return 1;
        }
        return uv_run(loop, UV_RUN_DEFAULT);
    }

``uv_getaddrinfo``の戻り値が0でない場合はセットアップに失敗しており、コールバックが呼び出されることはありません。 ``uv_getaddrinfo`` が戻ったあとはすぐに全ての引数を開放することができます。 `hostname` 、 `servname` と `hints` 構造体は `the getaddrinfo man page <getaddrinfo>`_ に文書化されています。

resolverコールバックの中で ``struct addrinfo(s)`` のリンクリストから任意のIPを取り出すことができます。また、 ``uv_tcp_connect`` についても例示しています。コールバックの中で ``uv_freeaddrinfo`` を呼び出すことが必要です。

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 41-59
    :emphasize-lines: 8,16

.. code-block:: c

    void on_resolved(uv_getaddrinfo_t *resolver, int status, struct addrinfo *res) {
        if (status == -1) {
            fprintf(stderr, "getaddrinfo callback error %s\n", uv_err_name(uv_last_error(loop)));
            return;
        }

        char addr[17] = {'\0'};
        uv_ip4_name((struct sockaddr_in*) res->ai_addr, addr, 16);
        fprintf(stderr, "%s\n", addr);

        uv_connect_t *connect_req = (uv_connect_t*) malloc(sizeof(uv_connect_t));
        uv_tcp_t *socket = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
        uv_tcp_init(loop, socket);

        connect_req->data = (void*) socket;
        uv_tcp_connect(connect_req, socket, *(struct sockaddr_in*) res->ai_addr, on_connect);

        uv_freeaddrinfo(res);
    }

ネットワークインターフェイス
------------------

システムのネットワークインターフェイスに関する情報は ``uv_interface_addresses`` を使用してlibuvから得ることができます。できることの概要を把握するために全てのインターフェイスの詳細を印刷するプログラムを見てみましょう。これは何らかのサービスを開始した時にIPアドレスにバインドするために有用です。

.. rubric:: interfaces/main.c
.. literalinclude:: ../code/interfaces/main.c
    :linenos:
    :emphasize-lines: 9,17

.. code-block:: c

    #include <stdio.h>
    #include <uv.h>

    int main() {
        char buf[512];
        uv_interface_address_t *info;
        int count, i;

        uv_interface_addresses(&info, &count);
        i = count;

        printf("Number of interfaces: %d\n", count);
        while (i--) {
            uv_interface_address_t interface = info[i];

            printf("Name: %s\n", interface.name);
            printf("Internal? %s\n", interface.is_internal ? "Yes" : "No");
        
            if (interface.address.address4.sin_family == AF_INET) {
                uv_ip4_name(&interface.address.address4, buf, sizeof(buf));
                printf("IPv4 address: %s\n", buf);
            }
            else if (interface.address.address4.sin_family == AF_INET6) {
                uv_ip6_name(&interface.address.address6, buf, sizeof(buf));
                printf("IPv6 address: %s\n", buf);
            }

            printf("\n");
        }

        uv_free_interface_addresses(info, count);
        return 0;
    }

``is_internal`` はループバックインターフェイスではtrueとなります。もし物理インターフェイスが複数のIPv4/IPv6アドレスを保つ場合、名称は複数回報告され、各アドレスはそれぞれ一回ずつ報告されます。

.. _c-ares: http://c-ares.haxx.se
.. _getaddrinfo: http://www.kernel.org/doc/man-pages/online/pages/man3/getaddrinfo.3.html

.. _User Datagram Protocol: http://en.wikipedia.org/wiki/User_Datagram_Protocol
.. _DHCP: http://tools.ietf.org/html/rfc2131

----

.. [#] http://beej.us/guide/bgnet/output/html/multipage/advanced.html#broadcast
.. [#] on Windows only supported on Windows Vista and later.
.. [#] http://www.tldp.org/HOWTO/Multicast-HOWTO-6.html#ss6.1
.. [#] libuv use the system ``getaddrinfo`` in the libuv threadpool. libuv
    v0.8.0 and earlier also included c-ares_ as an alternative, but this has been
    removed in v0.9.0.
