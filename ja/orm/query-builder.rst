クエリービルダー
################

.. php:namespace:: Cake\ORM

.. php:class:: Query

ORM のクエリービルダーにより簡単に流れるようなインターフェイスを使ってクエリーを作り、
実行することができます。クエリーを組み立てることで、 union やサブクエリーを使った
高度なクエリーも簡単に作成することができます。

クエリービルダーは裏側で PDO プリペアードステートメント（prepared statement）を使うことで、
SQL インジェクション攻撃から守っています。

クエリーオブジェクト
====================

``Query`` オブジェクトを作成するもっとも簡単な方法は ``Table`` オブジェクトから ``find()``
を使うことです。このメソッドは完結していない状態のクエリーを返し、このクエリーは変更可能です。
必要なら、テーブルのコネクションオブジェクトも使うことで、ORM 機能を含まない、
より低い層のクエリービルダーにアクセスすることもできます。この詳細は :ref:`database-queries`
のセクションを参照してください。 ::

    use Cake\ORM\TableRegistry;

    $articles = TableRegistry::getTableLocator()->get('Articles');

    // 新しいクエリーを始めます。
    $query = $articles->find();

コントローラーの中では自動的に慣習的な機能を使って作成されるテーブル変数を使うことができます。 ::

    // ArticlesController.php の中で

    $query = $this->Articles->find();

テーブルから行を取得する
------------------------

::

    use Cake\ORM\Locator\LocatorAwareTrait;

    $query = $this->getTableLocator()->get('Articles')->find();

    foreach ($query->all() as $article) {
        debug($article->title);
    }

以降の例では ``$articles`` は :php:class:`~Cake\\ORM\\Table` であると想定します。
なお、コントローラーの中では ``$articles`` の代わりに ``$this->Articles`` を使うことができます。

``Query`` オブジェクトのほとんどのメソッドが自分自身のクエリーオブジェクトを返します。
これは ``Query`` が遅延評価される(lazy)ことを意味し、必要になるまで実行されないことを
意味します。 ::

    $query->where(['id' => 1]); // 自分自身のクエリーオブジェクトを返します
    $query->order(['title' => 'DESC']); // 自分自身を返し、SQL はまだ実行されません。

もちろん Query オブジェクトの呼び出しではメソッドをチェーンすることもできます。 ::

    $query = $articles
        ->find()
        ->select(['id', 'name'])
        ->where(['id !=' => 1])
        ->order(['created' => 'DESC']);

    foreach ($query->all() as $article) {
        debug($article->created);
    }

Query オブジェクトを ``debug()`` で使うと、内部の状態とデータベースで実行されることになる
SQL が出力されます。 ::

    debug($articles->find()->where(['id' => 1]));

    // 出力
    // ...
    // 'sql' => 'SELECT * FROM articles where id = ?'
    // ...

``foreach`` を使わずに、クエリーを直接実行することができます。もっとも簡単なのは
``all()`` メソッドか ``toList()`` メソッドのどちらかを呼ぶ方法です。 ::

    $resultsIteratorObject = $articles
        ->find()
        ->where(['id >' => 1])
        ->all();

    foreach ($resultsIteratorObject as $article) {
        debug($article->id);
    }

    $resultsArray = $articles
        ->find()
        ->where(['id >' => 1])
        ->all()
        ->toList();

    foreach ($resultsArray as $article) {
        debug($article->id);
    }

    debug($resultsArray[0]->title);

上記の例では、 ``$resultsIteratorObject`` は ``Cake\ORM\ResultSet`` のインスタンスです。
このインスタンスはイテレートすることができ、それが持つメソッドで部分取り出しなどができます。

多くの場合、 ``all()`` を呼ぶ必要はなく、単に Query オブジェクトをイテレートすることで、
結果を得ることができます。Query オブジェクトはまた、結果オブジェクトとして直接使うこともできます。
クエリーをイテレートしたり、 ``toList()`` を呼んだり、
:doc:`Collection </core-libraries/collections>` から継承したメソッドを呼んだりすると、
クエリーは実行され、結果が返ります。

テーブルから単一行を取得する
----------------------------

``first()`` メソッドを使うことでクエリーの最初の結果を取得することができます。 ::

    $article = $articles
        ->find()
        ->where(['id' => 1])
        ->first();

    debug($article->title);

カラムから値リストを取得する
----------------------------

::

    // Collection ライブラリーの extract() メソッドを使います
    // これもクエリーを実行します
    $allTitles = $articles->find()->all()->extract('title');

    foreach ($allTitles as $title) {
        echo $title;
    }

クエリーの結果から key-value リストを得ることもできます。 ::

    $list = $articles->find('list');

    $list = $articles->find('list')->all();
    foreach ($list as $id => $title) {
        echo "$id : $title"
    }

リストを生成するために使用するフィールドのカスタマイズの方法の詳しい情報は、
:ref:`table-find-list` セクションを参照してください。

クエリーは Collection オブジェクトである
----------------------------------------

Query オブジェクトのメソッドに慣れたら、 :doc:`Collection </core-libraries/collections>`
を見て、効果的にデータを横断するスキルを磨くことを強くお勧めします。つまり、 Collection
オブジェクトで呼ぶことができるものは、 Query オブジェクトでも呼ぶことができることを、
知っておくことは重要です。 ::

    // Collection ライブラリーの combine() メソッドを使います
    // これは find('list') と等価です
    $keyValueList = $articles->find()->combine('id', 'title');

    // 上級な例
    $results = $articles->find()
        ->where(['id >' => 1])
        ->order(['title' => 'DESC'])
        ->all()
        ->map(function ($row) {
        // map() は Collection のメソッドで、クエリーを実行します
            $row->trimmedTitle = trim($row->title);

            return $row;
        })
        ->combine('id', 'trimmedTitle') // combine() も Collection のメソッドです
        ->toArray(); // これも Collection のメソッドです

    foreach ($results as $id => $trimmedTitle) {
        echo "$id : $trimmedTitle";
    }

クエリーの遅延評価
------------------

Query オブジェクトは遅延評価されます。
これはクエリーが次のいずれかが起こるまで実行されないということを意味します。

- クエリーが ``foreach()`` でイテレートされる。
- クエリーの ``execute()`` メソッドが呼ばれる。これは下層の statement オブジェクトを返し、
  insert/update/delete クエリーで使うことができます。
- クエリーの ``first()`` メソッドが呼ばれる。 ``SELECT`` (それがクエリーに ``LIMIT 1``
  を加えます) で構築された結果セットの最初の結果が返ります。
- クエリーの ``all()`` メソッドが呼ばれる。結果セットが返り、
  ``SELECT`` ステートメントでのみ使うことができます。
- クエリーの ``toList()`` や ``toArray()`` メソッドが呼ばれる。

このような条件が合致するまでは、 SQL をデータベースへ送らずに、クエリーを変更することができます。
つまり、 Query が評価されないかぎり、SQL はデータベースへ送信されないのです。
クエリーが実行された後に、クエリーを変更・再評価したら、追加で SQL が走ることになります。

CakePHP が生成している SQL がどんなものか見たいなら、
:ref:`クエリーロギング <database-query-logging>` を on にしてください。

データを select する
====================

CakePHP では ``SELECT`` クエリーを簡単につくれます。取得するフィールドを制限するのには、
``select()`` メソッドを使います。 ::

    $query = $articles->find();
    $query->select(['id', 'title', 'body']);
    foreach ($query as $row) {
        debug($row->title);
    }

連想配列でフィールドを渡すことでフィールドのエイリアス (別名) をセットすることができます。 ::

    // SELECT id AS pk, title AS aliased_title, body ... になる
    $query = $articles->find();
    $query->select(['pk' => 'id', 'aliased_title' => 'title', 'body']);

フィールドを select distinct するために、 ``distinct()`` メソッドを使うことができます。 ::

    // SELECT DISTINCT country FROM ... になる
    $query = $articles->find();
    $query->select(['country'])
        ->distinct(['country']);

基本の条件をセットするには、 ``where()`` メソッドを使うことができます。 ::

    // 条件は AND で連結されます
    $query = $articles->find();
    $query->where(['title' => 'First Post', 'published' => true]);

    // where() を複数回呼んでもかまいません
    $query = $articles->find();
    $query->where(['title' => 'First Post'])
        ->where(['published' => true]);

無名関数を ``where()`` メソッドに渡すこともできます。渡された無名関数は、第一引数として
``\Cake\Database\Expression\QueryExpression`` のインスタンス、第二引数として
``\Cake\ORM\Query`` を受け取ります。 ::

    $query = $articles->find();
    $query->where(function (QueryExpression $exp, Query $q) {
        return $exp->eq('published', true);
    });

さらに複雑な ``WHERE`` 条件の作り方を知りたい場合は :ref:`advanced-query-conditions`
のセクションをご覧ください。ソート順を適用するために、 ``order`` メソッドが使用できます。 ::

    $query = $articles->find()
        ->order(['title' => 'ASC', 'id' => 'ASC']);

一つのクエリーで ``order()`` を複数回呼ぶと、複数の句が追加されます。
しかし、ファインダーを使用する時、 ``ORDER BY`` を上書きする必要が生じることもあります。
``order()`` (``orderAsc()`` や ``orderDesc()`` も同様に) の第２パラメーターに
``Query::OVERWRITE`` または ``true`` をセットしてください。 ::

    $query = $articles->find()
        ->order(['title' => 'ASC']);
    // あとで、ORDER BY 句を追加の代わりに上書きします。
    $query = $articles->find()
        ->order(['created' => 'DESC'], Query::OVERWRITE);

複合的な式でソートする必要があるなら ``order`` に加えて、 ``orderAsc`` と ``orderDesc``
メソッドが使えます。 ::

    $query = $articles->find();
    $concat = $query->func()->concat([
        'title' => 'identifier',
        'synopsis' => 'identifier'
    ]);
    $query->orderAsc($concat);

行の数を制限したり、行のオフセットをセットするためには、 ``limit()`` と ``page()``
メソッドを使うことができます。 ::

    // 50 から 100 行目をフェッチする
    $query = $articles->find()
        ->limit(50)
        ->page(2);

上記の例にあるように、クエリーを変更するすべてのメソッドは流暢 (fluent)
なインターフェイスを提供しますので、クエリーを構築する際にチェーンメソッドの形で呼び出すことができます。

特定のフィールドを選択
----------------------

クエリーはデフォルトでテーブルのすべてのフィールドを選択します。
例外となるのは ``select()`` 関数をあえて呼び、特定のフィールドを指定した場合だけです。 ::

    // articles テーブルから id と title だけが選択される
    $articles->find()->select(['id', 'title']);

``select($fields)`` を呼んで、なおもテーブルのすべてのフィールドを選択したいなら、
次の方法でテーブルインスタンスを ``select()`` に渡すことができます。 ::

    // 計算された slug フィールドを含む、 articles テーブルのすべてのフィールドを取得
    $query = $articlesTable->find();
    $query
        ->select(['slug' => $query->func()->concat(['title' => 'identifier', '-', 'id' => 'identifier'])])
        ->select($articlesTable); // articles のすべてのフィールドを選択する

テーブル上のいくつかのフィールド以外のすべてのフィールドを選択したい場合には
``selectAllExcept()`` を使用できます。 ::

    $query = $articlesTable->find();

    // published フィールドを除く全てのフィールドを取得
    $query->selectAllExcept($articlesTable, ['published']);

アソシエーションが含まれる場合、 ``Association`` オブジェクトを渡すこともできます。

.. _using-sql-functions:

SQL 関数を使う
--------------

CakePHP の ORM では抽象化された馴染み深い SQL 関数をいくつか使えるようになっています。
抽象化により ORM は、プラットフォーム固有の実装を選んで関数を実行できるようになっています。
たとえば、 ``concat`` は MySQL、PostgreSQL、SQL Server で異なる実装がされています。
抽象化によりあなたのコードが移植しやすいものになります。 ::

    // SELECT COUNT(*) count FROM ... になる
    $query = $articles->find();
    $query->select(['count' => $query->func()->count('*')]);

多くのおなじみの関数が ``func()`` メソッドとともに作成できます:

``rand()``
    0から1の間の乱数をSQLで生成します。
``sum()``
    合計を算出します。 `引数はリテラル値として扱われます。`
``avg()``
    平均値を算出します。 `引数はリテラル値として扱われます。`
``min()``
    カラムの最小値を算出します。 `引数はリテラル値として扱われます。`
``max()``
    カラムの最大値を算出します。 `引数はリテラル値として扱われます。`
``count()``
    件数を算出します。 `引数はリテラル値として扱われます。`
``concat()``
    ２つの値を結合します。 `引数はバインドパラメーターとして扱われます。`
``coalesce()``
    Coalesce を算出します。 `引数はバインドパラメーターとして扱われます。`
``dateDiff()``
    ２つの日にち/時間の差を取得します。 `引数はバインドパラメーターとして扱われます。`
``now()``
    デフォルトでは日付と時刻を返しますが、 'time' または 'date' を指定してこれらの値のみを返すこともできます。
``extract()``
    SQL 式から特定の日付部分(年など)を返します。
``dateAdd()``
    日付式に単位時間を加算します。
``dayOfWeek()``
    SQL の WEEKDAY 関数を呼ぶ FunctionExpression を返します。

SQL 関数に渡す引数には、リテラルの引数と、バインドパラメーターの２種類がありえます。
識別子やリテラルのパラメーターにより、カラムや他の SQL リテラルを参照できます。
バインドパラメーターにより、ユーザーデータを SQL 関数へと安全に渡すことができます。
たとえば ::

    $query = $articles->find()->innerJoinWith('Categories');
    $concat = $query->func()->concat([
        'Articles.title' => 'identifier',
        ' - CAT: ',
        'Categories.name' => 'identifier',
        ' - Age: ',
        $query->func()->dateDiff(
            'NOW()' => 'literal',
            'Articles.created' => 'identifier'
        )
    ]);
    $query->select(['link_title' => $concat]);

``literal`` の値を伴う引数を作ることで、 ORM はそのキーをリテラルな SQL 値として扱うべきであると
知ることになります。 ``identifier`` の値を伴う引数を作ることで、ORM は、そのキーがフィールドの
識別子として扱うべきであると知ることになります。

上記では MySQL にて下記の SQL が生成されます。

.. code-block:: mysql

    SELECT CONCAT(
        Articles.title,
        :c0,
        Categories.name,
        :c1,
        (DATEDIFF(NOW(), Articles.created))
    ) FROM articles;

クエリーが実行される際には、 ``:c0`` という値に ``' - CAT'`` というテキストがバインドされることになります。
``dateDiff`` 式は適切なSQLに変換されます。

カスタム関数
^^^^^^^^^^^^

もし ``func()`` が必要なSQL関数をラップしていない場合は、 ``func()`` を使って直接呼び出すことができ、
引数やユーザデータを安全に渡すことができます。
カスタム関数には適切な型の引数を渡すようにしてください。 ::

    $query = $articles->find();
    $year = $query->func()->year([
        'created' => 'identifier'
    ]);
    $time = $query->func()->date_format([
        'created' => 'identifier',
        "'%H:%i'" => 'literal'
    ]);
    $query->select([
        'yearCreated' => $year,
        'timeCreated' => $time
    ]);

これらのカスタム関数は、 MySQL にて下記の SQL を生成します。

.. code-block:: mysql

    SELECT YEAR(created) as yearCreated,
           DATE_FORMAT(created, '%H:%i') as timeCreated
    FROM articles;

.. note::
    信頼されていないユーザのデータを任意のSQL関数に渡すには ``func()`` を使ってください。

集約 - Group と Having
----------------------

``count`` や ``sum`` のような集約関数を使う際には、
``group by`` や ``having`` 句を使いたいことでしょう。 ::

    $query = $articles->find();
    $query->select([
        'count' => $query->func()->count('view_count'),
        'published_date' => 'DATE(created)'
    ])
    ->group('published_date')
    ->having(['count >' => 3]);

Case 文
-------

ORM ではまた、 SQL の ``case`` 式も使えます。 ``case`` 式により ``if ... then ... else``
のロジックを SQL の中に実装することができます。これは条件付きで sum や count をしなければならない
状況や、条件に基いてデータを特定しなければならない状況で、データを出力するのに便利です。

公開済みの記事（published articles）がデータベース内にいくつあるのか知りたい場合、次の
SQL を使用することができます。

.. code-block:: sql

    SELECT
    COUNT(CASE WHEN published = 'Y' THEN 1 END) AS number_published,
    COUNT(CASE WHEN published = 'N' THEN 1 END) AS number_unpublished
    FROM articles

これはクエリービルダーでは次のようなコードになります。 ::

    $query = $articles->find();
    $publishedCase = $query->newExpr()
        ->addCase(
            $query->newExpr()->add(['published' => 'Y']),
            1,
            'integer'
        );
    $unpublishedCase = $query->newExpr()
        ->addCase(
            $query->newExpr()->add(['published' => 'N']),
            1,
            'integer'
        );

    $query->select([
        'number_published' => $query->func()->count($publishedCase),
        'number_unpublished' => $query->func()->count($unpublishedCase)
    ]);

``addCase`` 関数は SQL 内で ``if .. then .. [elseif .. then .. ] [ .. else ]``
ロジックを構築するために複数の文を一まとめに書くことができます。

町 (city) を人口 (population size) に基いて SMALL、MEDIUM、LARGE に分類するなら、
次のようになります。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->addCase(
                [
                    $q->newExpr()->lt('population', 100000),
                    $q->newExpr()->between('population', 100000, 999000),
                    $q->newExpr()->gte('population', 999001),
                ],
                ['SMALL',  'MEDIUM', 'LARGE'], # 条件に合致したときの値
                ['string', 'string', 'string'] # それぞれの値の型
            );
        });
    # WHERE CASE
    #   WHEN population < 100000 THEN 'SMALL'
    #   WHEN population BETWEEN 100000 AND 999000 THEN 'MEDIUM'
    #   WHEN population >= 999001 THEN 'LARGE'
    #   END

値リストよりも case 条件リストの方が少ない場合はいつでも、 ``addCase`` は自動的に
``if .. then .. else`` 文を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->addCase(
                [
                    $q->newExpr()->eq('population', 0),
                ],
                ['DESERTED', 'INHABITED'], # 条件に合致したときの値
                ['string', 'string'] # それぞれの値の型
            );
        });
    # WHERE CASE
    #   WHEN population = 0 THEN 'DESERTED' ELSE 'INHABITED' END

エンティティーの代わりに配列を取得
----------------------------------

ORM とオブジェクトの結果セットは強力である一方で、エンティティーの作成が不要なときもあります。
たとえば、集約されたデータにアクセスする場合、Entity を構築するのは無意味です。
データベースの結果をエンティティーに変換する処理は、ハイドレーション (hydration) と呼ばれます。
もし、この処理を無効化したい場合、このようにします。 ::

    $query = $articles->find();
    $query->enableHydration(false); // エンティティーの代わりに配列を返す
    $result = $query->toList(); // クエリーを実行し、配列を返す

これらの行を実行した後、結果はこのようになります。 ::

    [
        ['id' => 1, 'title' => 'First Article', 'body' => 'Article 1 body' ...],
        ['id' => 2, 'title' => 'Second Article', 'body' => 'Article 2 body' ...],
        ...
    ]

.. _format-results:

計算フィールドを追加する
------------------------

クエリー後に何か後処理をする必要があるかもしれません。
計算フィールドや生成された (derived) データをいくつか追加する必要があるなら、
``formatResults()`` メソッドを使うことができます。
これにより軽い負荷で、結果セットを map することができます。
この処理をこれ以上に制御する必要があるなら、もしくは、結果セットを reduce する必要があるなら、
:ref:`Map/Reduce <map-reduce>` 機能を代わりに使ってください。
人々のリストを問い合わせる際に、formatResults を使って年齢 (age) を算出するなら次のようになります。 ::

    // フィールド、条件、関連が構築済であると仮定します。
    $query->formatResults(function (\Cake\Collection\CollectionInterface $results) {
        return $results->map(function ($row) {
            $row['age'] = $row['birth_date']->diff(new \DateTime)->y;

            return $row;
        });
    });

上記の例にあるように、フォーマッタ関数(フォーマットするコールバック)の第１引数に
``ResultSetDecorator`` が渡されています。第２引数にはフォーマッタ関数がセットされる
Query インスタンスが渡されます。引数の ``$results`` は必要に応じて、
取り出しでも変換でもすることができます。

フォーマッタ関数は、クエリーが値を返せるようにするために、イテレータオブジェクトを返す必要があります。
フォーマッタ関数はすべての Map/Reduce が実行し終わった後、適用されます。
contain された関連の中からでも同じようにフォーマッタ関数を適用することができます。
CakePHP はフォーマッタ関数が適切なスコープになるよう保証します。
たとえば、下記のようにした場合でも、期待どおりに動きます。 ::

    // Articles テーブル内のメソッドで
    $query->contain(['Authors' => function ($q) {
        return $q->formatResults(function (\Cake\Collection\CollectionInterface $authors) {
            return $authors->map(function ($author) {
                $author['age'] = $author['birth_date']->diff(new \DateTime)->y;

                return $author;
            });
        });
    }]);

    // 結果を取得する
    $results = $query->all();

    // 29 が出力される
    echo $results->first()->author->age;

上記にあるように、関連付いたクエリービルダーに設置されたフォーマッタ関数は、
関連付いたデータの中だけの操作にスコープが限定されます。
CakePHP は計算された値が正しい Entity にセットされることを保証します。

.. _advanced-query-conditions:

高度な条件
==========

クエリービルダーは複雑な ``where`` 句の構築を簡単にします。
グループ化された条件は、 ``where()`` と Expression オブジェクトを組み合わせることで表現できます。
単純なクエリーの場合、条件の配列を使用して条件を作成できます。 ::

    $query = $articles->find()
        ->where([
            'author_id' => 3,
            'OR' => [['view_count' => 2], ['view_count' => 3]],
        ]);

上記は次のような SQL を生成します。

.. code-block:: sql

    SELECT * FROM articles WHERE author_id = 3 AND (view_count = 2 OR view_count = 3)

深くネストされた配列を避けたい場合は、 ``where()`` のコールバック形式を使用して
クエリーを構築することができます。コールバック形式を使用すると、
式ビルダーを使用して配列なしでより複雑な条件を作成できます。例::

    $query = $articles->find()->where(function (QueryExpression $exp, Query $query) {
        // 同一フィールドに複数条件を追加するために add() を使用
        $author = $query->newExpr()->or(['author_id' => 3])->add(['author_id' => 2]);
        $published = $query->newExpr()->and(['published' => true, 'view_count' => 10]);

        return $exp->or([
            'promoted' => true,
            $query->newExpr()->and([$author, $published])
        ]);
    });

上記は次のような SQL を出力します。

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        (
            (author_id = 2 OR author_id = 3)
            AND
            (published = 1 AND view_count = 10)
        )
        OR promoted = 1
    )

コールバックに渡される ``QueryExpression`` は完全な式を構築するために、
**combinators** と **conditions** を使用します。

Combinators
    これらは新しい ``QueryExpression`` オブジェクトを作成し、
    その式に追加された条件をどのように結合するかを設定します。

    - ``and()`` は、すべての条件を ``AND`` で結合する新しい式オブジェクトを作成します。
    - ``or() `` は、すべての条件を ``OR`` で結合する新しい式オブジェクトを作成します。

Conditions
    これらは式に追加され、どの組み合わせが使用されたかに応じて自動的に結合されます。

コールバック関数に渡される QueryExpression のデフォルトは ``and()`` です。 ::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            return $exp
                ->eq('author_id', 2)
                ->eq('published', true)
                ->notEq('spam', true)
                ->gt('view_count', 10);
        });

``where()`` を使い始めた場合、 ``and()`` は暗黙的に選ばれているため、それを呼ぶ必要はありません。
上記の例では新たな条件がいくつか ``AND`` で結合されています。結果の SQL は次のようになります。

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
    author_id = 2
    AND published = 1
    AND spam != 1
    AND view_count > 10)

ただし、 ``AND`` と ``OR`` の両方を使いたいなら、次のようにすることもできます。 ::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            $orConditions = $exp->or(['author_id' => 2])
                ->eq('author_id', 5);

            return $exp
                ->add($orConditions)
                ->eq('published', true)
                ->gte('view_count', 10);
        });

これは下記のような SQL を生成します。

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        (author_id = 2 OR author_id = 5)
        AND published = 1
        AND view_count >= 10
    )

**combinators** では、メソッドのチェーンを分離したい場合に、
新しい式オブジェクトをパラメータとして受け取るコールバックを渡すこともできます。 ::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            $orConditions = $exp->or(function (QueryExpression $or) {
                return $or->eq('author_id', 2)
                    ->eq('author_id', 5);
            });

            return $exp
                ->not($orConditions)
                ->lte('view_count', 10);
        });

``not()`` を使って式を否定することができます。 ::

    $query = $articles->find()
        ->where(function (QueryExpression $exp) {
            $orConditions = $exp->or(['author_id' => 2])
                ->eq('author_id', 5);

            return $exp
                ->not($orConditions)
                ->lte('view_count', 10);
        });

これは下記のような SQL を生成します。

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        NOT (author_id = 2 OR author_id = 5)
        AND view_count <= 10
    )

SQL 関数を使った式を構築することも可能です。 ::

    $query = $articles->find()
        ->where(function (QueryExpression $exp, Query $q) {
            $year = $q->func()->year([
                'created' => 'identifier'
            ]);

            return $exp
                ->gte($year, 2014)
                ->eq('published', true);
        });

これは下記のような SQL を生成します。

.. code-block:: sql

    SELECT *
    FROM articles
    WHERE (
        YEAR(created) >= 2014
        AND published = 1
    )

Expression オブジェクトを使う際、下記のメソッド使って条件を作成できます:

- ``eq()`` 等号の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->eq('population', '10000');
        });
    # WHERE population = 10000

- ``notEq()`` 不等号の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->notEq('population', '10000');
        });
    # WHERE population != 10000

- ``like()`` ``LIKE`` 演算子を使った条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->like('name', '%A%');
        });
    # WHERE name LIKE "%A%"

- ``notLike()`` ``LIKE`` 条件の否定を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->notLike('name', '%A%');
        });
    # WHERE name NOT LIKE "%A%"

- ``in()`` ``IN`` を使った条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->in('country_id', ['AFG', 'USA', 'EST']);
        });
    # WHERE country_id IN ('AFG', 'USA', 'EST')

- ``notIn()`` ``IN`` を使った条件の否定を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->notIn('country_id', ['AFG', 'USA', 'EST']);
        });
    # WHERE country_id NOT IN ('AFG', 'USA', 'EST')

- ``gt()`` ``>`` の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->gt('population', '10000');
        });
    # WHERE population > 10000

- ``gte()`` ``>=`` の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->gte('population', '10000');
        });
    # WHERE population >= 10000

- ``lt()`` ``<`` の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->lt('population', '10000');
        });
    # WHERE population < 10000

- ``lte()`` ``<=`` の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->lte('population', '10000');
        });
    # WHERE population <= 10000

- ``isNull()`` ``IS NULL`` の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->isNull('population');
        });
    # WHERE (population) IS NULL

- ``isNotNull()`` ``IS NULL`` の条件の否定を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->isNotNull('population');
        });
    # WHERE (population) IS NOT NULL

- ``between()`` ``BETWEEN`` の条件を作成します。 ::

    $query = $cities->find()
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->between('population', 999, 5000000);
        });
    # WHERE population BETWEEN 999 AND 5000000,

- ``exists()`` ``EXISTS`` を使用した条件を作成します。 ::

    $subquery = $cities->find()
        ->select(['id'])
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->equalFields('countries.id', 'cities.country_id');
        })
        ->andWhere(['population >' => 5000000]);

    $query = $countries->find()
        ->where(function (QueryExpression $exp, Query $q) use ($subquery) {
            return $exp->exists($subquery);
        });
    # WHERE EXISTS (SELECT id FROM cities WHERE countries.id = cities.country_id AND population > 5000000)

- ``notExists()`` ``EXISTS`` を使用した条件の否定を作成します。 ::

    $subquery = $cities->find()
        ->select(['id'])
        ->where(function (QueryExpression $exp, Query $q) {
            return $exp->equalFields('countries.id', 'cities.country_id');
        })
        ->andWhere(['population >' => 5000000]);

    $query = $countries->find()
        ->where(function ($exp, $q) use ($subquery) {
            return $exp->notExists($subquery);
        });
    # WHERE NOT EXISTS (SELECT id FROM cities WHERE countries.id = cities.country_id AND population > 5000000)

式オブジェクトは、多くの一般的に使用される関数や式をカバーしている必要があります。
式を使って必要な条件を作ることができない場合は、
``bind()`` を使ってパラメータを条件に手動でバインドすることができます。 ::

    $query = $cities->find()
        ->where([
            'start_date BETWEEN :start AND :end'
        ])
        ->bind(':start', '2014-01-01', 'date')
        ->bind(':end',   '2014-12-31', 'date');

あなたの望む条件を作成するビルダーメソッドが取得できなかったり利用したくない場合、
WHERE 句の中で SQL スニペットを使えるようにもできます。 ::

    // ２つのフィールドをお互いに比較
    $query->where(['Categories.parent_id != Parents.id']);

.. warning::

    式 (expression) の中で使われる列名や SQL スニペットには安全性が確実でない内容を
    **絶対に含めてはいけません** 。関数の呼び出しで、安全でないデータを安全に渡す
    方法については :ref:`using-sql-functions` のセクションを参照してください。

式の中で識別子を使用する
------------------------

クエリーの中でカラムや SQL 識別子を参照する必要がある場合、
``identifier()`` メソッドが使用できます。 ::

    $query = $countries->find();
    $query->select([
            'year' => $query->func()->year([$query->identifier('created')])
        ])
        ->where(function ($exp, $query) {
            return $exp->gt('population', 100000);
        });

.. warning::

    SQL インジェクションを防ぐため、識別子の式には信頼できないデータを渡すべきではありません。

IN 句を自動生成する
-------------------

ORM を使ってクエリーをビルドする際、大抵の場合、利用するカラムのデータ型を指定する必要はありません。
なぜなら CakePHP はスキーマデータに基いて推測することができるためです。
もしクエリーの中で CakePHP に自動的に等号を ``IN`` に変えさせたいなら、
カラムのデータ型を明示する必要があります。 ::

    $query = $articles->find()
        ->where(['id' => $ids], ['id' => 'integer[]']);

    // もしくは、自動的に配列へとキャストさせるために IN を含めます。
    $query = $articles->find()
        ->where(['id IN' => $ids]);

上記では自動的に ``id = ?`` ではなく ``id IN (...)`` が作成されます。
これは、パラメーターが単数か配列か判らない場合に便利です。データ型名の末尾に付く ``[]`` という接尾辞は、
扱いたいデータが配列であることをクエリービルダーに知らせます。
もしもデータが配列でなかったなら、まず、配列へとキャストされることになります。
その後、配列の各値は :ref:`type system <database-data-types>` を使ってキャストされることになります。
これは複合型であっても同様に動きます。たとえば、DateTime オブジェクトのリストも使うことができます。 ::

    $query = $articles->find()
        ->where(['post_date' => $dates], ['post_date' => 'date[]']);

IS NULL を自動生成する
----------------------

条件の値が ``null`` かもしれないし、他の値かもしれない場合、 ``IS`` 演算子を使うことで
自動的に正しい式が作成されます。 ::

    $query = $categories->find()
        ->where(['parent_id IS' => $parentId]);

上記は ``$parentId`` の型に応じて ``parent_id` = :c1`` もしくは ``parent_id IS NULL``
が自動的に作成されます。

IS NOT NULL を自動生成する
--------------------------

条件として非 ``null`` もしくは、他の値でないことを期待する場合、 ``IS NOT`` 演算子を使うことで
自動的に正しい式が作成されます。 ::

    $query = $categories->find()
        ->where(['parent_id IS NOT' => $parentId]);

上記は ``$parentId`` の型に応じて ``parent_id` != :c1`` もしくは ``parent_id IS NOT NULL``
が自動的に作成されます。


未加工の式
----------

クエリービルダーでは目的の SQL が構築できない場合、Expression オブジェクトを使って、
SQL の断片をクエリーに追加することができます。 ::

    $query = $articles->find();
    $expr = $query->newExpr()->add('1 + 1');
    $query->select(['two' => $expr]);

``Expression`` オブジェクトは ``where()`` 、 ``limit()`` 。 ``group()`` 、 ``select()``
等のようなクエリービルダーのメソッドで使うことができます。

.. warning::

    Expression オブジェクトを使うと SQL インジェクションに対して脆弱になります。
    式の中で信頼できないデータを使用しないでください。

結果を取得する
==============

クエリーができたら、それから行を受け取りたいでしょう。これにはいくつかの方法があります。 ::

    // クエリーをイテレートする
    foreach ($query as $row) {
        // なにかする
    }

    // 結果を取得する
    $results = $query->all();

Query オブジェクトでは :doc:`Collection </core-libraries/collections>`
のメソッドがどれでも使えます。それらで結果を前処理したり、変換したりすることができます。 ::

    // コレクションのメソッドを使う
    $ids = $query->map(function ($row) {
        return $row->id;
    });

    $maxAge = $query->max(function ($max) {
        return $max->age;
    });

``first`` や ``firstOrFail`` を使って、単一のレコードを受け取ることができます。
これらのメソッドはクエリーに ``LIMIT 1`` 句を付加した形に変更します。 ::

    // 最初の行だけを取得する
    $row = $query->first();

    // 最初の行を取得する。できないなら例外とする。
    $row = $query->firstOrFail();

.. _query-count:

レコードの合計数を返す
----------------------

Query オブジェクトを使って、条件の結果見つかった行の合計数を取得することができます。 ::

    $total = $articles->find()->where(['is_active' => true])->count();

``count()`` メソッドは ``limit``、 ``offset``、 ``page`` 句を無視します。
それゆえ、下記でも同じ結果を返すことになります。 ::

    $total = $articles->find()->where(['is_active' => true])->limit(10)->count();

これは、別の ``Query`` オブジェクトを構築する必要なく、結果セットの合計数を前もって知ることが
できるので便利です。同様に、結果のフォーマット (result formatting)、Map/Reduce 処理は
``count()`` を使う際には無視されます。

加えて言うと、group by 句を含んだクエリーの合計数を、クエリーを少しも書き直すことなく、
取得することが可能です。たとえば、記事 (article) の id と、そのコメント (comment) 件数を
取得するクエリーを考えてみましょう。 ::

    $query = $articles->find();
    $query->select(['Articles.id', $query->func()->count('Comments.id')])
        ->matching('Comments')
        ->group(['Articles.id']);
    $total = $query->count();

カウント後でも、結びついたレコードをフェッチするのにクエリーを使うことができます。 ::

    $list = $query->all();

ときには、クエリーの合計件数を返すメソッドをカスタマイズしたくなることもあるでしょう。
たとえば、値をキャッシュしたり、合計行数を見積もったり、あるいは、left join のような
高負荷な部分を取り除くようにクエリーを変更したりなどです。
CakePHP のページネーション (pagination) システムでは ``count()`` メソッドを呼び出しますので、
これは特に有用です。 ::

    $query = $query->where(['is_active' => true])->counter(function ($query) {
        return 100000;
    });
    $query->count(); // 100000 を返す

上記の例では、PaginatorComponent が count メソッドを呼ぶ際には、
ハードコーディングされた行数を受け取ることになります。

.. _caching-query-results:

ロードされた結果をキャッシュする
--------------------------------

変更されることのない Entity をフェッチする際、結果をキャッシュしたいと思うかもしれません。
``Query`` クラスでは簡単にこれを実現できます。 ::

    $query->cache('recent_articles');

これでクエリーの結果セットのキャッシュが有効になります。もし ``cache()`` に１つだけの引数を渡した場合は、
'default' のキャッシュ・コンフィグレーションが使われることになります。
第２引数でどのキャッシュ・コンフィグレーションを使うのかを制御できます。 ::

    // 文字列で Config 名
    $query->cache('recent_articles', 'dbResults');

    // CacheEngine のインスタンス
    $query->cache('recent_articles', $memcache);

``cache()`` メソッドは静的なキーだけでなく、 キーを生成する関数も受け取れます。
渡す関数は引数でクエリーを受け取りますので、クエリーの内容を読んで動的にキャッシュキーを
生成することができます。 ::

    // クエリーの where 句の単純なチェックサムに基づくキーを生成します
    $query->cache(function ($q) {
        return 'articles-' . md5(serialize($q->clause('where')));
    });

キャッシュメソッドはキャッシュされた結果をカスタム finder に渡したり、
イベントリスナで使ったりするのを簡単にします。

キャッシュ設定されたクエリーが結果を返すときには次のようなことが起こります:

1. クエリーが結果セットを保持しているなら、それを返します。
2. キャッシュキーを解決して、キャッシュデータを読みす。
   キャッシュデータが空でなければ、その結果を返します。
3. キャッシュが見つからない場合、クエリが実行され、 ``Model.beforeFind``
   イベントがトリガーされ、新しい ``ResultSet`` が作成されます。
   この ``ResultSet`` はキャッシュに書き込まれ、返されます。

.. note::

    ストリーミングクエリー (streaming query) の結果をキャッシュすることはできません。

関連付くデータをロードする
==========================

クエリービルダーは同時に複数テーブルからデータを取り出す際に、できるだけ最少のクエリーで取り出せるように
してくれます。関連付くデータをフェッチする際には、まず、:doc:`/orm/associations` のセクションに
あるようにテーブル間の関連をセットアップしてください。他のテーブルから関連するデータを
フェッチするためにクエリーを合成する技術を **イーガーロード** (eager load) といいます。

.. include:: ./retrieving-data-and-resultsets.rst
    :start-after: start-contain
    :end-before: end-contain

関連付くデータでフィルターする
------------------------------

.. include:: ./retrieving-data-and-resultsets.rst
    :start-after: start-filtering
    :end-before: end-filtering

.. _adding-joins:

Join を追加する
---------------

関連付くデータを ``contain()`` でロードすることもできますが、
追加の join をクエリービルダーに加えることもできます。 ::

    $query = $articles->find()
        ->join([
            'table' => 'comments',
            'alias' => 'c',
            'type' => 'LEFT',
            'conditions' => 'c.article_id = articles.id',
        ]);

複数 join の連想配列を渡すことで、複数の join を一度に追加できます。 ::

    $query = $articles->find()
        ->join([
            'c' => [
                'table' => 'comments',
                'type' => 'LEFT',
                'conditions' => 'c.article_id = articles.id',
            ],
            'u' => [
                'table' => 'users',
                'type' => 'INNER',
                'conditions' => 'u.id = articles.user_id',
            ]
        ]);

上記にあるように、join を加える際に、エイリアス (別名) を配列のキーで渡すことができます。
join の条件も条件の配列と同じように表現できます。 ::

    $query = $articles->find()
        ->join([
            'c' => [
                'table' => 'comments',
                'type' => 'LEFT',
                'conditions' => [
                    'c.created >' => new DateTime('-5 days'),
                    'c.moderated' => true,
                    'c.article_id = articles.id'
                ]
            ],
        ], ['c.created' => 'datetime', 'c.moderated' => 'boolean']);

手動で join を作成する際、配列による条件を使うなら、join 条件内の各列ごとにデータ型を渡す必要があります。
join 条件のデータ型を渡すことで、ORM はデータの型を SQL へと正しく変換できるのです。
join を作成する際には ``join()`` だけでなく、``rightJoin()``、 ``leftJoin()``、``innerJoin()``
を使うこともできます。 ::

    // エイリアスと文字列の条件で join する
    $query = $articles->find();
    $query->leftJoin(
        ['Authors' => 'authors'],
        ['Authors.id = Articles.author_id']);

    // エイリアスと配列の条件・型で join する
    $query = $articles->find();
    $query->innerJoin(
        ['Authors' => 'authors'],
        [
        'Authors.promoted' => true,
        'Authors.created' => new DateTime('-5 days'),
        'Authors.id = Articles.author_id'
        ],
        ['Authors.promoted' => 'boolean', 'Authors.created' => 'datetime']);

注意しなければならないのは、 ``Connection`` を定義する際に
``quoteIdentifiers`` オプションが ``true`` の場合には、
テーブルの列間の join 条件は次のようにしなければならないということです。 ::

    $query = $articles->find()
        ->join([
            'c' => [
                'table' => 'comments',
                'type' => 'LEFT',
                'conditions' => [
                    'c.article_id' => new \Cake\Database\Expression\IdentifierExpression('articles.id')
                ]
            ],
        ]);

これは Query 内のすべての識別子に引用符が付くことを保証し、いくつかのデータベースドライバ
（とくに PostgreSQL）でエラーが起こらないようにします。

データを insert する
=====================

前の例とは違って、insert するクエリーを作成するのに ``find()`` は使わないでください。
代わりに、 ``query()`` を使って新たな ``Query`` オブジェクトを作成します。 ::

    $query = $articles->query();
    $query->insert(['title', 'body'])
        ->values([
            'title' => 'First post',
            'body' => 'Some body text'
        ])
        ->execute();

１つのクエリーで複数の行を insert するために ``values()`` メソッドを、必要な回数分
つなげることができます。 ::

    $query = $articles->query();
    $query->insert(['title', 'body'])
        ->values([
            'title' => 'First post',
            'body' => 'Some body text'
        ])
        ->values([
            'title' => 'Second post',
            'body' => 'Another body text'
        ])
        ->execute();

通常は、エンティティーを使い、 :php:meth:`~Cake\\ORM\\Table::save()` でデータを
insert するほうが簡単です。また、``SELECT`` と ``INSERT`` を一緒に構築すれば、
``INSERT INTO ... SELECT`` スタイルのクエリーを作成することができます。 ::

    $select = $articles->find()
        ->select(['title', 'body', 'published'])
        ->where(['id' => 3]);

    $query = $articles->query()
        ->insert(['title', 'body', 'published'])
        ->values($select)
        ->execute();

.. note::
    クエリービルダーでレコードを insert すると、 ``Model.afterSave`` のような
    イベントは発生しません。代わりに :doc:`データ保存のための ORM </orm/saving-data>`
    が利用できます。

.. _query-builder-updating-data:

データを update する
====================

クエリーの insert と同様に、update のクエリーを作成するのに ``find()`` を使わないでください。
代わりに、 ``query()`` を使って新たな ``Query`` オブジェクトを作成します。 ::

    $query = $articles->query();
    $query->update()
        ->set(['published' => true])
        ->where(['id' => $id])
        ->execute();

通常は、エンティティーを使い、 :php:meth:`~Cake\\ORM\\Table::patchEntity()` でデータを
update するほうが簡単です。

.. note::
    クエリービルダーでレコードを update すると、 ``Model.afterSave`` のような
    イベントは発生しません。代わりに :doc:`データ保存のための ORM </orm/saving-data>`
    が利用できます。

データを delete する
====================

クエリーの insert と同様に、delete のクエリーを作成するのに ``find()`` を使わないでください。
代わりに、 ``query()`` を使って新たな クエリーオブジェクトを作成します::

    $query = $articles->query();
    $query->delete()
        ->where(['id' => $id])
        ->execute();

通常は、エンティティーを使い、 :php:meth:`~Cake\\ORM\\Table::delete()` でデータを
delete するほうが簡単です。

SQL インジェクションを防止する
==============================

ORM とデータベースの抽象層では、ほとんどの SQL インジェクション問題を防止してはいますが、
不適切な用法により危険な値が入り込む余地も依然としてありえます。

条件配列を使用している場合、キー/左辺および単一の値の入力は、ユーザーデータを含んではいけません。 ::

    $query->where([
        // キー/左辺のデータは、生成されたクエリーにそのまま挿入されるので、
        // 安全ではありません
        $userData => $value,

        // 単一の値の入力にも同じことが言えますが、
        // どの形式のユーザーデータでも安全に使用することはできません
        $userData,
        "MATCH (comment) AGAINST ($userData)",
        'created < NOW() - ' . $userData
    ]);

Expression ビルダーを使う際には、カラム名にユーザーデータを含めてはいけません。 ::

    $query->where(function (QueryExpression $exp) use ($userData, $values) {
        // いずれの式 (expression) の中であってもカラム名は安全ではありません
        return $exp->in($userData, $values);
    });

関数式を構築する際、関数名にユーザーデータを含めてはいけません。 ::

    // 安全ではありません
    $query->func()->{$userData}($arg1);

    // 関数式の引数としてユーザーデータの配列を使うことも安全ではありません
    $query->func()->coalesce($userData);

未加工 (raw) の式は安全ではありません。 ::

    $expr = $query->newExpr()->add($userData);
    $query->select(['two' => $expr]);

値のバインディング
-------------------

バインディングを使用することにより、多くの安全でない状況から保護することが可能です。
:ref:`プリペアードステートメントへの値のバインディング <database-basics-binding-values>`
と同様に、 :php:meth:`Cake\\Database\\Query::bind()` メソッドを使用して、
クエリーに値をバインドすることができます。

次の例は、上記の安全ではない SQL インジェクションが発生しやすい例を安全に変更したものです。 ::

    $query
        ->where([
            'MATCH (comment) AGAINST (:userData)',
            'created < NOW() - :moreUserData'
        ])
        ->bind(':userData', $userData, 'string')
        ->bind(':moreUserData', $moreUserData, 'datetime');

.. note::

    :php:meth:`Cake\\Database\\StatementInterface::bindValue()` と異なり、
    ``Query::bind()`` は、コロンを含む名前付けされたプレースホルダーを渡す必要があります!

より複雑なクエリー
===================

クエリービルダーでは ``UNION`` クエリーやサブクエリーのような複雑なクエリーも構築することができます。

UNION
------

UNION は１つ以上のクエリーを一緒に構築して作成します。 ::

    $inReview = $articles->find()
        ->where(['need_review' => true]);

    $unpublished = $articles->find()
        ->where(['published' => false]);

    $unpublished->union($inReview);

``unionAll()`` メソッドを使うことで ``UNION ALL`` クエリーを作成することもできます。 ::

    $inReview = $articles->find()
        ->where(['need_review' => true]);

    $unpublished = $articles->find()
        ->where(['published' => false]);

    $unpublished->unionAll($inReview);

サブクエリー
------------

サブクエリーはリレーショナル・データベースにおいて強力な機能であり、CakePHP ではそれを実に直感的に
構築することができます。クエリーを一緒に構築することで、サブクエリーを作ることができます。 ::

    $matchingComment = $articles->getAssociation('Comments')->find()
        ->select(['article_id'])
        ->distinct()
        ->where(['comment LIKE' => '%CakePHP%']);

    $query = $articles->find()
        ->where(['id IN' => $matchingComment]);

サブクエリーはクエリー式のどこにでも使うことができます。
たとえば、 ``select()`` や ``join()`` メソッドの中でもです。

ステートメントのロックの追加
----------------------------

ほとんどのリレーショナル・データベース製品は、SELECT 操作を行う際のロックをサポートします。
これは、 ``epilog()`` メソッドを使用することで可能です。 ::

    // MySQL の場合、
    $query->epilog('FOR UPDATE');

``epilog()`` メソッドは、クエリーの最後に生の SQL を追加することができます。
決して生のユーザーデータを ``epilog()`` にセットしないでください。

複雑なクエリーを実行する
------------------------

クエリービルダーはほとんどのクエリーを簡単に構築できるようにしてくれますが、
あまりに複雑なクエリーだと、構築するにも退屈で入り組んだものになるかもしれません。
:ref:`望む SQL を直接実行 <running-select-statements>` したいかもしれません。

SQL を直接実行するということは、走ることになるクエリーを微調整できることになります。
ただし、そうしてしまうと、 ``contain`` や他の高レベルな ORM 機能は使えません。
