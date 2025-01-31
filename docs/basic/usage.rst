.. currentmodule:: psycopg

..
    .. _module-usage:

    Basic module usage
    ==================

.. _module-usage:

モジュールの基本的な使用方法
============================

..
    The basic Psycopg usage is common to all the database adapters implementing
    the `DB-API`__ protocol. Other database adapters, such as the builtin
    `sqlite3` or `psycopg2`, have roughly the same pattern of interaction.

psycopg の基本的な使用方法は、`DB-API`__ プロトコルを実装しているすべてのデータべース アダプターと共通するものです。`sqlite3` や `psycopg2` などの他のデータベース アダプターには、おおよそ同じ対話のパターンがあります。

.. __: https://www.python.org/dev/peps/pep-0249/

.. index::
    pair: Example; Usage

..
    .. _usage:

    Main objects in Psycopg 3
    -------------------------

.. _usage:

psycopg 3 のメイン オブジェクト
-------------------------------

..
    Here is an interactive session showing some of the basic commands:

次に示すのは、基本的なコマンドの一部を表示している対話セッションです。

..
    .. code:: python

        # Note: the module name is psycopg, not psycopg3
        import psycopg

        # Connect to an existing database
        with psycopg.connect("dbname=test user=postgres") as conn:

            # Open a cursor to perform database operations
            with conn.cursor() as cur:

                # Execute a command: this creates a new table
                cur.execute("""
                    CREATE TABLE test (
                        id serial PRIMARY KEY,
                        num integer,
                        data text)
                    """)

                # Pass data to fill a query placeholders and let Psycopg perform
                # the correct conversion (no SQL injections!)
                cur.execute(
                    "INSERT INTO test (num, data) VALUES (%s, %s)",
                    (100, "abc'def"))

                # Query the database and obtain data as Python objects.
                cur.execute("SELECT * FROM test")
                cur.fetchone()
                # will return (1, 100, "abc'def")

                # You can use `cur.fetchmany()`, `cur.fetchall()` to return a list
                # of several records, or even iterate on the cursor
                for record in cur:
                    print(record)

                # Make the changes to the database persistent
                conn.commit()

.. code:: python

    # メモ: モジュール名は psycopg3 ではなく psycopg
    import psycopg

    # 既存のデータベースに接続する
    with psycopg.connect("dbname=test user=postgres") as conn:

        # データベースの操作を実行するカーソルをオープンする
        with conn.cursor() as cur:

            # コマンドの実行: 新しいテーブルを作成する
            cur.execute("""
                CREATE TABLE test (
                    id serial PRIMARY KEY,
                    num integer,
                    data text)
                """)

            # クエリのプレースホルダーを埋めるためにデータを渡して
            # psycopg に正しい変換を行わせる (SQL インジェクションなし！)
            cur.execute(
                "INSERT INTO test (num, data) VALUES (%s, %s)",
                (100, "abc'def"))

            # データベースにクエリし、データを Python オブジェクトとして取得する
            cur.execute("SELECT * FROM test")
            cur.fetchone()
            # これは (1, 100, "abc'def") を返す

            # `cur.fetchmany()` や `cur.fetchall()` を使用して複数のレコードのリストを返したり
            # または、カーソルをイテレートすることもできる
            for record in cur:
                print(record)

            # データベースへの変更を永続化する
            conn.commit()

..
    In the example you can see some of the main objects and methods and how they
    relate to each other:

この例では、メインオブジェクトとメソッドの一部と、それらが互いにどう関係しているのかがわかります。

..
    - The function `~Connection.connect()` creates a new database session and
      returns a new `Connection` instance. `AsyncConnection.connect()`
      creates an `asyncio` connection instead.

- 関数 `~Connection.connect()` は、新しいデータベース セッションを作成し、新しい `Connection` インスタンスを返します。`AsyncConnection.connect()` は、代わりに `asyncio` コネクションを返します。

..
    - The `~Connection` class encapsulates a database session. It allows to:

      - create new `~Cursor` instances using the `~Connection.cursor()` method to
        execute database commands and queries,

      - terminate transactions using the methods `~Connection.commit()` or
        `~Connection.rollback()`.

- `~Connection` クラスはデータベースセッションをカプセル化していて、これを使用して次のことができます。

  - `~Connection.cursor()` メソッドを使用して、データベースコマンドとクエリを実行するための新しい `~Cursor` インスタンスを作成できます。

  - `~Connection.commit()` や `~Connection.rollback()` メソッドを使用してトランザクションを終端できます。

..
    - The class `~Cursor` allows interaction with the database:

      - send commands to the database using methods such as `~Cursor.execute()`
        and `~Cursor.executemany()`,

      - retrieve data from the database, iterating on the cursor or using methods
        such as `~Cursor.fetchone()`, `~Cursor.fetchmany()`, `~Cursor.fetchall()`.

- `~Cursor` クラスを使用すると、次のようにデータベースとの対話ができます。

  - `~Cursor.execute()` や `~Cursor.executemany()` などのメソッドを使用して、データベースにコマンドを送れます。

  - カーソル上でイテレートしたり、`~Cursor.fetchone()`、`~Cursor.fetchmany()`、`~Cursor.fetchall()` などのメソッドを使用したりして、データベースからデータを取得できます。

..
    - Using these objects as context managers (i.e. using `!with`) will make sure
      to close them and free their resources at the end of the block (notice that
      :ref:`this is different from psycopg2 <diff-with>`).

- これらのオブジェクトをコンテクスト マネージャとして使用することで (つまり `!with` を使用することで)、オブジェクトを確実にクローズしてオブジェクトのリソースをブロックの最後で開放できる (:ref:`これは psycopg2 とは異なる <diff-with>` ことに注意してください)。

..
    .. seealso::

        A few important topics you will have to deal with are:

.. seealso::

    後に対処する必要になる重要なトピックがいくつかあります。

    - :ref:`query-parameters`
    - :ref:`types-adaptation`
    - :ref:`transactions`

..
    Shortcuts
    ---------

ショートカット
--------------

..
    The pattern above is familiar to `!psycopg2` users. However, Psycopg 3 also
    exposes a few simple extensions which make the above pattern leaner:

上記のパターンは `!psycopg2` ユーザーには馴染みのあるものです。しかし、psycopg 3 は上記のパターンをより簡潔にするシンプルな拡張もいくつか公開しています。

..
    - the `Connection` objects exposes an `~Connection.execute()` method,
      equivalent to creating a cursor, calling its `~Cursor.execute()` method, and
      returning it.

- `Connection` オブジェクトは、`~Cursor.execute()` メソッドを呼び出して返すカーソルの作成に相当する `~Connection.execute()` メソッドを公開しています。

  .. code::

      # In Psycopg 2
      cur = conn.cursor()
      cur.execute(...)

      # In Psycopg 3
      cur = conn.execute(...)

..
    - The `Cursor.execute()` method returns `!self`. This means that you can chain
      a fetch operation, such as `~Cursor.fetchone()`, to the `!execute()` call:

- `Cursor.execute()` メソッドは `!self` を返します。つまり、`~Cursor.fetchone()` などの fetch 操作を `!execute()` の呼び出しにチェーンできます。

  .. code::

      # In Psycopg 2
      cur.execute(...)
      record = cur.fetchone()

      cur.execute(...)
      for record in cur:
          ...

      # In Psycopg 3
      record = cur.execute(...).fetchone()

      for record in cur.execute(...):
          ...

..
    Using them together, in simple cases, you can go from creating a connection to
    using a result in a single expression:

単純なケースでは、これらを一緒に使うことで、コネクションの作成から1つの式の結果の使用までが実行できます。

.. code::

    print(psycopg.connect(DSN).execute("SELECT now()").fetchone()[0])
    # 2042-07-12 18:15:10.706497+01:00

..
    .. index::
        pair: Connection; `!with`

    .. _with-connection:

    Connection context
    ------------------

.. index::
    pair: Connection; `!with`

.. _with-connection:

コネクション コンテクスト
----------------------

..
    Psycopg 3 `Connection` can be used as a context manager:

psycopg 3 の `Connection` はコンテクスト マネージャとして使用できます。

..
    .. code:: python

        with psycopg.connect() as conn:
            ... # use the connection

        # the connection is now closed

.. code:: python

    with psycopg.connect() as conn:
        ... # コネクションを使用する

    # ここでコネクションは閉じられている

..
    When the block is exited, if there is a transaction open, it will be
    committed. If an exception is raised within the block the transaction is
    rolled back. In both cases the connection is closed. It is roughly the
    equivalent of:

ブロックを出るときに、トランザクションがオープンであれば、そのトランザクションはコミットされます。ブロック内で例外が発生した場合、トランザクションはロールバックされます。どちらの場合もコネクションはクローズされます。これはおおよそ次のコードと同等です。

..
    .. code:: python

        conn = psycopg.connect()
        try:
            ... # use the connection
        except BaseException:
            conn.rollback()
        else:
            conn.commit()
        finally:
            conn.close()

.. code:: python

    conn = psycopg.connect()
    try:
        ... # コネクションを使用する
    except BaseException:
        conn.rollback()
    else:
        conn.commit()
    finally:
        conn.close()

..
    .. note::
        This behaviour is not what `!psycopg2` does: in `!psycopg2` :ref:`there is
        no final close() <pg2:with>` and the connection can be used in several
        `!with` statements to manage different transactions. This behaviour has
        been considered non-standard and surprising so it has been replaced by the
        more explicit `~Connection.transaction()` block.

.. note::
    この動作は `!psycopg2` の動作とは異なります。`!psycopg2` では、:ref:`最後の close() は存在せず <pg2:with>`、コネクションは異なるトランザクションを管理するために複数の `!with` ステートメントで使用できます。この動作は非標準でユーザーを驚かせるものだと考えられてきたため、より明示的な `~Connection.transaction()` ブロックと置換されました。

..
    Note that, while the above pattern is what most people would use, `connect()`
    doesn't enter a block itself, but returns an "un-entered" connection, so that
    it is still possible to use a connection regardless of the code scope and the
    developer is free to use (and responsible for calling) `~Connection.commit()`,
    `~Connection.rollback()`, `~Connection.close()` as and where needed.

上記のパターンはほとんどの人が使うものですが、`connect()` がブロック自体に入るわけではなく、入る前のコネクション ("un-entered" connection) を返すことに注意してください。そのため、コネクションはコードのスコープに関わらずまだ使える可能性があり、開発者は必要なときに必要に応じて `~Connection.commit()`、`~Connection.rollback()`、`~Connection.close()` を自由に使えます (そして、開発者にはそれらを呼び出す責任があります)。

..
    .. warning::
        If a connection is just left to go out of scope, the way it will behave
        with or without the use of a `!with` block is different:

        - if the connection is used without a `!with` block, the server will find
          a connection closed INTRANS and roll back the current transaction;

        - if the connection is used with a `!with` block, there will be an
          explicit COMMIT and the operations will be finalised.

        You should use a `!with` block when your intention is just to execute a
        set of operations and then committing the result, which is the most usual
        thing to do with a connection. If your connection life cycle and
        transaction pattern is different, and want more control on it, the use
        without `!with` might be more convenient.

        See :ref:`transactions` for more information.

.. warning::
    コネクションがそのままスコープ外に残された場合、`!with` ブロックを使用する場合と使用しない場合でその動作方法は異なります。

    - コネクションが `!with` ブロックなしで使用された場合、サーバーは INTRANS で閉じられたコネクションを探して現在のトランザクションをロールバックします。

    - コネクションが `!with` ブロックとともに使用された場合、明示的な COMMIT があり、操作はファイナライズされます。

    もし一連の操作をただ実行して、そして結果をコミットすることを意図しているのなら、`!with` ブロックを使うべきです。これはコネクションを使用して行う最も一般的な操作です。コネクションのライフサイクルとトランザクションのパターンが異なり、コネクションをよりコントロールしたい場合には、`!with` なしに使用するほうがより便利なことがあるかもしれません。

    より詳しい情報については :ref:`transactions` を参照してください。

`AsyncConnection` も ``async with`` を使用してコンテクスト マネージャとして使用できますが、変わった振る舞いをするため注意してください。詳細は :ref:`async-with` を参照してください。

..
    Adapting pyscopg to your program
    --------------------------------

psycopg をプログラムに適応させる
--------------------------------

..
    The above :ref:`pattern of use <usage>` only shows the default behaviour of
    the adapter. Psycopg can be customised in several ways, to allow the smoothest
    integration between your Python program and your PostgreSQL database:

上記の :ref:`使用パターン <usage>` はアダプターのデフォルトの動作を示したに過ぎません。psycopg は複数の方法でカスタマイズして、Python プログラムと PostgreSQL データベース間で最もスムーズなインテグレーションができるようになります。

..
    - If your program is concurrent and based on `asyncio` instead of on
      threads/processes, you can use :ref:`async connections and cursors <async>`.

- プログラムが並行で スレッド/プロセスの代わりに `asyncio` を元にしている場合、:ref:`async のコネクションとカーソル <async>` が利用できます。

..
    - If you want to customise the objects that the cursor returns, instead of
      receiving tuples, you can specify your :ref:`row factories <row-factories>`.

- カーソルが返すオブジェクトをタプルを受け取るのではなくカスタマイズしたい場合、:ref:`行ファクトリ <row-factories>` を指定できます。

..
    - If you want to customise how Python values and PostgreSQL types are mapped
      into each other, beside the :ref:`basic type mapping <types-adaptation>`,
      you can :ref:`configure your types <adaptation>`.

- Python の値と PostgreSQL の型を相互にマッピングする方法をカスタマイズしたい場合、:ref:`基本的な型のマッピング <types-adaptation>` に加えて、:ref:`独自の型を設定 <adaptation>` できます。
