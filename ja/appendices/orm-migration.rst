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

結果オブジェクトとして見ることが出来るクエリー、反復しようとするクエリー、
``toArray`` 呼び出しや他の  :ref:`collection <collection-objects>` から継承されたメソッド呼び出しは、
クエリーが実行されあなたに結果が返される時点で発生します。

CakePHP 2.xからの最も大きな相違点は、 ``find('first')`` はもう存在しないというこ
とで確認出来るでしょう。そのささいな代わりとして、  ``first()`` メソッドがあります::

    // 旧
    $article = $this->Article->find('first');

    // 新
    $article = $this->Articles->find()->first();

    // 旧
    $article = $this->Article->find('first', [
        'conditions' => ['author_id' => 1]
    ]);

    // 新
    $article = $this->Articles->find('all', [
        'conditions' => ['author_id' => 1]
    ])->first();

    // もしくはこうも書けます
    $article = $this->Articles->find()
        ->where(['author_id' => 1])
        ->first();

あなたがシングルレコードをプライマリーキーで取得する場合は、単に ``get()`` を呼
べば良いです::

    $article = $this->Articles->get(10);

ファインダーメソッドの変更点
---------------------

findメソッドからクエリーオブジェクトが返ることは、いくつかの利点が有りますが、
2.x.からの移行の手間が掛かります。もし、Modelにカスタムfindメソッドがある場合は、
それらの変更も必要になるでしょう。これは3.0におけるファインダーメソッドの作り方
です::

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

ご覧のとおり、とても単純明快です、配列の代わりにオブジェクトを使い、オブジェクト
で返します。カスタムファインダーにafterFindロジックを入れていた2.xユーザーは、
 :ref:`map-reduce` の章を参照して下さい、もしくは :ref:`collection-objects` の
機能を使って下さい。もしあなたのモデルにおいて、すべてのfind処理にafterFindを含む
のであれば、次のいずれかの方法で移行することができます:

1. エンティティーのconstructorメソッドをオーバーライドして、追加の書式設定をします。
2. バーチャルプロパティを作るため、エンティティーにaccessorメソッド作成します。
3.  ``findAll()`` を再定義し map/reduce 関数に結びつけます。

上の3番目の手法は次のようになるでしょう::

    public function findAll(Query $query, array $options)
    {
        $mapper = function ($row, $key, $mr) {
            // Your afterFind logic
        };
        return $query->mapReduce($mapper);
    }

あなたは、カスタムファインダーがoptions配列を受け取ることに気づいているかもしれま
せん、このパラメーターを使ってファインダーに追加情報を渡すことができます。これは
2.x.から移行する人にとって素晴らしいことです。旧バージョンで使っていた全てのクエ
リーキーは、3.xの正しい関数へ自動的に変換されるでしょう::

    // これは CakePHP 2.x and 3.0 の両方で動きます
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

願わくば、旧バージョンからの移行は、最初に思ったほど困難ではなく、私達が追加した
多くの機能が、あなたがコードを減らすのを助け、新しいORMを使って要件をうまく表現す
ることができ、同時に互換ラッパーが、迅速かつ痛みのない方法で、小さな相違点を書き
直すのを助けます。

ファインダーメソッド廻りの3.xの他の優れた改善点の1つは、ビヘイビアが難なくファイ
ンダーメソッドを実装できることです。ビヘイビア上に、一致する名前とシグネチャーを
持つメソッドを単純に定義することで、ファインダーは自動的に、ビヘイビアが接続され
ている全てのテーブル上で利用可能になります。

RecursiveとContainableBehaviorは削除しました
-----------------------------------------

旧バージョンのCakePHPににおいて、アソシエーションのセットにロードされたデータを減
らすために、あなたが興味を持っていた ``recursive``, ``bindModel()``, 
``unbindModel()`` および ``ContainableBehavior`` を使う必要がありました。アソシ
エーションを管理するための一般的な手法は、 ``recursive`` に ``-1`` をセットしたり、
全てのアソシエーションを管理するためには、Containableを使いました。 CakePHP 3.0に
おいて、ContainableBehavior, recursive, bindModel および unbindModelは全て削除
されています。代わりに ``contain()`` メソッドがクエリービルダーのコア機能に昇格
されました。アソシエーションは、明示的にオンになっている場合にだけ読み込まれます。
例えば::

    $query = $this->Articles->find('all');

この場合は、アソシエーションが含まれない場合は、 ``articles`` テーブルから ** のみ **
読み込まれます。記事とその関連作者を読み込むためには、次のようになるでしょう::

    $query = $this->Articles->find('all')->contain(['Authors']);

明確に要求されたアソシエーションデータだけを読み込むことによって、必要なデーター
だけを取得しようとするので、ORMと格闘するのに費やすのは僅かな時間です。

afterFindイベントとバーチャルフィールドは無い
------------------------------------

旧バージョンのCakePHPにおいて、あなたは、任意のデータープロパティをするために
``afterFind`` コールバックとバーチャルフィールドを広く利用する必要があった。
これらの機能は、3.0において削除されました。ResultSetsがエンティティーを反復的に
構築する方法は、 ``afterFind`` のコールバックでは不可能であるためです。
afterFindとバーチャルフィールドの両方共に、エンティティーのバーチャルプロパティ
へ大々的に置き換えることができました。例えば、あなたのUserエンティティーが、姓と名
の両方のカラムを持つなら、 `full_name` 用のaccessorを追加することができ、その場で
生成できます::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class User extends Entity
    {
        protected function _getFullName()
        {
            return $this->first_name . '  ' . $this->last_name;
        }
    }

一度定義すれば、あなたは、 ``$user->full_name`` を使って新しいプロパティにアクセ
スできます。ORMの :ref:`map-reduce` 機能を使うと、集約されたデーターを結果から構
築できるようになります、これは、 ``afterFind`` コールバックの使い道として多く使わ
れた方法でした。

一方バーチャルフィールドはORMの明示的な機能ではなくなりました、ファインダーメソッ
ドにおいてcalculatedフィールドを追加することが簡単にできます。クエリービルダーと
表現オブジェクトを使うことによって、バーチャルフィールドと同じ結果に到達できます::

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

関連付けはプロパティに定義されなくなくなりました。
--------------------------------------------

旧バージョンのCakePHPにおいて、モデルが持っていた様々な関連付けは、
``$belongsTo`` や ``$hasMany`` のようなプロパティにて定義されていました。
CakePHP 3.0において、関連付はメソッドとして構築されました。メソッドを使うことは
クラス定義による多くの制限を回避できるようにします、関連付けのためのただひとつだ
けの方法を提供します。 ``initialize()`` メソッドと、あなたの他全てのアプリケー
ションコードは、アソシエーションを操作する際に、同じAPIを対話します::
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

ご覧のように上の例は、アソシエーションのそれぞれのタイプは、アソシエーションを
構築するためのメソッドとして使います。もう一つの違いは、``hasAndBelongsToMany`` 
が ``belongsToMany`` にリネームされたことです。3.0のアソシエーション構築について
もっと知りたい場合は、 :doc:`/orm/associations` セクションを参照して下さい。

CakePHPのもう一つの歓迎すべき向上点は、自作のアソシエーションクラスを構築出来る
ことです。ビルトインのリレーションタイプがカバーしていないアソシエーションタイプ
がある場合、あなたはカスタム ``Association`` サブクラスを構築することができ、
あなたが作りたいロジックのアソシエーションを定義できます。

バリデーションはプロパティ定義ではなくなった
------------------------------------------

アソシエーションのように、バリデーションルールは旧バージョンのCakePHPにおいては
クラスのプロパティとして定義されていました。そしてこの配列は ``ModelValidator`` 
オブジェクトにすっかり姿を変えました。この改変のステップは、実行時の複雑なルール
を変更する間接のレイヤーを追加しました。さらに、バリデーションルールは、プロパティ
として定義されていたことで、複数セットのバリデーション持つモデルを難しくしていま
した。CakePHP 3.0 はこの両方の問題を改善しています。バリデーションルールは常に
 ``Validator`` オブジェクトとして構築します、それは複数セットのルールをもつことを
 簡単にします::

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

あなたは、多くのバリデーションルールを思い通りに定義できます。それぞれのメソッドは
``validation`` プレフィックスを持ち、 ``$validator`` 引数を許可します。

旧バージョンのCakePHPにおいて'validation'と関連するコールバックは、いくつかの関連
付をカバーしましたが、使い道が違います。CakePHP 3.0 において、正式なバリデー
ションは、２つのコンセプトに分かれます:

#. データータイプおよび形式の検証。
#. アプリケーションとビジネスルールへの施行

バリデーションは、リクエストデーターから作られたORMエンティティーの前に追加され
ます。このステップは、データータイプ、形式、アプリケーションが期待する基本形を
保証します。あなたは、リクエストデーターをエンティティに変換する際に、
``validate`` オプションを使うことで、バリデーターを使うことができます。詳しくは
:ref:`converting-request-data` のドキュメントを参照して下さい。

:ref:`Application rules <application-rules>` は、アプリケーションのルールと状態
およびワークフローの施行を保証する、ルールの定義を許可します。ルールはテーブルの
 ``buildRules()`` メソッドにてい定義されます。ビヘイビアはe ``buildRules()`` 
フックメソッドを使ってルールを追加できます。我々の articlesテーブルにおける
``buildRules()`` メソッドの例は、次のようになります::

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

デフォルトで識別子のクウォート無効
--------------------------------------

旧バージョンのCakePHPにおいては、常に識別子はクウォートされていました。SQLスニ
ペットの解析と識別子のクウォートをしようとする
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
