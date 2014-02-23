イントロダクション
============

この'書籍'はWindowsとUnixで同じAPIを提供するハイパフォーマンスなイベント駆動(evented)I/Oライブラリとしてlibuvを使うためのチュートリアルです。

この書籍はlibuvの主要な部分について言及するよう意図していますが、全ての関数やデータ構造を扱うリファレンスではありません。 `公式のlibuvのドキュメント`_ はlibuvのヘッダファイル自身に含まれています。

.. _公式のlibuvのドキュメント: https://github.com/joyent/libuv/blob/master/include/uv.h

この書籍はまだ執筆中であるため不完全ですが、成長するにつれて読み応えがあるものになるでしょう。

想定読者
--------------------

この書籍の読者は下記のいずれかを想定しています:

1) デーモンかネットワークサービスとクライアントのようなローレベルのプログラムを作成するシステムプログラマ。イベントループのアプローチがあなたのアプリケーションに適していることを発見し、libuvを使おうと思うでしょう。

2) C/C++で書かれたプラットフォームのAPIを、Javascriptに公開された(非)同期のAPI群と共にラップしたいnode.jsモジュールの作者。あなたはlibuvを純粋にnode.jsのコンテキストで使用するでしょう。この用途では、この書籍では言及しないv8/node.jsに特有の部品のための他の資料が必要となるでしょう。

この本はあなたがC言語に慣れていることを仮定しています。

背景
----------

node.js_ プロジェクトはブラウザと分離されたJavascript環境として2009年に開始されました。Googleの V8_ とMarc Lehmannの libev_ を用いることによって、node.jsはイベント駆動のI/Oモデルとブラウザのために形成されたプログラミングスタイルに適した言語とを結びつけました。node.jsが知名度を増すにつれて、Windows上で動作することが重要となりましたが、libevはUnixに依存して動作していました。kqueueか(e)pollのようなイベント通知機構のWindowsで等価なものはIOCPです。libuvはプラットフォームに依存してlibevとIOCPを抽象化するモジュールであり、libevをベースにしたAPIをユーザに提供します。node.jsのv0.9.0でlibuvが採用され、 `libev was removed`_ となりました。

それからlibuvは成熟を続け、システムプログラミングのための高品質な単独のライブラリとなりました。node.js以外のユーザにはMozzilaの Rust_ プログラミング言語と、 `様々な`_ 言語 のバインディングなどが含まれます。

libuvの最初の独立したリリースのバージョンは0.10.2でした。

コード
----

この書籍に含まれる全てのコードはGithub上の書籍のソースの一部として含まれています。サンプルの全てをコンパイルするには、この本を `Clone`_ もしくは `Download`_ し、 ``code/`` フォルダで ``make`` を実行してください。この書籍とコードはlibuvのv0.11.1をもとにしており、このバージョンは ``libuv/`` フォルダに含まれ自動的にコンパイルされるようになっています。

.. _Clone: https://github.com/nikhilm/uvbook
.. _Download: https://github.com/nikhilm/uvbook/downloads
.. _v0.11.1: https://github.com/joyent/libuv/tags
.. _V8: http://code.google.com/p/v8/
.. _libev: http://software.schmorp.de/pkg/libev.html
.. _libuv: https://github.com/joyent/libuv
.. _node.js: http://www.nodejs.org
.. _libev was removed: https://github.com/joyent/libuv/issues/485
.. _Rust: http://rust-lang.org
.. _様々な: https://github.com/joyent/libuv/wiki/Projects-that-use-libuv
