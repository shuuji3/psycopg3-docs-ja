.. currentmodule:: psycopg

..
    .. _pipeline-mode:

    Pipeline mode support
    =====================

.. _pipeline-mode:

パイプライン モードのサポート
=============================

.. versionadded:: 3.1

..
    The *pipeline mode* allows PostgreSQL client applications to send a query
    without having to read the result of the previously sent query. Taking
    advantage of the pipeline mode, a client will wait less for the server, since
    multiple queries/results can be sent/received in a single network roundtrip.
    Pipeline mode can provide a significant performance boost to the application.

*パイプライン モード* を使用すると、PostgreSQL クライアント アプリケーションが前回送信したクエリの結果を読み込む必要なく、クエリを送信できるようになります。パイプライン モードを活用することで、複数のクエリや結果が単一のネットワークラウンドトリップで送受信できるようになるため、クライアントがサーバーを待つ時間が短くなります。パイプライン モードにより、アプリケーションは大幅な性能向上が得られます。

..
    Pipeline mode is most useful when the server is distant, i.e., network latency
    (“ping time”) is high, and also when many small operations are being performed
    in rapid succession. There is usually less benefit in using pipelined commands
    when each query takes many multiples of the client/server round-trip time to
    execute. A 100-statement operation run on a server 300 ms round-trip-time away
    would take 30 seconds in network latency alone without pipelining; with
    pipelining it may spend as little as 0.3 s waiting for results from the
    server.

パイプライン モードが最も役に立つのは、サーバーが遠い場合、つまりネットワーク レイテンシ (「ping 時間」) が高い場合や、多数の小さな操作が立て続けに実行される場合です。各クエリの実行時間がクライアント/サーバー間のラウンドトリップの何倍もかかる場合には、通常、パイプライン コマンドを使っても得られる恩恵は小さくなります。300 ms のラウンドトリップ時間がかかるサーバー上で実行される 100 ステートメントの操作は、パイプラインがないとネットワーク レイテンシだけで 30 秒かかることになるでしょう。パイプラインを使用すると、サーバーからの結果を待つのにわずか 0.3 秒しかかからなくなる可能性があります。

..
    The server executes statements, and returns results, in the order the client
    sends them. The server will begin executing the commands in the pipeline
    immediately, not waiting for the end of the pipeline. Note that results are
    buffered on the server side; the server flushes that buffer when a
    :ref:`synchronization point <pipeline-sync>` is established.

サーバーはステートメントを実行し、クライアントが送信した順序で結果を返します。サーバーはパイプラインの終わりを待たずに、パイプライン内ですぐにコマンドを実行し始めます。結果はサーバー側でバッファリングされ、サーバーは :ref:`同期ポイント <pipeline-sync>` が確立されたときにバッファをフラッシュすることに注意してください。

..
    .. seealso::

        The PostgreSQL documentation about:

        - `pipeline mode`__
        - `extended query message flow`__

        contains many details around when it is most useful to use the pipeline
        mode and about errors management and interaction with transactions.

        .. __: https://www.postgresql.org/docs/current/libpq-pipeline-mode.html
        .. __: https://www.postgresql.org/docs/current/protocol-flow.html#PROTOCOL-FLOW-EXT-QUERY

.. seealso::

    以下の PostgreSQL のドキュメント

    - `パイプライン モード`__
    - `拡張クエリのメッセージ フロー`__

    には、パイプライン モードが最も役に立つ場合と、エラー管理やトランザクションとのやり取りについて、多数の詳細が説明されています。

    .. __: https://www.postgresql.org/docs/current/libpq-pipeline-mode.html
    .. __: https://www.postgresql.org/docs/current/protocol-flow.html#PROTOCOL-FLOW-EXT-QUERY


..
    Client-server messages flow
    ---------------------------

クライアント-サーバー間のメッセージ フロー
------------------------------------------

..
    In order to understand better how the pipeline mode works, we should take a
    closer look at the `PostgreSQL client-server message flow`__.

パイプライン モードの仕組みについてよく理解するためには、`PostgreSQL のクライアント-サーバー間のメッセージ フロー`__ を詳しく見る必要があります。

..
    During normal querying, each statement is transmitted by the client to the
    server as a stream of request messages, terminating with a **Sync** message to
    tell it that it should process the messages sent so far. The server will
    execute the statement and describe the results back as a stream of messages,
    terminating with a **ReadyForQuery**, telling the client that it may now send a
    new query.

通常のクエリの間、各ステートメントはクライアントからサーバーにリクエスト メッセージのストリームとして送られ、サーバーに対してこれまで送ったメッセージを処理するべきことを伝えるために **Sync** メッセージで終端されます。サーバーはステートメントを実行し、クライアントに対して結果をメッセージのストリームとして説明 (describe) して返し、クライアントに対してクライアントが新しいクエリを送っても良いということを伝えるために **ReadyForQuery** で終端します。

..
    For example, the statement (returning no result):

たとえば、次の (何も結果を返さない) ステートメントは、

.. code:: python

    conn.execute("INSERT INTO mytable (data) VALUES (%s)", ["hello"])

..
    results in the following two groups of messages:

結果として、以下の2つのメッセージ グループになります。

..
    .. table::
        :align: left

        +---------------+-----------------------------------------------------------+
        | Direction     | Message                                                   |
        +===============+===========================================================+
        | Python        | - Parse ``INSERT INTO ... (VALUE $1)`` (skipped if        |
        |               |   :ref:`the statement is prepared <prepared-statements>`) |
        | |>|           | - Bind ``'hello'``                                        |
        |               | - Describe                                                |
        | PostgreSQL    | - Execute                                                 |
        |               | - Sync                                                    |
        +---------------+-----------------------------------------------------------+
        | PostgreSQL    | - ParseComplete                                           |
        |               | - BindComplete                                            |
        | |<|           | - NoData                                                  |
        |               | - CommandComplete ``INSERT 0 1``                          |
        | Python        | - ReadyForQuery                                           |
        +---------------+-----------------------------------------------------------+

.. table::
    :align: left

    +---------------+-------------------------------------------------------------------+
    | 方向          | メッセージ                                                        |
    +===============+===================================================================+
    | Python        | - Parse ``INSERT INTO ... (VALUE $1)`` (                          |
    |               |   :ref:`ステートメントが prepare されている <prepared-statements>`|
    |               |   場合はスキップ)                                                 |
    | |>|           | - Bind ``'hello'``                                                |
    |               | - Describe                                                        |
    | PostgreSQL    | - Execute                                                         |
    |               | - Sync                                                            |
    +---------------+-------------------------------------------------------------------+
    | PostgreSQL    | - ParseComplete                                                   |
    |               | - BindComplete                                                    |
    | |<|           | - NoData                                                          |
    |               | - CommandComplete ``INSERT 0 1``                                  |
    | Python        | - ReadyForQuery                                                   |
    +---------------+-------------------------------------------------------------------+

..
    and the query:

そして、次のクエリは、

.. code:: python

    conn.execute("SELECT data FROM mytable WHERE id = %s", [1])

..
    results in the two groups of messages:

結果として、以下の2つのメッセージ グループになります。

..
    .. table::
        :align: left

        +---------------+-----------------------------------------------------------+
        | Direction     | Message                                                   |
        +===============+===========================================================+
        | Python        | - Parse ``SELECT data FROM mytable WHERE id = $1``        |
        |               | - Bind ``1``                                              |
        | |>|           | - Describe                                                |
        |               | - Execute                                                 |
        | PostgreSQL    | - Sync                                                    |
        +---------------+-----------------------------------------------------------+
        | PostgreSQL    | - ParseComplete                                           |
        |               | - BindComplete                                            |
        | |<|           | - RowDescription    ``data``                              |
        |               | - DataRow           ``hello``                             |
        | Python        | - CommandComplete ``SELECT 1``                            |
        |               | - ReadyForQuery                                           |
        +---------------+-----------------------------------------------------------+

.. table::
    :align: left

    +---------------+-----------------------------------------------------------+
    | 方向          | メッセージ                                                |
    +===============+===========================================================+
    | Python        | - Parse ``SELECT data FROM mytable WHERE id = $1``        |
    |               | - Bind ``1``                                              |
    | |>|           | - Describe                                                |
    |               | - Execute                                                 |
    | PostgreSQL    | - Sync                                                    |
    +---------------+-----------------------------------------------------------+
    | PostgreSQL    | - ParseComplete                                           |
    |               | - BindComplete                                            |
    | |<|           | - RowDescription    ``data``                              |
    |               | - DataRow           ``hello``                             |
    | Python        | - CommandComplete ``SELECT 1``                            |
    |               | - ReadyForQuery                                           |
    +---------------+-----------------------------------------------------------+

..
    The two statements, sent consecutively, pay the communication overhead four
    times, once per leg.

2つのステートメントは連続的に送信されるため、1回ごとに通信のオーバーヘッドが4回も課されてしまいます。

..
    The pipeline mode allows the client to combine several operations in longer
    streams of messages to the server, then to receive more than one response in a
    single batch. If we execute the two operations above in a pipeline:

パイプラインモードを使うと、クライアントは複数の操作を、サーバーへのメッセージのより長いストリームとしてまとめられるようになり、1つ以上のレスポンスを1つのバッチにまとまった1つのレスポンスとして受け取れるようになります。上記の2つの操作を次のようにパイプラインで実行した場合、

.. code:: python

    with conn.pipeline():
        conn.execute("INSERT INTO mytable (data) VALUES (%s)", ["hello"])
        conn.execute("SELECT data FROM mytable WHERE id = %s", [1])

..
    they will result in a single roundtrip between the client and the server:

結果として、クライアントとサーバー間で1つのラウンドトリップしかかからなくなるでしょう。

.. table::
    :align: left

    +---------------+-----------------------------------------------------------+
    | 方向          | メッセージ                                                |
    +===============+===========================================================+
    | Python        | - Parse ``INSERT INTO ... (VALUE $1)``                    |
    |               | - Bind ``'hello'``                                        |
    | |>|           | - Describe                                                |
    |               | - Execute                                                 |
    | PostgreSQL    | - Parse ``SELECT data FROM mytable WHERE id = $1``        |
    |               | - Bind ``1``                                              |
    |               | - Describe                                                |
    |               | - Execute                                                 |
    |               | - Sync (1回だけ送る)                                      |
    +---------------+-----------------------------------------------------------+
    | PostgreSQL    | - ParseComplete                                           |
    |               | - BindComplete                                            |
    | |<|           | - NoData                                                  |
    |               | - CommandComplete ``INSERT 0 1``                          |
    | Python        | - ParseComplete                                           |
    |               | - BindComplete                                            |
    |               | - RowDescription    ``data``                              |
    |               | - DataRow           ``hello``                             |
    |               | - CommandComplete ``SELECT 1``                            |
    |               | - ReadyForQuery (1回だけ送る)                             |
    +---------------+-----------------------------------------------------------+

.. |<| unicode:: U+25C0
.. |>| unicode:: U+25B6

.. __: https://www.postgresql.org/docs/current/protocol-flow.html

..
    .. _pipeline-usage:

    Pipeline mode usage
    -------------------

.. _pipeline-usage:

パイプライン モードを使用する
-----------------------------

..
    Psycopg supports the pipeline mode via the `Connection.pipeline()` method. The
    method is a context manager: entering the ``with`` block yields a `Pipeline`
    object. At the end of block, the connection resumes the normal operation mode.

psycopg は `Connection.pipeline()` メソッド経由でパイプライン モードをサポートしています。このメソッドはコンテクスト マネージャです。``with`` ブロックに入ると、`Pipeline` オブジェクトが yield されます。ブロックの終わりで、コネクションは通常の操作モードを継続します。

..
    Within the pipeline block, you can use normally one or more cursors to execute
    several operations, using `Connection.execute()`, `Cursor.execute()` and
    `~Cursor.executemany()`.

パイプライン ブロック内では、1つ以上のカーソルで `Connection.execute()`、`Cursor.execute()`、`~Cursor.executemany()` を普通に使用して、複数の操作を実行できます。

.. code:: python

    >>> with conn.pipeline():
    ...     conn.execute("INSERT INTO mytable VALUES (%s)", ["hello"])
    ...     with conn.cursor() as cur:
    ...         cur.execute("INSERT INTO othertable VALUES (%s)", ["world"])
    ...         cur.executemany(
    ...             "INSERT INTO elsewhere VALUES (%s)",
    ...             [("one",), ("two",), ("four",)])

..
    Unlike in normal mode, Psycopg will not wait for the server to receive the
    result of each query; the client will receive results in batches when the
    server flushes it output buffer. You can receive more than a single result
    by using more than one cursor in the same pipeline.

ノーマル モードとは違い、psycopg はサーバーが各クエリの結果を受信するのを待ちません。クライアントは、サーバーが結果を出力バッファにフラッシュしたときにバッチで受信します。同じパイプライン内の2つ以上のカーソルを使用することで、2つ以上の結果を受信できます。

..
    If any statement encounters an error, the server aborts the current
    transaction and will not execute any subsequent command in the queue until the
    next :ref:`synchronization point <pipeline-sync>`; a `~errors.PipelineAborted`
    exception is raised for each such command. Query processing resumes after the
    synchronization point.

何らかのステートメントでエラーが発生した場合、サーバーは現在のトランザクションを中断し、キューにある後続のコマンドは、次の :ref:`同期ポイント <pipeline-sync>` まで実行しません。そのようなコマンドそれぞれに対して `~errors.PipelineAborted` 例外が発生します。クエリ処理は同期ポイントの後に再開します。

..
    .. warning::

        Certain features are not available in pipeline mode, including:

        - COPY is not supported in pipeline mode by PostgreSQL.
        - `Cursor.stream()` doesn't make sense in pipeline mode (its job is the
          opposite of batching!)
        - `ServerCursor` are currently not implemented in pipeline mode.

.. warning::

    特定の機能はパイプラインでは利用できません。これには次の機能が含まれます。

    - COPY は PostgreSQL のパイプライン モードではサポートされていません。
    - `Cursor.stream()` はパイプライン モードでは意味がありません。(この機能はバッチとは逆です！)
    - `ServerCursor` は現在パイプライン モードでは実装されていません。

..
    .. note::

        Starting from Psycopg 3.1, `~Cursor.executemany()` makes use internally of
        the pipeline mode; as a consequence there is no need to handle a pipeline
        block just to call `!executemany()` once.

.. note::

    psycopg 3.1 以降、`~Cursor.executemany()` 内部でパイプライン モードを使用するようになります。その結果、`!executemany()` を1回だけ呼び出すためにパイプライン ブロックを扱う必要はありません。

..
    .. _pipeline-sync:

    Synchronization points
    ----------------------

.. _pipeline-sync:

同期ポイント
------------

..
    Flushing query results to the client can happen either when a synchronization
    point is established by Psycopg:

クエリ結果のクライアントへのフラッシュは、同期ポイントが psycopg により確立される以下のようなタイミングで行われる可能性があります。

..
    - using the `Pipeline.sync()` method;
    - on `Connection.commit()` or `~Connection.rollback()`;
    - at the end of a `!Pipeline` block;
    - possibly when opening a nested `!Pipeline` block;
    - using a fetch method such as `Cursor.fetchone()` (which only flushes the
      query but doesn't issue a Sync and doesn't reset a pipeline state error).

- `Pipeline.sync()` メソッドの使用時
- `Connection.commit()` または `~Connection.rollback()`
- `!Pipeline` ブロックの最後
- 場合によっては、ネストした `!Pipeline` ブロックのオープン時
- `Cursor.fetchone()` などの fetch メソッドの使用時 (クエリだけをフラッシュしますが、Sync は発行せず、パイプラインのステート エラーもリセットしません)

..
    The server might perform a flush on its own initiative, for instance when the
    output buffer is full.

サーバーは、たとえば出力バッファがフルの場合など、自身の裁量でフラッシュを実行する可能性があります。

..
    Note that, even in :ref:`autocommit <autocommit>`, the server wraps the
    statements sent in pipeline mode in an implicit transaction, which will be
    only committed when the Sync is received. As such, a failure in a group of
    statements will probably invalidate the effect of statements executed after
    the previous Sync, and will propagate to the following Sync.

たとえ :ref:`autocommit <autocommit>` 中であっても、サーバーはパイプライン モードで送信されるステートメントを、暗黙のトランザクション内にラッピングすることに注意してください。これは、Sync を受信したときにしかコミットされません。そのため、ステートメントのグループ内で失敗すると、おそらく前回の Sync より後に実行されたステートメントの効果が無効化され、それが後続の Sync に伝搬します。

..
    For example, in the following block:

たとえば、以下のブロック内では、

.. code:: python

    >>> with psycopg.connect(autocommit=True) as conn:
    ...     with conn.pipeline() as p, conn.cursor() as cur:
    ...         try:
    ...             cur.execute("INSERT INTO mytable (data) VALUES (%s)", ["one"])
    ...             cur.execute("INSERT INTO no_such_table (data) VALUES (%s)", ["two"])
    ...             conn.execute("INSERT INTO mytable (data) VALUES (%s)", ["three"])
    ...             p.sync()
    ...         except psycopg.errors.UndefinedTable:
    ...             pass
    ...         cur.execute("INSERT INTO mytable (data) VALUES (%s)", ["four"])

..
    there will be an error in the block, ``relation "no_such_table" does not
    exist`` caused by the insert ``two``, but probably raised by the `!sync()`
    call. At at the end of the block, the table will contain:

ブロック内で、``two`` の insert が原因の ``relation "no_such_table" does not exist`` というエラーが、しかしおそらく `!sync()` によって発生するでしょう。ブロックの終わりでは、テーブルには以下のデータがあることになります。

.. code:: text

    =# SELECT * FROM mytable;
    +----+------+
    | id | data |
    +----+------+
    |  2 | four |
    +----+------+
    (1 row)

..
    because:

理由は次のとおりです。

..
    - the value 1 of the sequence is consumed by the statement ``one``, but
      the record discarded because of the error in the same implicit transaction;
    - the statement ``three`` is not executed because the pipeline is aborted (so
      it doesn't consume a sequence item);
    - the statement ``four`` is executed with
      success after the Sync has terminated the failed transaction.

- シーケンス値 1 はステートメント ``one`` で消費されるが、レコードは同じ暗黙のトランザクション内でのエラーのため破棄された。
- パイプラインが中断されるため、ステートメント ``three`` は実行されない (したがって、シーケンスのアイテムを消費しない)。
- ステートメント ``four`` は、Sync が失敗したトランザクションを終了させた後、成功裏に実行される。

..
    .. warning::

        The exact Python statement where an exception caused by a server error is
        raised is somewhat arbitrary: it depends on when the server flushes its
        buffered result.

        If you want to make sure that a group of statements is applied atomically
        by the server, do make use of transaction methods such as
        `~Connection.commit()` or `~Connection.transaction()`: these methods will
        also sync the pipeline and raise an exception if there was any error in
        the commands executed so far.

.. warning::

    サーバーエラーによって例外が発生する正確な Python ステートメントは、いくぶん不確定です。サーバーがバッファされた結果をどのタイミングでフラッシュするかによって異なるためです。

    ステートメントのグループがサーバーによってアトミックに適用されることを保証したい場合は、`~Connection.commit()` or `~Connection.transaction()` などのトランザクション メソッドを活用してください。これらのメソッドもパイプラインを同期して、それまでに実行されたコマンドにエラーがあった場合に例外を発生させます。

..
    The fine prints
    ---------------

注意事項
--------

..
    .. warning::

    The Pipeline mode is an experimental feature.

    Its behaviour, especially around error conditions and concurrency, hasn't
    been explored as much as the normal request-response messages pattern, and
    its async nature makes it inherently more complex.

    As we gain more experience and feedback (which is welcome), we might find
    bugs and shortcomings forcing us to change the current interface or
    behaviour.

.. warning::

    パイプライン モードは実験的な機能です。

    パイプライン モードの動作、特に、エラー コンディションや並行性の周辺は、通常のリクエスト-レスポンスメッセージのパターンほど多くは探索されておらず、その非同期の性質により、本質的により複雑になります。

    より経験を積んでフィードバックをもらうにつれて (フィードバックは歓迎です)、バグや欠点を発見し、現在のインターフェイスや動作を変更せざるを得なくなる可能性があります。

..
    The pipeline mode is available on any currently supported PostgreSQL version,
    but, in order to make use of it, the client must use a libpq from PostgreSQL
    14 or higher. You can use `Pipeline.is_supported()` to make sure your client
    has the right library.

パイプライン モードは現在サポートされている PostgreSQL の全てのバージョンで利用可能ですが、利用するには、クライアントが PostgreSQL 14 以上の libpg を使用する必要があります。`Pipeline.is_supported()` を使用すると、クライアントが正しいライブラリを持っていることを確認できます。
