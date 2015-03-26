新しいORMアップグレードガイド
#####################

CakePHPの3.0は、始めから書きなおされた新しいORMを備えています。
1.xと2.x でずっと慣れ親しんできたORMには、いくつかの改善したい項目がありました。

* フランケンシュタイン - レコードか、テーブルか？現状は両方です。
* 一貫性のないAPI - Model::read() の様な。
* オブジェクトでないクエリー - クエリーが配列で定義されることにより、いくつかの
  制限や制約があります。例えば、ユニオンやサブクエリーを定義することがとても
  難しいです。
* 返り値が配列。これは、CakePHPにおける共通の不満で、おそらくいくつかの水準で
  採用を妨げていました。
* オブジェクトでないレコード - これは関連付け書式設定を、困難/不可能にしています。
* Containable - はORMの一部であるべきで、クレイジーなビヘイビアで行うべきで
  はありません。
* Recursive - これは関連付けを含む定義として、きちんと制御されるべきで、再帰性
  のレベルで制御するべきではありません。
* DboSource - それは獣だ。Modelはデーターソースよりそれに依存する。それらの分離
  は、もっとクリーンで簡単な可能性があります。
* Validation - は分離するべきです、現状はバカでかい関数になっています。少し
  再利用性をつけると、フレームワークはもっと拡張性を持ちます。

CakePHP 3.0のORMは、これらの項目とより多くの問題を解決します。今回、新しいORMは
リレーショナルデータストアに注力しています。将来的には、プラグインを通して、
我々はElasticSearchなどのような非リレーショナルストアを追加します。

新しいORMのデザイン
=====================

新しいORMは、いくつかの、より特化された、フォーカスされたクラスの問題を解決します。
かつては、``Model`` やDatasourceをすべてのオペレーションで使っていたでしょう。
今回、ORMはより多くのレイヤーに分離しました:

* ``Cake\Database\Connection`` - コネクションを生成し利用する方法の革命的な
  プラットフォームを提供します。このクラスは、トランザクション、クエリーの実行
  およびスキーマデータへのアクセスを提供します。
* ``Cake\Database\Dialect`` - この名前空間内のクラスは、プラットフォーム特化
  SQLの提供と、プラットフォーム特化の制限廻りで動作するクエリーの変換をします。
* ``Cake\Database\Type`` - CakePHPのデーターベースタイプ変換システムのゲート
  ウェイクラスです。これは着脱可能なフォレームワークで、抽象的カラムタイプの
  追加と、データベース、PHP表現とデータータイプのためのPDO結合間のマッピングを
  提供します。例えば、datetimeカラムは、コード中において``DateTime`インスタンス
  として表現されます。
* ``Cake\ORM\Table`` - 新しいORMのメインな導入ポイントです。シングルテーブルへの
  アクセスを提供します。アソシエーション定義、ビヘイビアの利用とエンティティの
  生成ならびに、クエリーオブジェクトを扱います。
* ``Cake\ORM\Behavior`` - ビヘイビアのベースクラスです、旧バージョンのCakePHP
  のビヘイビアと同等に動作します。
* ``Cake\ORM\Query`` - クエリービルダーベースのfluentオブジェクトです、旧
  バージョンのCakePHPにおいて、深くネストされた配列であったのもに置き換わる
  オブジェクトです。
* ``Cake\ORM\ResultSet`` - 結果のコレクションです、集計データの操作において
  強力なツールを提供します。
* ``Cake\ORM\Entity`` - シングルレコードの結果を生成します。データーへのアクセス
  と、snapを様々なフォーマットにシリアライズします。

ここで、あなたはいくつかのクラスにより精通します、新しいORMにおいて最も頻繁に相互
作用する、もっとも重要な3つのクラスについて見ていきます。``Table``、 ``Query``
および``Entity``クラスは、新しいORMにおいて多くの力仕事を行い、それぞれ異なる役目
があります。

テーブルオブジェクト
-------------

テーブルオブジェクトは、データーへのゲートウェイです。それらは、旧バージョンの
``Model``が行っていた多くのタスクを扱います。テーブルクラスは、以下の様なタスク
を扱います:

- クエリーの生成。
- ファインダーの提供。
- エンティティーのバリデートとセーブ。
- エンティティーの削除。
- アソシエーションの定義とアクセス。
- コールバックイベントのトリガー。
- ビヘイビアとの対話。

ドキュメントのチャプター :doc:`/orm/table-objects` では、テーブルオブジェクトに
ついて本ガイドより詳細な使い方を解説しています。一般的には、既存のモデルコード
を移動する場合は、テーブルオブジェクトになるでしょう。テーブルオブジェクトは、
プラットフォームに依存SQLは含みません。代わりに、エンティティーとクエリービル
ダーと併せて使用します。テーブルオブジェクトは、公開されたイベントを通じて、
ビヘイビアとその他関連処理とも相互作用します。

クエリーオブジェクト
-------------

これらはあなた自身を構築するクラスはありませんが、あなたのアプリケーション
コードは新しいORMの中心であるクエリービルダーを広範囲に使用していきます。
クエリービルダーは、単純クエリーも複雑なクエリー構築も、簡単にします、それは
以前はCakePHPにおいて非常に難しかった ``HAVING``、``UNION``およびサブクエリー
も含みます。

あなたのアプリケーションを呼ぶ様々なfind()は、現状は、新しいクエリービ
ルダーを使う様アップデートする必要があるでしょう。クエリーオブジェクトは、
クエリーを実行する以外の、クエリーを構築するデーターを含めて責任を持ちます。
これは、プラットフォームの出力として`` ResultSet``の作成が実行される特定のSQLを
生成するために、接続/方言とのコラボレーション。

エンティティオブジェクト
--------------

旧バージョンのCakePHPにおいて、 ``Model`` クラスは、ロジックやビヘイビアを含まな
い様な、ダメな配列を返しました。一方でコミュニティは、CakeEntityのようなプロジェ
クトにより、この欠点を致命的でないものにしました、配列の返り値は、しばしば多くの
開発者のトラブルの原因となる欠点でした。CakePHP 3.0のために、明示的に無効にしない
限りは常に、ORMはオブジェクトのリザルトセットを返します。 :doc:`/orm/entities` の
章は、あなたがエンティティで到達できる様々なタスクをカバーします。

エンティティは次のいずれかの方法により作られました。それは、データーベースから
データーをロードするか、リクエストデーターをエンティティに変換するかです。
一度構築されると、エンティティはあなたに、データーの操作を許可します、それらの
データーは、テーブルオブジェクト連携しデーターを含み持ち続けます。

主な相違点
===============

新しいORMは既存の ``Model`` レイヤーから大きく逸脱しています、新しいORMのオペ
レーションやあなたのコードをアップデートする方法を理解するための、多くの重要な
相違点があります。

語尾変化ルールのアップデート
------------------------

あなたは、テーブルクラスが複数形の名前を持つことに気づいていたかも知れません。
テーブルが複数形の名前を持つことに加えて、アソシエーションも複数形で呼ばれます。
クラス名とアソシエーションエイリアスが単数形であったことは、 ``Model`` とは
対照的です。この変更には次のような理由があります:

* テーブルクラスは **collections** のデータとして表現されます、単一行ではなく。
* アソシエーションリンクテーブル同士は、多くのものとのリレーションを記述している。

テーブルオブジェクトの表記は複数形である一方で、エンティティアソシエーションプロ
パティは、アソシエーションタイプに基づいて取り込まれます。

.. note::

    BelongsToとHasOneアソシエーションは、 エンティティプロパティにおいて単数形を
    使い、HasManyとBelongsToMany (HABTM)は複数形を使います。 

テーブルオブジェクトの表記変更は、クエリー構築時は最も明らかです。
クエリーを次のように表記する代わりに::

    // 誤
    $query->where(['User.active' => 1]);

あなたは複数形を使う必要があります::

    // 正
    $query->where(['Users.active' => 1]);

クエリーオブジェクトを返すFind
---------------------------

新しいORMの1つの大きな違いは、テーブルに ``find`` 呼び出しをしてもすぐには結果を
返さないことです、結果はクエリーオブジェクトとして返します; これにはいくつかの
目的があります。

これは、findを呼び出した後に、さらにクエリを変更することを可能にします::

    $articles = TableRegistry::get('Articles');
    $query = $articles->find();
    $query->where(['author_id' => 1])->order(['title' => 'DESC']);

それは、条件、ソート、制限、そのほか色々な句を追加するためのカスタムファインダー
を、同じクエリーに実行前にスタックすることを可能にします::

    $query = $articles->find('approved')->find('popular');
    $query->find('latest');

あなたは、より簡単にサブクエリーを作るためのクエリーを構成することが出来ます::

    $query = $articles->find('approved');
    $favoritesQuery = $article->find('favorites', ['for' => $user]);
    $query->where(['id' => $favoritesQuery->select(['id'])]);

あなたは、データベースに触れることなく、イテレータとメソッド呼び出しを持つクエリ
を作ることができます、これはキャッシュされたビューのパーツがあると大いに役立ちま
す、データベースから取得する結果は、実際には必要としません::

    // この例ではクエリーが構築されない!
    $results = $articles->find()
        ->order(['title' => 'DESC'])
        ->formatResults(function ($results) {
            return $results->extract('title');
        });

Queries can be seen as the result object, trying to iterate the query, calling
``toArray`` or any method inherited from :ref:`collection <collection-objects>`,
will result in the query being executed and results returned to you.

The biggest difference you will find when coming from CakePHP 2.x is that
``find('first')`` does not exist anymore. There is a trivial replacement for it,
and it is the ``first()`` method::

    // Before
    $article = $this->Article->find('first');

    // Now
    $article = $this->Articles->find()->first();

    // Before
    $article = $this->Article->find('first', [
        'conditions' => ['author_id' => 1]
    ]);

    // Now
    $article = $this->Articles->find('all', [
        'conditions' => ['author_id' => 1]
    ])->first();

    // Can also be written
    $article = $this->Articles->find()
        ->where(['author_id' => 1])
        ->first();

If you are a loading a single record by its primary key, it will be better to
just call ``get()``::

    $article = $this->Articles->get(10);

Finder Method Changes
---------------------

Returning a query object from a find method has several advantages, but comes at
a cost for people migrating from 2.x. If you had some custom find methods in
your models, they will need some modifications. This is how you create custom
finder methods in 3.0::

    class ArticlesTable
    {

        public function findPopular(Query $query, array $options)
        {
            return $query->where(['times_viewed' > 1000]);
        }

        public function findFavorites(Query $query, array $options)
        {
            $for = $options['for'];
            return $query->matching('Users.Favorites', function ($q) use ($for) {
                return $q->where(['Favorites.user_id' => $for]);
            });
        }
    }

As you can see, they are pretty straightforward, they get a Query object instead
of an array and must return a Query object back. For 2.x users that implemented
afterFind logic in custom finders, you should check out the :ref:`map-reduce`
section, or use the features found on the :ref:`collection-objects`. If in your
models you used to rely on having an afterFind for all find operations you can
migrate this code in one of a few ways:

1. Override your entity constructor method and do additional formatting there.
2. Create accessor methods in your entity to create the virtual properties.
3. Redefine ``findAll()`` and attach a map/reduce function.

In the 3rd case above your code would look like::

    public function findAll(Query $query, array $options)
    {
        $mapper = function ($row, $key, $mr) {
            // Your afterFind logic
        };
        return $query->mapReduce($mapper);
    }

You may have noticed that custom finders receive an options array, you can pass
any extra information to your finder using this parameter. This is great
news for people migrating from 2.x. Any of the query keys that were used in
previous versions will be converted automatically for you in 3.x to the correct
functions::

    // This works in both CakePHP 2.x and 3.0
    $articles = $this->Articles->find('all', [
        'fields' => ['id', 'title'],
        'conditions' => [
            'OR' => ['title' => 'Cake', 'author_id' => 1],
            'published' => true
        ],
        'contain' => ['Authors'], // The only change! (notice plural)
        'order' => ['title' => 'DESC'],
        'limit' => 10,
    ]);

Hopefully, migrating from older versions is not as daunting as it first seems,
much of the features we have added helps you remove code as you can better
express your requirements using the new ORM and at the same time the
compatibility wrappers will help you rewrite those tiny differences in a fast
and painless way.

One of the other nice improvements in 3.x around finder methods is that
behaviors can implement finder methods with no fuss. By simply defining a method
with a matching name and signature on a Behavior the finder will automatically
be available on any tables the behavior is attached to.

Recursive and ContainableBehavior Removed
-----------------------------------------

In previous versions of CakePHP you needed to use ``recursive``,
``bindModel()``, ``unbindModel()`` and ``ContainableBehavior`` to reduce the
loaded data to the set of associations you were interested in. A common tactic
to manage associations was to set ``recursive`` to ``-1`` and use Containable to
manage all associations. In CakePHP 3.0 ContainableBehavior, recursive,
bindModel, and unbindModel have all been removed. Instead the ``contain()``
method has been promoted to be a core feature of the query builder. Associations
are only loaded if they are explicitly turned on. For example::

    $query = $this->Articles->find('all');

Will **only** load data from the ``articles`` table as no associations have been
included. To load articles and their related authors you would do::

    $query = $this->Articles->find('all')->contain(['Authors']);

By only loading associated data that has been specifically requested you spend
less time fighting the ORM trying to get only the data you want.

No afterFind Event or Virtual Fields
------------------------------------

In previous versions of CakePHP you needed to make extensive use of the
``afterFind`` callback and virtual fields in order to create generated data
properties. These features have been removed in 3.0. Because of how ResultSets
iteratively generate entities, the ``afterFind`` callback was not possible.
Both afterFind and virtual fields can largely be replaced with virtual
properies on entities. For example if your User entity has both first and last
name columns you can add an accessor for `full_name` and generate the property
on the fly::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class User extends Entity
    {
        protected function _getFullName()
        {
            return $this->first_name . '  ' . $this->last_name;
        }
    }

Once defined you can access your new property using ``$user->full_name``.
Using the :ref:`map-reduce` features of the ORM allow you to build aggregated
data from your results, which is another use case that the ``afterFind``
callback was often used for.

While virtual fields are no longer an explicit feature of the ORM, adding
calculated fields is easy to do in your finder methods. By using the query
builder and expression objects you can achieve the same results that virtual
fields gave::

    namespace App\Model\Table;

    use Cake\ORM\Table;
    use Cake\ORM\Query;

    class ReviewsTable extends Table
    {
        public function findAverage(Query $query, array $options = [])
        {
            $avg = $query->func()->avg('rating');
            $query->select(['average' => $avg]);
            return $query;
        }
    }

Associations No Longer Defined as Properties
--------------------------------------------

In previous versions of CakePHP the various associations your models had were
defined in properties like ``$belongsTo`` and ``$hasMany``. In CakePHP 3.0,
associations are created with methods. Using methods allows us to sidestep the
many limitations class definitions have, and provide only one way to define
associations. Your ``initialize()`` method and all other parts of your application
code, interact with the same API when manipulating associations::

    namespace App\Model\Table;

    use Cake\ORM\Table;
    use Cake\ORM\Query;

    class ReviewsTable extends Table
    {

        public function initialize(array $config)
        {
            $this->belongsTo('Movies');
            $this->hasOne('Ratings');
            $this->hasMany('Comments')
            $this->belongsToMany('Tags')
        }

    }

As you can see from the example above each of the association types uses
a method to create the association. One other difference is that
``hasAndBelongsToMany`` has been renamed to ``belongsToMany``. To find out more
about creating associations in 3.0 see the section on :doc:`/orm/associations`.

Another welcome improvement to CakePHP is the ability to create your own
association classes. If you have association types that are not covered by the
built-in relation types you can create a custom ``Association`` sub-class and
define the association logic you need.

Validation No Longer Defined as a Property
------------------------------------------

Like associations, validation rules were defined as a class property in previous
versions of CakePHP. This array would then be lazily transformed into
a ``ModelValidator`` object. This transformation step added a layer of
indirection, complicating rule changes at runtime. Futhermore, validation rules
being defined as a property made it difficult for a model to have multiple sets
of validation rules. In CakePHP 3.0, both these problems have been remedied.
Validation rules are always built with a ``Validator`` object, and it is trivial
to have multiple sets of rules::

    namespace App\Model\Table;

    use Cake\ORM\Table;
    use Cake\ORM\Query;
    use Cake\Validation\Validator;

    class ReviewsTable extends Table
    {

        public function validationDefault(Validator $validator)
        {
            $validator->requirePresence('body')
                ->add('body', 'length', [
                    'rule' => ['minLength', 20],
                    'message' => 'Reviews must be 20 characters or more',
                ])
                ->add('user_id', 'numeric', [
                    'rule' => 'numeric'
                ]);
            return $validator;
        }

    }

You can define as many validation methods as you need. Each method should be
prefixed with ``validation`` and accept a ``$validator`` argument.

In previous versions of CakePHP 'validation' and the related callbacks covered
a few related but different uses. In CakePHP 3.0, what was formerly called
validation is now split into two concepts:

#. Data type and format validation.
#. Enforcing application, or business rules.

Validation is now applied before ORM entities are created from request data.
This step lets you ensure data matches the data type, format, and basic shape
your application expects. You can use your validators when converting request
data into entities by using the ``validate`` option. See the documentation on
:ref:`converting-request-data` for more information.

:ref:`Application rules <application-rules>` allow you to define rules that
ensure your application's rules, state and workflows are enforced. Rules are
defined in your Table's ``buildRules()`` method. Behaviors can add rules using
the ``buildRules()`` hook method. An example ``buildRules()`` method for our
articles table could be::

    // In src/Model/Table/ArticlesTable.php
    namespace App\Model\Table;

    use Cake\ORM\Table;
    use Cake\ORM\RulesChecker;

    class Articles extends Table
    {
        public function buildRules(RulesChecker $rules)
        {
            $rules->add($rules->existsIn('user_id', 'Users'));
            $rules->add(
                function ($article, $options) {
                    return ($article->published && empty($article->reviewer));
                },
                'isReviewed',
                [
                    'errorField' => 'published',
                    'message' => 'Articles must be reviewed before publishing.'
                ]
            );
            return $rules;
        }
    }

Identifier Quoting Disabled by Default
--------------------------------------

In the past CakePHP has always quoted identifiers. Parsing SQL snippets and
attempting to quote identifiers was both error prone and expensive. If you are
following the conventions CakePHP sets out, the cost of identifier quoting far
outweighs any benefit it provides. Because of this identifier quoting has been
disabled by default in 3.0. You should only need to enable identifier quoting if
you are using column names or table names that contain special characters or are
reserved words. If required, you can enable identifier quoting when configuring
a connection::

    // In config/app.php
    'Datasources' => [
        'default' => [
            'className' => 'Cake\Database\Driver\Mysql',
            'username' => 'root',
            'password' => 'super_secret',
            'host' => 'localhost',
            'database' => 'cakephp',
            'quoteIdentifiers' => true
        ]
    ],

.. note::

    Identifiers in ``QueryExpression`` objects will not be quoted, and you will
    need to quote them manually or use IdentifierExpression objects.

Updating Behaviors
==================

Like most ORM related features, behaviors have changed in 3.0 as well. They now
attach to ``Table`` instances which are the conceptual descendent of the
``Model`` class in previous versions of CakePHP. There are a few key
differences from behaviors in CakePHP 2.x:

- Behaviors are no longer shared across multiple tables. This means you no
  longer have to 'namespace' settings stored in a behavior. Each table using
  a behavior will get its own instance.
- The method signatures for mixin methods have changed.
- The method signatures for callback methods have changed.
- The base class for behaviors have changed.
- Behaviors can easily add finder methods.

New Base Class
--------------

The base class for behaviors has changed. Behaviors should now extend
``Cake\ORM\Behavior``; if a behavior does not extend this class an exception
will be raised. In addition to the base class changing, the constructor for
behaviors has been modified, and the ``startup()`` method has been removed.
Behaviors that need access to the table they are attached to should define
a constructor::

    namespace App\Model\Behavior;

    use Cake\ORM\Behavior;

    class SluggableBehavior extends Behavior
    {

        protected $_table;

        public function __construct(Table $table, array $config)
        {
            parent::__construct($table, $config);
            $this->_table = $table;
        }

    }

Mixin Methods Signature Changes
-------------------------------

Behaviors continue to offer the ability to add 'mixin' methods to Table objects,
however the method signature for these methods has changed. In CakePHP 3.0,
behavior mixin methods can expect the **same** arguments provided to the table
'method'. For example::

    // Assume table has a slug() method provided by a behavior.
    $table->slug($someValue);

The behavior providing the ``slug()`` method will receive only 1 argument, and its
method signature should look like::

    public function slug($value)
    {
        // Code here.
    }

Callback Method Signature Changes
---------------------------------

Behavior callbacks have been unified with all other listener methods. Instead of
their previous arguments, they need to expect an event object as their first
argument::

    public function beforeFind(Event $event, Query $query, array $options)
    {
        // Code.
    }

See :ref:`table-callbacks` for the signatures of all the callbacks a behavior
can subscribe to.
