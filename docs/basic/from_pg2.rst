.. index::
    pair: psycopg2; Differences

.. currentmodule:: psycopg

..
    .. _from-psycopg2:

    Differences from `!psycopg2`
    ============================

.. _from-psycopg2:

`!psycopg2` との違い
====================

..
    Psycopg 3 uses the common DBAPI structure of many other database adapters and
    tries to behave as close as possible to `!psycopg2`. There are however a few
    differences to be aware of.

Psycopg 3 は、他の多くのデータベースアダプターと共通の DBAPI 構造を使用しており、できるだけ `!psycopg2` に近い動作になるように努めています。しかし、注意するべき異なる点がいくつかあります。

..
    .. tip::
        Most of the times, the workarounds suggested here will work with both
        Psycopg 2 and 3, which could be useful if you are porting a program or
        writing a program that should work with both Psycopg 2 and 3.

.. tip::
    ほとんどの場合、ここで提案しているワークアラウンドは Psycopg 2 と 3 の両方で動作します。これは Psycopg 2 と 3 の両方で動作する必要があるプログラムをポーティング・作成している場合に役に立つかもしれません。

..
    .. _server-side-binding:

    Server-side binding
    -------------------

.. _server-side-binding:

サーバーサイド バインディング
------------------------------

..
    Psycopg 3 sends the query and the parameters to the server separately, instead
    of merging them on the client side. Server-side binding works for normal
    :sql:`SELECT` and data manipulation statements (:sql:`INSERT`, :sql:`UPDATE`,
    :sql:`DELETE`), but it doesn't work with many other statements. For instance,
    it doesn't work with :sql:`SET` or with :sql:`NOTIFY`::

Psycopg 3 は、クエリとパラメータをクライアントサイドでマージするのではなく、サーバーに別々に送信します。サーバーサイド バインディングは 通常の :sql:`SELECT` とデータ操作のステートメント (:sql:`INSERT`、:sql:`UPDATE`、:sql:`DELETE`) に対して動作しますが、他の多くのステートメントでは動作しません。たとえば、次のように :sql:`SET` や :sql:`NOTIFY` では動作しません。

.. code:: python

    >>> conn.execute("SET TimeZone TO %s", ["UTC"])
    Traceback (most recent call last):
    ...
    psycopg.errors.SyntaxError: syntax error at or near "$1"
    LINE 1: SET TimeZone TO $1
                            ^

    >>> conn.execute("NOTIFY %s, %s", ["chan", 42])
    Traceback (most recent call last):
    ...
    psycopg.errors.SyntaxError: syntax error at or near "$1"
    LINE 1: NOTIFY $1, $2
                   ^

..
    and with any data definition statement::

そして、次のようなデータ定義のステートメントでも動作しません。

.. code:: python

    >>> conn.execute("CREATE TABLE foo (id int DEFAULT %s)", [42])
    Traceback (most recent call last):
    ...
    psycopg.errors.UndefinedParameter: there is no parameter $1
    LINE 1: CREATE TABLE foo (id int DEFAULT $1)
                                             ^

..
    Sometimes, PostgreSQL offers an alternative: for instance the `set_config()`__
    function can be used instead of the :sql:`SET` statement, the `pg_notify()`__
    function can be used instead of :sql:`NOTIFY`::

ときには、PostgreSQL が代替手段を提供してくれることもあります。たとえば、:sql:`SET` ステートメントの代わりに `set_config()`__ 関数が、:sql:`NOTIFY` の代わりに `pg_notify()`__ 関数がそれぞれ利用できます。

.. code:: python

    >>> conn.execute("SELECT set_config('TimeZone', %s, false)", ["UTC"])

    >>> conn.execute("SELECT pg_notify(%s, %s)", ["chan", "42"])

.. __: https://www.postgresql.org/docs/current/functions-admin.html
    #FUNCTIONS-ADMIN-SET

.. __: https://www.postgresql.org/docs/current/sql-notify.html
    #id-1.9.3.157.7.5

..
    If this is not possible, you must merge the query and the parameter on the
    client side. You can do so using the `psycopg.sql` objects::

これが不可能な場合は、クエリとパラメータをクライアントサイドでマージしなければなりません。次のように `psycopg.sql` オブジェクトを使うとマージが行なえます。

.. code:: python

    >>> from psycopg import sql

    >>> cur.execute(sql.SQL("CREATE TABLE foo (id int DEFAULT {})").format(42))

あるいは、次のように、`ClientCursor` などの :ref:`クライアントサイドのバインディング カーソル <client-side-binding-cursors>` を作成します。

.. code:: python

    >>> cur = ClientCursor(conn)
    >>> cur.execute("CREATE TABLE foo (id int DEFAULT %s)", [42])

..
    If you need `!ClientCursor` often, you can set the `Connection.cursor_factory`
    to have them created by default by `Connection.cursor()`. This way, Psycopg 3
    will behave largely the same way of Psycopg 2.

`!ClientCursor` が頻繁に必要になる場合、`Connection.cursor_factory` を設定すると、`Connection.cursor()` がデフォルトで作成するように設定できます。このようにすると、Psycopg 3 は大部分で Psycopg 2 と同じように動作するようになります。

..
    Note that, both server-side and client-side, you can only specify **values**
    as parameters (i.e. *the strings that go in single quotes*). If you need to
    parametrize different parts of a statement (such as a table name), you must
    use the `psycopg.sql` module::

サーバーサイドとクライアントサイドのいずれでも、パラメータとして指定できるのは特定の **値** (つまり、*シングルクォート内に入る文字列*) だけであることに注意してください。ステートメントの異なるパーツ (テーブル名など) をパラメータ化する必要がある場合には、次のように `psycopg.sql` モジュールを使う必要があります。

..
        >>> from psycopg import sql

        # This will quote the user and the password using the right quotes
        # e.g.: ALTER USER "foo" SET PASSWORD 'bar'
        >>> conn.execute(
        ...     sql.SQL("ALTER USER {} SET PASSWORD {}")
        ...     .format(sql.Identifier(username), password))

.. code:: python

    >>> from psycopg import sql

    # これはユーザーとパスワードを正しいクォートを使ってクォートします
    # 例: ALTER USER "foo" SET PASSWORD 'bar'
    >>> conn.execute(
    ...     sql.SQL("ALTER USER {} SET PASSWORD {}")
    ...     .format(sql.Identifier(username), password))

..
    .. _multi-statements:

    Multiple statements in the same query
    -------------------------------------

.. _multi-statements:

同じクエリ内の複数のステートメント
-------------------------------------

..
    As a consequence of using :ref:`server-side bindings <server-side-binding>`,
    when parameters are used, it is not possible to execute several statements in
    the same `!execute()` call, separating them by semicolon::

:ref:`サーバーサイド バインディング <server-side-binding>` の利用結果として、パラメータが使用されている場合、同じ `!execute()` 呼び出しで、カンマで区切られた複数のステートメントを実行することは不可能になります。

    >>> conn.execute(
    ...     "INSERT INTO foo VALUES (%s); INSERT INTO foo VALUES (%s)",
    ...     (10, 20))
    Traceback (most recent call last):
    ...
    psycopg.errors.SyntaxError: cannot insert multiple commands into a prepared statement

..
    One obvious way to work around the problem is to use several `!execute()`
    calls.

問題を回避する明らかな方法の1つは、複数の `!execute()` 呼び出しを行うことです。

..
    **There is no such limitation if no parameters are used**. As a consequence, you
    can compose a multiple query on the client side and run them all in the same
    `!execute()` call, using the `psycopg.sql` objects::

**パラメータが使用されていない場合には、そのような制限はありません**。結果として、次のように `psycopg.sql` オブジェクトを使用することで、複数のクエリをクライアントサイドで構築して、同じ `!execute()` 呼び出し内で実行できます。

    >>> from psycopg import sql
    >>> conn.execute(
    ...     sql.SQL("INSERT INTO foo VALUES ({}); INSERT INTO foo values ({})"
    ...     .format(10, 20))

または、次のように :ref:`クライアントサイド バインディング カーソル <client-side-binding-cursors>` を使用します。

    >>> cur = psycopg.ClientCursor(conn)
    >>> cur.execute(
    ...     "INSERT INTO foo VALUES (%s); INSERT INTO foo VALUES (%s)",
    ...     (10, 20))

..
    .. warning::

        If a statement must be executed outside a transaction (such as
        :sql:`CREATE DATABASE`), it cannot be executed in batch with other
        statements, even if the connection is in autocommit mode::

            >>> conn.autocommit = True
            >>> conn.execute("CREATE DATABASE foo; SELECT 1")
            Traceback (most recent call last):
            ...
            psycopg.errors.ActiveSqlTransaction: CREATE DATABASE cannot run inside a transaction block

        This happens because PostgreSQL itself will wrap multiple statements in a
        transaction. Note that you will experience a different behaviour in
        :program:`psql` (:program:`psql` will split the queries on semicolons and
        send them to the server separately).

        This is not new in Psycopg 3: the same limitation is present in
        `!psycopg2` too.

.. warning::

    ステートメントをトランザクションの外部で実行するする必要がある場合 (:sql:`CREATE DATABASE` など)、次のように、たとえコネクションが autocommit モードだとしても、他のステートメントとともにバッチで実行することはできません。

        >>> conn.autocommit = True
        >>> conn.execute("CREATE DATABASE foo; SELECT 1")
        Traceback (most recent call last):
        ...
        psycopg.errors.ActiveSqlTransaction: CREATE DATABASE cannot run inside a transaction block

    この問題が起きるのは、PostgreSQL 自体が複数のステートメントをトランザクション内にラッピングするためです。:program:`psql` では異なる動作を経験することに注意してください (:program:`psql` はクエリをセミコロンで分割して、これらを別々にサーバーに送信します)。

    これは Psycopg 3 の新たな変更というわけではありません。`!psycopg2` にも同じ制限がありました。

..
    .. _multi-results:

    Multiple results returned from multiple statements
    --------------------------------------------------

.. _multi-results:

複数のステートメントから返される複数の結果
--------------------------------------------------

..
    If more than one statement returning results is executed in psycopg2, only the
    result of the last statement is returned::

結果を返す2つ以上のステートメントが psycopg2 で実行された場合、次のように最後のステートメントの結果だけが返されます。

    >>> cur_pg2.execute("SELECT 1; SELECT 2")
    >>> cur_pg2.fetchone()
    (2,)

..
    Psycopg 3 instead, all the results are available. After running the query,
    the first result will be readily available in the cursor and can be consumed
    using the usual `!fetch*()` methods. In order to access the following
    results, you can use the `Cursor.nextset()` method::

代わりに psycopg 3 では、すべての結果が利用できます。クエリを実行した後、最初の結果はカーソル上ですぐに利用可能になり、通常の `!fetch*()` メソッドを使用して取得できます。後続の結果にアクセスするためには、次のように `Cursor.nextset()` メソッドが使用できます。

    >>> cur_pg3.execute("SELECT 1; SELECT 2")
    >>> cur_pg3.fetchone()
    (1,)

    >>> cur_pg3.nextset()
    True
    >>> cur_pg3.fetchone()
    (2,)

    >>> cur_pg3.nextset()
    None  # no more results

..
    Remember though that you cannot use server-side bindings to :ref:`execute more
    than one statement in the same query <multi-statements>`, if you are passing
    parameters to the query.

ただし、パラメータをクエリに渡している場合、サーバーサイド バインディングを使って :ref:`2つ以上のステートメントを同じクエリ内で実行する <multi-statements>` ことはできないことに注意してください。

..
    .. _difference-cast-rules:

    Different cast rules
    --------------------

.. _difference-cast-rules:

異なるキャスト ルール
----------------------

..
    In rare cases, especially around variadic functions, PostgreSQL might fail to
    find a function candidate for the given data types::

稀な状況では、特に可変引数関数の辺りでは、次のように、PostgreSQL が与えられたデータ型に対する関数の候補を見つけるのに失敗する可能性があります

    >>> conn.execute("SELECT json_build_array(%s, %s)", ["foo", "bar"])
    Traceback (most recent call last):
    ...
    psycopg.errors.IndeterminateDatatype: could not determine data type of parameter $1

..
    This can be worked around specifying the argument types explicitly via a cast::

これは、次のようにキャストを介して引数の型を明示的に指定することで回避できます。

    >>> conn.execute("SELECT json_build_array(%s::text, %s::text)", ["foo", "bar"])

..
    .. _in-and-tuple:

    You cannot use ``IN %s`` with a tuple
    -------------------------------------

.. _in-and-tuple:

``IN %s`` はタプルとともに使えない
-------------------------------------

``IN`` は単一のパラメータとしてタプルとともに使うことはできません。これは ``psygopg2`` では可能でした。

.. code::

    >>> conn.execute("SELECT * FROM foo WHERE id IN %s", [(10,20,30)])
    Traceback (most recent call last):
    ...
    psycopg.errors.SyntaxError: syntax error at or near "$1"
    LINE 1: SELECT * FROM foo WHERE id IN $1
                                          ^
..
    What you can do is to use the `= ANY()`__ construct and pass the candidate
    values as a list instead of a tuple, which will be adapted to a PostgreSQL
    array::

できることは、`= ANY()`__ construct を使用して、次のように候補値をタプルではなくリストとして渡すことです。これは PostgreSQL の配列に適応されます。

.. code:: python

    >>> conn.execute("SELECT * FROM foo WHERE id = ANY(%s)", [[10,20,30]])

..
    Note that `ANY()` can be used with `!psycopg2` too, and has the advantage of
    accepting an empty list of values too as argument, which is not supported by
    the :sql:`IN` operator instead.

`ANY()` は `!psycopg2` でも使用でき、空の値のリストも引数として受け取れる利点があることに注意してください。:sql:`IN` 空のリストは演算子ではサポートされていません。

.. __: https://www.postgresql.org/docs/current/functions-comparisons.html
    #id-1.5.8.30.16

..
    .. _is-null:

    You cannot use ``IS %s``
    ------------------------

.. _is-null:

``IS %s`` は使用できない
------------------------

..
    You cannot use :sql:`IS %s` or :sql:`IS NOT %s`::

次のように、:sql:`IS %s` や :sql:`IS NOT %s` は使用できません。

.. code:: python

    >>> conn.execute("SELECT * FROM foo WHERE field IS %s", [None])
    Traceback (most recent call last):
    ...
    psycopg.errors.SyntaxError: syntax error at or near "$1"
    LINE 1: SELECT * FROM foo WHERE field IS $1
                                         ^

..
    This is probably caused by the fact that :sql:`IS` is not a binary operator in
    PostgreSQL; rather, :sql:`IS NULL` and :sql:`IS NOT NULL` are unary operators
    and you cannot use :sql:`IS` with anything else on the right hand side.
    Testing in psql:

これはおそらく、:sql:`IS` が PostgreSQL では二項演算子ではないという事実に起因します。むしろ、:sql:`IS NULL` と :sql:`IS NOT NULL` は単項演算子であり、:sql:`IS` を右側にある他の何かとともに使用することはできません。psql でテストしてみると、次のようになります。

.. code:: text

    =# SELECT 10 IS 10;
    ERROR:  syntax error at or near "10"
    LINE 1: select 10 is 10;
                         ^

..
    What you can do instead is to use the `IS DISTINCT FROM operator`__, which
    will gladly accept a placeholder::

代わりにできることは、`IS DISTINCT FROM 演算子`__ を使用することです。この演算子は、次のようにプレースホルダーを喜んで受け入れます。

.. code:: python

    >>> conn.execute("SELECT * FROM foo WHERE field IS NOT DISTINCT FROM %s", [None])

.. __: https://www.postgresql.org/docs/current/functions-comparison.html

..
    Analogously you can use :sql:`IS DISTINCT FROM %s` as a parametric version of
    :sql:`IS NOT %s`.

同様に、:sql:`IS DISTINCT FROM %s` はパラメータ バージョンの :sql:`IS NOT %s` として使用できます。

..
    .. _diff-cursors:

    Cursors subclasses
    ------------------

.. _diff-cursors:

カーソル サブクラス
-------------------

..
    In `!psycopg2`, a few cursor subclasses allowed to return data in different
    form than tuples. In Psycopg 3 the same can be achieved by setting a :ref:`row
    factory <row-factories>`:

`!psycopg2` では、少数のカーソルのサブクラスが、タプル以外の形式でデータを返すことができました。psycopg 3 では、同じことが次のように :ref:`行ファクトリ <row-factories>` を設定することで可能になります。

..
    - instead of `~psycopg2.extras.RealDictCursor` you can use
      `~psycopg.rows.dict_row`;

- `~psycopg2.extras.RealDictCursor` の代わりに `~psycopg.rows.dict_row` が使えます。

..
    - instead of `~psycopg2.extras.NamedTupleCursor` you can use
      `~psycopg.rows.namedtuple_row`.

- `~psycopg2.extras.NamedTupleCursor` の代わりに `~psycopg.rows.namedtuple_row` が使えます。

..
    Other row factories are available in the `psycopg.rows` module. There isn't an
    object behaving like `~psycopg2.extras.DictCursor` (whose results are
    indexable both by column position and by column name).

他の利用可能な行ファクトリは、`psycopg.rows` モジュールにあります。`~psycopg2.extras.DictCursor` (その結果が列の位置と列の名前の両方でインデックス可能なもの) のように動作するオブジェクトはありません。

..
    .. code::

        from psycopg.rows import dict_row, namedtuple_row

        # By default, every cursor will return dicts.
        conn = psycopg.connect(DSN, row_factory=dict_row)

        # You can set a row factory on a single cursor too.
        cur = conn.cursor(row_factory=namedtuple_row)

.. code::

    from psycopg.rows import dict_row, namedtuple_row

    # デフォルトでは、すべてのカーソルがディクショナリを返す。
    conn = psycopg.connect(DSN, row_factory=dict_row)

    # 行ファクトリには単一のカーソルも設定できる。
    cur = conn.cursor(row_factory=namedtuple_row)

..
    .. _diff-adapt:

    Different adaptation system
    ---------------------------

.. _diff-adapt:

異なる適応システム
-------------------

..
    The adaptation system has been completely rewritten, in order to address
    server-side parameters adaptation, but also to consider performance,
    flexibility, ease of customization.

適応システムは完全に書き換えられました。これは、サーバーサイドのパラメータの適応に対応するためですが、性能、柔軟性、カスタマイズを簡単にすることも考慮しています。

..
    The default behaviour with builtin data should be :ref:`what you would expect
    <types-adaptation>`. If you have customised the way to adapt data, or if you
    are managing your own extension types, you should look at the :ref:`new
    adaptation system <adaptation>`.

ビルトインのデータを使用したデフォルトの動作は、:ref:`あなたが期待するとおり <types-adaptation>` のはずです。データの適応方法をカスタマイズしていた場合、または、独自の拡張型を管理している場合、:ref:`新しい適応システム <adaptation>` に目を通す必要があります。

..
    .. seealso::

        - :ref:`types-adaptation` for the basic behaviour.
        - :ref:`adaptation` for more advanced use.

.. seealso::

    - 基本的な動作については :ref:`types-adaptation` を参照してください。
    - より発展的な使い方については :ref:`adaptation` を参照してください。

..
    .. _diff-copy:

    Copy is no longer file-based
    ----------------------------

.. _diff-copy:

copy はファイルベースではなくなった
-----------------------------------

..
    `!psycopg2` exposes :ref:`a few copy methods <pg2:copy>` to interact with
    PostgreSQL :sql:`COPY`. Their file-based interface doesn't make it easy to load
    dynamically-generated data into a database.

`!psycopg2` は PostgreSQL の :sql:`COPY` とやり取りするために :ref:`いくつかの copy メソッド <pg2:copy>` を公開していました。それらのファイルベースのインターフェイスでは、動的に生成されたデータをデータベースに読み込むのが簡単ではありません。

..
    There is now a single `~Cursor.copy()` method, which is similar to
    `!psycopg2` `!copy_expert()` in accepting a free-form :sql:`COPY` command and
    returns an object to read/write data, block-wise or record-wise. The different
    usage pattern also enables :sql:`COPY` to be used in async interactions.

現在は、1つの `~Cursor.copy()` メソッドだけがあります。このメソッドは、自由形式の :sql:`COPY` コマンドを受け取り、ブロック単位またはレコード単位のデータの読み込み/書き込みのためのオブジェクトを返すという点で `!psycopg2` の `!copy_expert()` に似ています。異なる使用パターンにより、:sql:`COPY` を非同期の対話でも使用することもできます。

..
    .. seealso:: See :ref:`copy` for the details.

.. seealso:: 詳細については :ref:`copy` も参照してください。

.. _diff-with:

`!with` connection
------------------

..
    In `!psycopg2`, using the syntax :ref:`with connection <pg2:with>`,
    only the transaction is closed, not the connection. This behaviour is
    surprising for people used to several other Python classes wrapping resources,
    such as files.

`!psycopg2` では、:ref:`with connection <pg2:with>` という構文を使うと、トランザクションだけがクローズされ、コネクションはクローズされませんでした。この動作は、他のさまざまなリソースをラッピングするファイルなどの Python クラスに慣れている人たちを驚かせました。

..
    In Psycopg 3, using :ref:`with connection <with-connection>` will close the
    connection at the end of the `!with` block, making handling the connection
    resources more familiar.

psycopg 3 では、:ref:`with connection <with-connection>` を使用すると `!with` ブロックの最後でコネクションをクローズするようになり、コネクションのリソースの処理をより慣れ親しんだものにしました。

..
    In order to manage transactions as blocks you can use the
    `Connection.transaction()` method, which allows for finer control, for
    instance to use nested transactions.

トランザクションをブロックとして管理するためには、`Connection.transaction()` メソッドが使えます。このメソッドを使うと、たとえばネストされたトランザクションの使用など、細かい制御が可能になります。

..
    .. seealso:: See :ref:`transaction-context` for details.

.. seealso:: 詳細については、:ref:`transaction-context` を参照してください。

..
    .. _diff-callproc:

    `!callproc()` is gone
    ---------------------

.. _diff-callproc:

`!callproc()` はなくなった
------------------------------

..
    `cursor.callproc()` is not implemented. The method has a simplistic semantic
    which doesn't account for PostgreSQL positional parameters, procedures,
    set-returning functions... Use a normal `~Cursor.execute()` with :sql:`SELECT
    function_name(...)` or :sql:`CALL procedure_name(...)` instead.

`cursor.callproc()` は実装されていません。このメソッドは単純なセマンティクスしか持たず、PostgreSQL の位置引数、プロシージャ、set-returning 関数などを考慮に入れません。代わりに、通常の `~Cursor.execute()` で :sql:`SELECT function_name(...)` または :sql:`CALL procedure_name(...)` を使ってください。

..
    .. _diff-client-encoding:

    `!client_encoding` is gone
    --------------------------

.. _diff-client-encoding:

`!client_encoding` はなくなった
-----------------------------------

..
    Psycopg automatically uses the database client encoding to decode data to
    Unicode strings. Use `ConnectionInfo.encoding` if you need to read the
    encoding. You can select an encoding at connection time using the
    `!client_encoding` connection parameter and you can change the encoding of a
    connection by running a :sql:`SET client_encoding` statement... But why would
    you?

psycopg は、データをUnicode 文字列にデコードするために、自動的にデータベース クライアントのエンコーディングを使います。エンコーディングを読み取る必要がある場合は、`ConnectionInfo.encoding` を使用してください。コネクション時のエンコーディングを選択するには `!client_encoding` コネクション パラメータが使用でき、コネクションのエンコーディングは :sql:`SET client_encoding` ステートメントを実行すれば変換することはできます。でも、なぜそのようなことをするのでしょうか？

..
    .. _transaction-characteristics-and-autocommit:

    Transaction characteristics attributes don't affect autocommit sessions
    -----------------------------------------------------------------------

.. _transaction-characteristics-and-autocommit:

トランザクションの性質の属性は autocommit セッションに影響を及ぼさない
-----------------------------------------------------------------------

..
    :ref:`Transactions characteristics attributes <transaction-characteristics>`
    such as `~Connection.read_only` don't affect automatically autocommit
    sessions: they only affect the implicit transactions started by non-autocommit
    sessions and the transactions created by the `~Connection.transaction()`
    block (for both autocommit and non-autocommit connections).

`~Connection.read_only` などの :ref:`トランザクションの性質の属性 <transaction-characteristics>` は自動的な autocommit セッションに影響を及ぼさなくなりました。これらの属性は非 autocommit なセッションにより開始された暗黙のトランザクションと、`~Connection.transaction()` ブロックにより作成されたトランザクション (autocommit および非 autocommit コネクションの両方) にのみ影響します。

..
    If you want to put an autocommit transaction in read-only mode, please use the
    default_transaction_read_only__ GUC, for instance executing the statement
    :sql:`SET default_transaction_read_only TO true`.

autocommit なトランザクションを read-only モードにしたい場合、default_transaction_read_only__ GUC を使用してください。たとえば、ステートメント :sql:`SET default_transaction_read_only TO true` を実行します。

.. __: https://www.postgresql.org/docs/current/runtime-config-client.html
       #GUC-DEFAULT-TRANSACTION-READ-ONLY

..
    .. _infinity-datetime:

    No default infinity dates handling
    ----------------------------------

.. _infinity-datetime:

デフォルトの infinity な日付の処理はない
----------------------------------------

..
    PostgreSQL can represent a much wider range of dates and timestamps than
    Python. While Python dates are limited to the years between 1 and 9999
    (represented by constants such as `datetime.date.min` and
    `~datetime.date.max`), PostgreSQL dates extend to BC dates and past the year
    10K. Furthermore PostgreSQL can also represent symbolic dates "infinity", in
    both directions.

PostgreSQL は、Python よりずっと広い範囲の日付とタイムスタンプを表現できます。Python の日付は 1 年から 9999 年に制限されますが (`datetime.date.min` と `~datetime.date.max` などの定数で表現される)、PostgreSQL の日付は BC の日付から 10,000 年を超えた日付まで拡張します。さらに、PostgreSQL では両方向でシンボルの日付 "infinity" を表現することもできます。

..
    In psycopg2, by default, `infinity dates and timestamps map to 'date.max'`__
    and similar constants. This has the problem of creating a non-bijective
    mapping (two Postgres dates, infinity and 9999-12-31, both map to the same
    Python date). There is also the perversity that valid Postgres dates, greater
    than Python `!date.max` but arguably lesser than infinity, will still
    overflow.

psycopg2 では、デフォルトで `infinity の日付とタイムスタンプがマッピングされるのは 'date.max'`__ および同様の定数です。これには、全単射ではないマッピングを作成してしまうという問題があります (2 つの Postgres の日付、infinity と 9999-12-31が、両方とも同じ Python の日付にマッピングされてしまいます)。有効な PostgreSQL の日付 (Python の `!date.max` より大きいが、おそらく無限大より小さい) が依然としてオーバーフローしてしまうという歪みもあります。

..
    In Psycopg 3, every date greater than year 9999 will overflow, including
    infinity. If you would like to customize this mapping (for instance flattening
    every date past Y10K on `!date.max`) you can subclass and adapt the
    appropriate loaders: take a look at :ref:`this example
    <adapt-example-inf-date>` to see how.

psycopg 3 では、9999 年より大きいすべての日付は、infinity を含めてオーバーフローします。このマッピングをカスタマイズしたい (たとえば、10,000 年を超えるすべての日付を `!date.max` に平坦化する) 場合には、適切なローダーをサブクラス化して適応できます。方法を学ぶには :ref:`この例 <adapt-example-inf-date>` を参照してください。

.. __: https://www.psycopg.org/docs/usage.html#infinite-dates-handling

..
    .. _whats-new:

    What's new in Psycopg 3
    -----------------------

.. _whats-new:

psycopg 3 の新機能
-----------------------

..
    - :ref:`Asynchronous support <async>`
    - :ref:`Server-side parameters binding <server-side-binding>`
    - :ref:`Prepared statements <prepared-statements>`
    - :ref:`Binary communication <binary-data>`
    - :ref:`Python-based COPY support <copy>`
    - :ref:`Support for static typing <static-typing>`
    - :ref:`A redesigned connection pool <connection-pools>`
    - :ref:`Direct access to the libpq functionalities <psycopg.pq>`

- :ref:`非同期のサポート <async>`
- :ref:`サーバーサイド パラメータ バインディング <server-side-binding>`
- :ref:`prepare されたステートメント <prepared-statements>`
- :ref:`バイナリ通信 <binary-data>`
- :ref:`Python ベースの COPY のサポート <copy>`
- :ref:`静的型付けのサポート <static-typing>`
- :ref:`再デザインされたコネクションプール <connection-pools>`
- :ref:`libpq の機能への直接アクセス <psycopg.pq>`
