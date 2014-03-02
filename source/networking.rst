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

The `User Datagram Protocol`_ offers connectionless, unreliable network
communication. Hence libuv doesn't offer a stream. Instead libuv provides
non-blocking UDP support via the `uv_udp_t` (for receiving) and `uv_udp_send_t`
(for sending) structures and related functions.  That said, the actual API for
reading/writing is very similar to normal stream reads. To look at how UDP can
be used, the example shows the first stage of obtaining an IP address from
a `DHCP`_ server -- DHCP Discover.

.. note::

    You will have to run `udp-dhcp` as **root** since it uses well known port
    numbers below 1024.

.. rubric:: udp-dhcp/main.c - Setup and send UDP packets
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 7-10,104-
    :emphasize-lines: 8,10-11,14,15,21

.. note::

    The IP address ``0.0.0.0`` is used to bind to all interfaces. The IP
    address ``255.255.255.255`` is a broadcast address meaning that packets
    will be sent to all interfaces on the subnet.  port ``0`` means that the OS
    randomly assigns a port.

First we setup the receiving socket to bind on all interfaces on port 68 (DHCP
client) and start a read watcher on it. Then we setup a similar send socket and
use ``uv_udp_send`` to send a *broadcast message* on port 67 (DHCP server).

It is **necessary** to set the broadcast flag, otherwise you will get an
``EACCES`` error [#]_. The exact message being sent is irrelevant to this book
and you can study the code if you are interested. As usual the read and write
callbacks will receive a status code of -1 if something went wrong.

Since UDP sockets are not connected to a particular peer, the read callback
receives an extra parameter about the sender of the packet. The ``flags``
parameter may be ``UV_UDP_PARTIAL`` if the buffer provided by your allocator
was not large enough to hold the data. *In this case the OS will discard the
data that could not fit* (That's UDP for you!).

.. rubric:: udp-dhcp/main.c - Reading packets
.. literalinclude:: ../code/udp-dhcp/main.c
    :linenos:
    :lines: 15-27,38-41
    :emphasize-lines: 1,16

UDP Options
+++++++++++

Time-to-live
~~~~~~~~~~~~

The TTL of packets sent on the socket can be changed using ``uv_udp_set_ttl``.

IPv6 stack only
~~~~~~~~~~~~~~~

IPv6 sockets can be used for both IPv4 and IPv6 communication. If you want to
restrict the socket to IPv6 only, pass the ``UV_UDP_IPV6ONLY`` flag to
``uv_udp_bind6`` [#]_.

Multicast
~~~~~~~~~

A socket can (un)subscribe to a multicast group using:

.. literalinclude:: ../libuv/include/uv.h
    :lines: 796-798

where ``membership`` is ``UV_JOIN_GROUP`` or ``UV_LEAVE_GROUP``.

Local loopback of multicast packets is enabled by default [#]_, use
``uv_udp_set_multicast_loop`` to switch it off.

The packet time-to-live for multicast packets can be changed using
``uv_udp_set_multicast_ttl``.

Querying DNS
------------

libuv provides asynchronous DNS resolution. For this it provides its own
``getaddrinfo`` replacement [#]_. In the callback you can
perform normal socket operations on the retrieved addresses. Let's connect to
Freenode to see an example of DNS resolution.

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 61-
    :emphasize-lines: 12

If ``uv_getaddrinfo`` returns non-zero, something went wrong in the setup and
your callback won't be invoked at all. All arguments can be freed immediately
after ``uv_getaddrinfo`` returns. The `hostname`, `servname` and `hints`
structures are documented in `the getaddrinfo man page <getaddrinfo>`_.

In the resolver callback, you can pick any IP from the linked list of ``struct
addrinfo(s)``. This also demonstrates ``uv_tcp_connect``. It is necessary to
call ``uv_freeaddrinfo`` in the callback.

.. rubric:: dns/main.c
.. literalinclude:: ../code/dns/main.c
    :linenos:
    :lines: 41-59
    :emphasize-lines: 8,16

Network interfaces
------------------

Information about the system's network interfaces can be obtained through libuv
using ``uv_interface_addresses``. This simple program just prints out all the
interface details so you get an idea of the fields that are available. This is
useful to allow your service to bind to IP addresses when it starts.

.. rubric:: interfaces/main.c
.. literalinclude:: ../code/interfaces/main.c
    :linenos:
    :emphasize-lines: 9,17

``is_internal`` is true for loopback interfaces. Note that if a physical
interface has multiple IPv4/IPv6 addresses, the name will be reported multiple
times, with each address being reported once.

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
