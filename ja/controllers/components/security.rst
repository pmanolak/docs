セキュリティ
############

.. php:class:: SecurityComponent(ComponentCollection $collection, array $config = [])

Security コンポーネントを使うと、アプリケーションにさらに堅牢なセキュリティを導入できます。
このコンポーネントは、以下の様々なタスクのためのメソッドを提供します。

* アプリケーションが受付ける HTTP メソッドの限定
* フォーム改ざん防止
* SSL の利用を要求
* コントローラー間の通信制限

全てのコンポーネントと同様に、いくつかの設定できるパラメーターがあり、
これら全てのプロパティーは、直接設定したり、コントローラーの ``beforeFilter()`` の中で、
プロパティーと同じ名前のセッターメソッドで設定できます。

Security コンポーネントを使用することで、自動的にフォーム改ざんを 防止します。
Security コンポーネントによって、hidden トークンフィールドがフォームに追加され、
チェックされます。

Security コンポーネントのフォーム保護機能と、 ``startup()`` コールバックで
フォームデータを処理する他のコンポーネントを使用している場合、 ``initialize()`` メソッドの中で、
それらのコンポーネントの前に Security コンポーネントを配置するようにしてください。

.. note::

    Security コンポーネントを使用している時、フォームの作成に FormHelper を
    **使わなければなりません** 。また、フィールドの "name" 属性を
    **上書きしてはいけません** 。Security コンポーネントは、FormHelper で作成され、
    管理されるインジケーターを確認します。(これらは
    :php:meth:`~Cake\\View\\Helper\\FormHelper::create()` と
    :php:meth:`~Cake\\View\\Helper\\FormHelper::end()` の中で作成されます。)
    例えば JavaScript で無効化・ 削除・新規作成するなどして POST リクエスト中の投稿された
    フィールドを動的に差し替えることは、リクエストが破棄されるきっかけになります。

    副作用を避けるために、実行前に使用されている HTTP メソッドを常に確認する必要があります。
    正しい HTTP メソッドが使用されていることを確認するために、
    :ref:`HTTP メソッドをチェックする <check-the-request>` か、
    :php:meth:`~Cake\\Http\\ServerRequest::allowMethod()` を使用してください。

ブラックホールコールバックの処理
================================

.. php:method:: blackHole(Controller $controller, string $error = '', ?SecurityException $exception = null)

あるアクションが Security コンポーネントによって制限されている時、
デフォルトでは、不正なリクエストとして 400 エラーを返し破棄します。
コントローラー中のコールバック関数を ``blackHoleCallback`` 設定オプションに
設定することによってこの振る舞いを変更できます。

コールバックメソッドを設定することによって、ブラックホール処理がどのように機能するかを
カスタマイズすることができます。 ::

    public function beforeFilter(EventInterface $event)
    {
        parent::beforeFilter($event);

        $this->Security->setConfig('blackHoleCallback', 'blackhole');
    }

    public function blackhole($type, SecurityException $exception)
    {
        if ($exception->getMessage() === 'リクエストがSSLではありません。アクションは安全である必要があります。') {
            // 例外メッセージを翻訳可能な文字列に書き換えます。
            $exception->setMessage(__('要求されたページにHTTPSでアクセスしてください'));
        }

        // 条件付きでリワードされた例外を再度スローします。
        throw $exception;

        // または、フラッシュメッセージを設定して、
        // 要求されたページの HTTPS バージョンにリダイレクトするなどして
        // エラーを処理することもできます。
    }

``$type`` パラメーターは、以下の値を指定できます。

* 'auth' は、フォームバリデーションエラー、もしくはコントローラー/アクションの不適合エラーを示します。
* 'secure' は、SSL メソッド制限の失敗を示します。

アクションを SSL 通信に限定
===========================

この機能は :ref:`https-enforcer-middleware` の登場に伴い削除されました。

フォーム改ざん防止
==================

デフォルトでは、 ``SecurityComponent`` は、ユーザーが特定の方法でフォームを改変することを
防ぎます。 ``SecurityComponent`` は、以下のことを防止します。

* フォームに新規フィールドを追加することはできません。
* フォームからフィールドを削除することはできません。
* hidden フィールドの値を更新することはできません。

これらの改変を防ぐには、 ``FormHelper`` を使って、フォーム内のフィールドを追跡することで実現します。
その際、 hidden フィールドの値も追跡対象になります。このデータの全てが結合され、
ハッシュに変換されます。フォームが投稿された時、 ``SecurityComponent`` は、
POST データをもとに同じ構造を構築し、ハッシュと比較します。

.. note::

    SecurityComponent は、セレクトオプションの追加や変更を防ぐことは **できません** 。
    また、ラジオオプションも追加や変更を防ぐことはできません。

unlockedFields
    POST バリデーションを解除したいフォームフィールドの一覧をセットします。
    このコンポーネントの他にも、 :php:meth:`FormHelper::unlockField()`
    でも解除できます。制限が解除されたフィールドは、POST 時に必須ではなくなり、
    hidden フィールドの値もチェックされません。

validatePost
    ``false`` をセットすると、POST リクエストのバリデーションを完全にスキップし、
    実質フォームバリデーションを無効化します。

使い方
======

Security コンポーネントは、一般的にコントローラーの
``initialize`` または ``beforeFilter()`` のコールバックの中で行われます。 ::

    namespace App\Controller;

    use App\Controller\AppController;
    use Cake\Event\EventInterface;

    class WidgetsController extends AppController
    {
        public function initialize(): void
        {
            parent::initialize();
            $this->loadComponent('Security');
        }

        public function beforeFilter(EventInterface $event)
        {
            parent::beforeFilter($event);

            if ($this->request->getParam('prefix') === 'Admin') {
                $this->Security->setConfig('validatePost', false);
            }
        }
    }

上記の例では、 管理者用ルーティングの全てのアクションは、セキュアな SSL 通信のみを許可します。

.. _security-csrf:

CSRF 防御
=========

CSRF つまり、クロスサイトリクエストフォージェリ (Cross Site Request Forgery) は、
アプリケーションの一般的な脆弱性です。攻撃者が、直前のリクエストを記録し再生することを許し、
他のドメイン上の画像タグやリソースを使用してデータを送信します。
CSRF 保護機能を有効にするには、:ref:`csrf-middleware` を利用してください。

指定したアクションの Security コンポーネントの無効化
====================================================

例えば AJAX リクエストなど、あるアクションで全てのセキュリティチェックを無効化したい場合があります。
``beforeFilter()`` 内で ``$this->Security->unlockedActions`` にリストアップすることで
これらのアクションを「アンロック」できます。 ::

    namespace App\Controller;

    use App\Controller\AppController;
    use Cake\Event\EventInterface;

    class WidgetController extends AppController
    {
        public function initialize(): void
        {
            parent::initialize();
            $this->loadComponent('Security');
        }

        public function beforeFilter(EventInterface $event)
        {
            parent::beforeFilter($event);

            $this->Security->setConfig('unlockedActions', ['edit']);
        }
    }

この例では、edit アクションのすべてのセキュリティチェックが無効になります。

.. meta::
    :title lang=ja: セキュリティ
    :keywords lang=ja: 設定可能パラメーター,セキュリティコンポーネント,設定パラメーター,不正なリクエスト,防御機能,堅牢なセキュリティ,穴あけ,php クラス,meth,404 エラー,有効期限切れ,csrf,配列,投稿,セキュリティクラス,セキュリティ無効化,unlockActions
