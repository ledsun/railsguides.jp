
Action Controller の概要
==========================

本ガイドでは、コントローラの動作と、アプリケーションのリクエストサイクルでコントローラがどのように使われるかについて説明します。

このガイドの内容:

* コントローラを経由するリクエストの流れを理解する
* コントローラに渡されるパラメータを制限する方法
* セッションやcookieにデータを保存する理由とその方法
* リクエストの処理中にフィルタを使ってコードを実行する方法
* Action ControllerにビルトインされているHTTP認証
* ユーザーのブラウザにデータを直接ストリーミング送信する方法
* 機密性の高いパラメータをフィルタしてログに出力されないようにする方法
* リクエスト処理中に発生する可能性のある例外の取り扱い

--------------------------------------------------------------------------------


コントローラの役割
--------------------------

Action Controllerは、[MVC](https://ja.wikipedia.org/wiki/Model_View_Controller)モデルのCに相当します。リクエストを処理するコントローラがルーティング設定によって指名されると、コントローラはリクエストの意味を理解し、適切な出力を行なうための責任を持ちます。幸い、これらの処理はほとんどAction Controllerが行ってくれます。しかも吟味された規則を使うので、リクエストの処理は可能な限り素直な方法で行われます。

従来のいわゆる[RESTful](https://ja.wikipedia.org/wiki/REST) なアプリケーションでは、コントローラはリクエストを受け取り (この部分は開発者からは見えないようになっています)、データをモデルから取得したりモデルに保存するなどの作業を行い、最後にビューを用いてHTML出力を生成する、という役割を担います。自分のコントローラのつくりがこれと少し違っていたりするかもしれませんが、気にする必要はありません。ここではコントローラの一般的な使用法について説明しています。

コントローラは、モデルとビューの間に立って仲介を行っているという見方もできます。コントローラはモデルのデータをビューで使えるようにすることで、データをビューで表示したり、入力されたデータでモデルを更新したりします。

NOTE: ルーティングの詳細については、本ガイドの[Railsのルーティング](routing.html)を参照してください。

コントローラの命名規則
----------------------------

Railsのコントローラ名(ここでは「Controller」という文字は除きます)は、基本的に名前の最後の部分に「複数形」を使います。ただしこれは絶対的に守らなければならないというものではありません (実際 `ApplicationController`はApplicationが単数になっています)。たとえば、`ClientsController`の方が`ClientController`より好ましく、`SiteAdminsController`は`SiteAdminController`や`SitesAdminsController`よりも好ましい、といった具合です。

しかし、この規則には従っておくことをお勧めします。その理由は、`resources`などのデフォルトのルーティングジェネレータがそのまま利用できるのと、名前付きルーティングヘルパーの用法がアプリケーション全体で一貫するからです。コントローラ名の最後が複数形になっていないと、たとえば`resources`で簡単に一括ルーティングできず、`:path`や`:controller`をいちいち指定しなければなりません。詳細については、[レイアウト・レンダリングガイド](layouts_and_rendering.html)を参照してください。

NOTE: モデルの命名規則は「単数形」であることが期待され、コントローラの命名規則とは異なります。


メソッドとアクション
-------------------

Railsのコントローラは、`ApplicationController`を継承したRubyのクラスであり、他のクラスと同様のメソッドが使えます。アプリケーションがブラウザからのリクエストを受け取ると、ルーティングによってコントローラとアクションが指名され、Railsはそれに応じてコントローラのインスタンスを生成し、アクション名と同じ名前のメソッドを実行します。

```ruby
class ClientsController < ApplicationController
  def new
  end
end
```


例として、クライアントを1人追加するためにブラウザでアプリケーションの`/clients/new`にアクセスすると、Railsは`ClientsController`のインスタンスを作成して`new`メソッドを呼び出します。ここでご注目いただきたいのは、`new`メソッドの内容が空であるにもかかわらず正常に動作するという点です。これは、Railsでは`new`アクションで特に指定のない場合には`new.html.erb`ビューをレンダリングするようになっているからです。`Client`を新規作成すると、`new`メソッドはビューでアクセス可能な`@client`インスタンス変数を1つ作成できるようになります。

```ruby
def new
  @client = Client.new
end
```

これについて詳しくは、[レイアウト・レンダリングガイド](layouts_and_rendering.html)を参照してください。

`ApplicationController`は、便利なメソッドが多数定義されている`ActionController::Base`を継承しています。本ガイドではそれらの一部について説明しますが、もっと知りたい場合は[APIドキュメント](https://api.rubyonrails.org/classes/ActionController.html)かRailsのソースコードを参照してください。

アクションはpublicなメソッドでないと呼び出せません。補助メソッドやフィルタのような、アクションとして扱われたくないメソッドは`private`や`protected`を指定して公開しないようにするのが定石です。

パラメータ
----------

コントローラのアクションでは、ユーザーから送信されたデータやその他のパラメータにアクセスして何か作業を行なうのが普通です。Railsに限らず、一般にWebアプリケーションでは2種類のパラメータを扱うことができます。1番目は、URLの一部として送信されるパラメータで、「クエリ文字列パラメータ」と呼ばれます。クエリ文字列は、常にURLの"?"の後に置かれます。2番目のパラメータは、「POSTデータ」と呼ばれるものです。POSTデータは通常、ユーザーが記入したHTMLフォームから受け取ります。これがPOSTデータと呼ばれているのは、HTTP POSTリクエストの一部として送信されるからです。Railsでは、クエリ文字列パラメータの受け取り方とPOSTデータの受け取り方に違いはありません。どちらもコントローラ内では`params`という名前のハッシュでアクセスできます。

```ruby
class ClientsController < ApplicationController
  # このアクションではクエリ文字列パラメータが使われています
  # 送信側でHTTP GETリクエストが使われているためです
  # ただしパラメータにアクセスするうえでは下との違いは生じません
  # 有効な顧客リストを得るため、このアクションへのURLは以下のようになっています
  # clients: /clients?status=activated
  def index
    if params[:status] == "activated"
      @clients = Client.activated
    else
      @clients = Client.inactivated
    end
  end

  # このアクションではPOSTパラメータが使われています。このパラメータは通常
  # ユーザーが送信したHTMLフォームが元になります。
  # これはRESTfulなアクセスであり、URLは"/clients"となります。
  # データはURLではなくリクエストのbodyの一部として送信されます。
  def create
    @client = Client.new(params[:client])
    if @client.save
      redirect_to @client
    else
      # 以下の行ではデフォルトのレンダリング動作を上書きします。
      # 本来は"create"ビューが描画されます。
      render "new"
    end
  end
end
```

### ハッシュと配列のパラメータ

`params`ハッシュは、一次元のキー・値ペアしか格納できないということはありません。配列や、ネストしたハッシュを格納することもできます。値の配列をフォームから送信したい場合は、以下のように空の角かっこ`[]`をキー名に追加してください。

```
GET /clients?ids[]=1&ids[]=2&ids[]=3
```

NOTE: `[`や`]`はURLで使えない文字なので、この例の実際のURLは`/clients?ids%5b%5d=1&ids%5b%5d=2&ids%5b%5d=3`のようになります。これについては、ブラウザで自動的にエンコードされ、Railsがパラメータ受け取り時に自動的に復元するので、通常は気にする必要はありません。ただし、何らかの理由でサーバーにリクエストを手動送信しなければならない場合には、この点を忘れないようにする必要があります。

これで、受け取った`params[:ids]`の値は`["1", "2", "3"]`になりました。もうひとつ注意すべきは、パラメータの値はすべて「文字列」であることです。Railsはパラメータの型推測や型変換を行いません。

NOTE: `params`の中に`[nil]`や`[nil, nil, ...]`などの値があると、すべて自動的に`[]`に置き換えられます。この動作は、デフォルトのセキュリティ上の理由にもとづいています。詳しくは [セキュリティガイド](security.html#安全でないクエリ生成) を参照してください。

フォームから、中かっこ`[]の中にキー名を含めたハッシュを送信するには以下のようにします。

```html
<form accept-charset="UTF-8" action="/clients" method="post">
  <input type="text" name="client[name]" value="Acme" />
  <input type="text" name="client[phone]" value="12345" />
  <input type="text" name="client[address][postcode]" value="12345" />
  <input type="text" name="client[address][city]" value="Carrot City" />
</form>
```

このフォームを送信すると、`params[:client]`の値は`{ "name" => "Acme", "phone" => "12345", "address" => { "postcode" => "12345", "city" => "Carrot City" } }`になります。`params[:client][:address]`のハッシュがネストしていることにご注目ください。

この`params`ハッシュは一見ハッシュのように振る舞いますが、キーとしてシンボルと文字列のどちらでも指定できる点がハッシュと異なります。

### JSONパラメータ

Webサービスを開発していると、パラメータをJSONフォーマットで受け取れたら便利なのにと思うことがあります。Railsでは、リクエストの「Content-Type」に「application/json」が指定されていれば、パラメータが自動的に`params`ハッシュに読み込まれ、以後は通常の`params`ハッシュと同様に操作できます。

たとえば、以下のJSONコンテンツを送信したとします。

```json
{ "company": { "name": "acme", "address": "123 Carrot Street" } }
```

`params[:company]`では`{ "name" => "acme", "address" => "123 Carrot Street" }`という値を受け取ります。

同様に、初期化設定で`config.wrap_parameters`をオンにした場合や、コントローラで`wrap_parameters`が呼び出された場合、JSONパラメータのルート要素を安全に取り除くことができます。このときデフォルトでは、このパラメータが複製されてコントローラ名に応じたキー名でラップされます。従って、上のJSONリクエストは以下のように書けます。

```json
{ "name": "acme", "address": "123 Carrot Street" }
```

データの送信先が`CompaniesController`であれば、以下のように`:company`というキーでラップされます。

```ruby
{ name: "acme", address: "123 Carrot Street", company: { name: "acme", address: "123 Carrot Street" } }
```

キー名のカスタマイズや、ラップしたい特定のパラメータについては[APIドキュメント](https://api.rubyonrails.org/classes/ActionController/ParamsWrapper.html)を参照してください。

NOTE: 従来のXMLパラメータ解析のサポートは、`actionpack-xml_parser`という別のgemに切り出されました。

### ルーティングパラメータ

`params`ハッシュに必ず含まれるキーは`:controller`キーと`:action`キーです。ただしこれらの値には直接アクセスせず、`controller_name`と`action_name`という専用のメソッドを使ってください。ルーティングで定義されるその他の値パラメータ (`id`など) にもアクセスできます。例として、「有効」と「無効」のいずれかで表される顧客のリストについて考えてみましょう。「プリティな」URLに含まれる`:status`パラメータを捉えるためのルートを1つ追加してみましょう。

```ruby
get '/clients/:status', to: 'clients#index', foo: 'bar'
```

この場合、ブラウザで`/clients/active`というURLを開くと、`params[:status]`が「active」(有効) に設定されます。このルーティングを使うと、あたかもクエリ文字列で渡したかのように`params[:foo]`にも"bar"が設定されます。コントローラでも`params[:action]`をindexとして、`params[:controller]`をclientsとして受け取ります。

### `default_url_options`

コントローラで`default_url_options`という名前のメソッドを定義すると、URL生成用のグローバルなデフォルトパラメータを設定できます。このようなメソッドは、必要なデフォルト値を持つハッシュを必ず1つ返さねばならず、そのハッシュのキーはシンボルでなければなりません。

```ruby
class ApplicationController < ActionController::Base
  def default_url_options
    { locale: I18n.locale }
  end
end
```

これらのオプションはURL生成の開始点として使われるので、`url_for`呼び出しに渡されるオプションで上書きできます。

`ApplicationController`で`default_url_options`を定義すると、上の例で示したように、すべてのURL生成で使われるようになります。このメソッドを特定のコントローラで定義すれば、そのコントローラで生成されるURLにだけ影響します。

指定されたリクエストでは、生成された単一のURLごとにこのメソッドが実際に呼ばれるわけではありません。パフォーマンス上の理由により戻り値であるハッシュがキャッシュされるので、呼び出しは最大でもリクエストごとに1回までとなります。

### Strong Parameters

strong parametersを用いることで、Action Controllerのパラメータが許可されるまでActive Modelの「マスアサインメント」に利用されることを禁止できます。つまり、多くの属性を一度に更新したい場合は、どの属性のマスアップデートを許可するかを開発者が明示的に指定しなければなりません。大雑把にすべての属性の更新を一括で許可してしまうと、外部に公開する必要のない属性まで誤って公開してしまう可能性が生じるため、そのような事態を防ぐための機能です。

さらに、パラメータの属性に「必須 (required)」を指定することで、事前に定義したraise/rescueフローによって、渡された必須パラメータが不足している場合に「400 Bad Request」で終了させることもできます。

```ruby
class PeopleController < ActionController::Base
  # 以下のコードはActiveModel::ForbiddenAttributesError例外を発生します
  # 明示的な許可を行なわずに、パラメータを一括で渡してしまう
  # 危険な「マスアサインメント」が行われているからです。
  def create
    Person.create(params[:person])
  end

  # 以下のコードは、パラメータにpersonキーがあれば成功します。
  # personキーがない場合は
  # ActionController::ParameterMissing例外を発生します。
  # この例外はActionController::Baseにキャッチされ、
  # 400 Bad Requestを返します。
  def update
    person = current_account.people.find(params[:id])
    person.update!(person_params)
    redirect_to person
  end

  private
    # 許可するパラメータはprivateメソッドでカプセル化します。
    # これは非常によい手法であり、createとupdateの両方で使いまわすことで
    # 同じ許可を与えることができます。また、許可する属性をユーザーごとにチェックするよう
    # このメソッドを特殊化することもできます。
    def person_params
      params.require(:person).permit(:name, :age)
    end
end
```

#### 許可済みスカラー値

以下の例で説明します。

```ruby
params.permit(:id)
```

`:id`キーが`params`にあり、それに対応する許可済みスカラー値に`:id`キーがあれば、許可リストチェックをパスします。この条件を満たさない場合は、`:id`キーはフィルタで除外されます。これにより、外部からハッシュなどのオブジェクトを不正に注入できなくなります。

スカラーで許可される型は、`String`、`Symbol`、`NilClass`、`Numeric`、`TrueClass`、`FalseClass`、`Date`、`Time`、`DateTime`、`StringIO`、`IO`、`ActionDispatch::Http::UploadedFile`、`Rack::Test::UploadedFile`です。

「`params`の値には許可されたスカラー値の**配列**を使わなければならない」ことを宣言するには、以下のようにキーに空の配列を対応付けます。

```ruby
params.permit(id: [])
```

ハッシュパラメータやその内部構造の正しいキーをすべて明示的に宣言できない場合や、すべて宣言するのが面倒な場合があります。次のように空のハッシュを割り当てることは一応可能です。

```ruby
params.permit(preferences: {})
```

ただしこのように指定してしまうと任意の入力を受け付けてしまうため、利用には十分ご注意ください。この場合`permit`によって、受け取った構造内の値が許可済みのスカラーとして扱われ、それ以外の値がフィルタで除外されます。

パラメータのハッシュ全体を許可したい場合は、`permit!`メソッドを使えます。

```ruby
params.require(:log_entry).permit!
```

こうすることで、`:log_entry`パラメータハッシュとすべてのサブハッシュが「許可済み（permitted）」としてマーキングされ、許可済みスカラーであるかどうかがチェックされなくなってあらゆる値を受け付けるようになります。ただし、`permit!`はくれぐれも慎重にお使いください。現在のモデルの属性はもちろん、将来モデルに追加される属性も一括で許可してしまうためです。

#### ネストしたパラメータ

`permit`は、以下のようにネストしたパラメータに対しても使えます。

```ruby
params.permit(:name, { emails: [] },
              friends: [ :name,
                         { family: [ :name ], hobbies: [] }])
```

この宣言では、`name`、`emails`、`friends`属性が許可されます。ここでは、`emails`は許可を受けたスカラー値の配列であることが期待され、`friends`は特定の属性を持つリソースの配列であることが期待され、どちらも`name`属性(許可を受けたあらゆるスカラー値を受け付ける)を持ちます。また、`hobbies`属性(許可を受けたスカラー値の配列)や、`family`属性(同じく、許可を受けたあらゆるスカラー値を受け付ける)を持つことも期待されます。

#### その他の事例

今度は`new`アクションでも許可済み属性を使いたいところです。しかし今度は`new`を呼び出す時点ではルートキーがないので、ルートキーに対して`require`を指定することができないという問題があります。

```ruby
# `fetch`を使うとデフォルト値を渡して
# Strong Parameters APIを使えるようになります
params.fetch(:blog, {}).permit(:title, :author)
```

このモデルの`accepts_nested_attributes_for`クラスメソッドを使えば、関連付けられたレコードを更新または削除できます。このメソッドは、`id`と`_destroy`パラメータに基づいて動作します。

```ruby
# :id と :_destroyを許可します。
params.require(:author).permit(:name, books_attributes: [:title, :id, :_destroy])
```

キーが整数のハッシュは異なる方法で処理されます。これらは、あたかも直接の子オブジェクトであるかのように属性を宣言できます。`has_many`関連付けとともに`accepts_nested_attributes_for`メソッドを使うと、このようなパラメータを取得できます。

```ruby
# 以下のデータを許可
# {"book" => {"title" => "Some Book",
#             "chapters_attributes" => { "1" => {"title" => "First Chapter"},
#                                        "2" => {"title" => "Second Chapter"}}}}

params.require(:book).permit(:title, chapters_attributes: [:title])
```

次のような状況を想像してみましょう。製品名と、その製品名に関連する任意のデータを表すパラメータがあるとします。そして、この製品名もデータハッシュ全体もまとめて許可したいとします。

```ruby
def product_params
  params.require(:product).permit(:name, data: {})
end
```


#### Strong Parametersのスコープの外

strong parameter APIは、最も一般的な使用状況を念頭に置いて設計されています。つまり、あらゆるパラメータのフィルタ問題を扱える「銀の弾丸」ではありません。しかし、このAPIを自分のコードに混在させてアプリの実情に対応しやすくなります。

セッション
-------

Railsアプリケーションにはユーザーごとにセッションが設定されます。前のリクエストの情報を次のリクエストでも利用するためにセッションに少量のデータが保存されます。セッションはコントローラとビューでのみ利用できます。また、以下のようにさまざまなストレージを選べます。

* `ActionDispatch::Session::CookieStore`: すべてのセッションをクライアント側のブラウザのcookieに保存する
* `ActionDispatch::Session::CacheStore`: データをRailsのキャッシュに保存する
* `ActionDispatch::Session::ActiveRecordStore`: Active Recordを用いてデータベースに保存する (`activerecord-session_store` gemが必要)
* `ActionDispatch::Session::MemCacheStore`: データをmemcachedクラスタに保存する (この実装は古いのでCacheStoreを検討すべき)

あらゆるセッションは、セッション固有のIDをcookieに保存します (注意: RailsでセッションIDをURLで渡すことはセキュリティ上の危険があるため許可されません。セッションIDは必ずcookieで渡さなくてはなりません)。

多くのセッションストアでは、このIDは単にサーバー上のセッションデータ (データベーステーブルなど) を検索するために使われています。ただしCookieStoreは例外的にcookie自身にすべてのセッションデータを保存します(必要であればセッションIDも利用できます)。そしてRailsではCookieStoreがデフォルトで使われ、かつRailsでの推奨セッションストアでもあります。CookieStoreの利点は、非常に軽量であることと、新規Webアプリケーションでセッションを利用するための準備がまったく不要である点です。cookieデータは改竄防止のために暗号署名が与えられています。さらにcookie自身も暗号化されているので、内容を他人に読まれないようになっています。(改ざんされたcookieはRailsが拒否します)

CookieStoreには約4KBのデータを保存できます。他のセッションストアに比べて少量ですが、通常はこれで十分です。利用するセッションストアの種類にかかわらず、セッションに大量のデータを保存することはお勧めできません。特に、セッションに複雑なオブジェクト (モデルインスタンスなどの基本的なRubyオブジェクトでないもの) を保存することはお勧めできません。このようなことをすると、サーバーがリクエスト間でセッションを再編成できずにエラーになることがあります。

ユーザーセッションに重要なデータが含まれていない場合、またはユーザーセッションを長期間保存する必要がない場合 (flashメッセージで使いたいだけの場合など) は、`ActionDispatch::Session::CacheStore`を検討してください。この方式では、Webアプリケーションに設定されているキャッシュ実装を利用してセッションを保存します。この方法のよい点は、既存のキャッシュインフラをそのまま利用してセッションを保存できることと、管理用の設定を追加する必要がないことです。この方法の欠点はセッションが短命になり、セッションがいつでも消える可能性がある点です。

セッションストレージの詳細については[セキュリティガイド](security.html)を参照してください。

別のセッションメカニズムが必要な場合は、イニシャライザを変更することで切り替えられます。

```ruby
# Cookie ベースのデフォルトは特に機密性の高い情報を保存する目的で使うべきではありません。
# 代わりにデータベースをセッションに使います。
# (セッションテーブルの作成は"rails g active_record:session_migration"で行なう)
# Rails.application.config.session_store :active_record_store
```

Railsは、セッションデータに署名するときにセッションキー(=cookieの名前)を設定します。この動作もイニシャライザで変更できます。

```ruby
# このファイルを変更後サーバーを必ず再起動してください。
Rails.application.config.session_store :cookie_store, key: '_your_app_session'
```

`:domain`キーを渡して、cookieで使うドメイン名を指定することもできます。

```ruby
# このファイルを変更後サーバーを必ず再起動してください。
Rails.application.config.session_store :cookie_store, key: '_your_app_session', domain: ".example.com"
```

Railsは、`config/credentials.yml.enc`のセッションデータの署名に用いる秘密鍵を (CookieStore用に) 設定します。この秘密鍵は`rails credentials:edit`で変更できます。


```ruby
# aws:
#   access_key_id: 123
#   secret_access_key: 345

# Used as the base secret for all MessageVerifiers in Rails, including the one protecting cookies.
secret_key_base: 492f...
```

NOTE: `CookieStore`を使用中に`secret_key_base`を変更すると、既存のセッションがすべて無効になります。

### セッションにアクセスする

コントローラ内では、`session`インスタンスメソッドを使ってセッションにアクセスできます。

NOTE: セッションは遅延読み込みされます（lazy loaded）。アクションのコードでセッションにアクセスしなかった場合、セッションは読み込まれません。つまり、セッションにアクセスしていなければセッションを無効にする必要はまったくありません。アクセスしないようにすればセッションは無効になります。

セッションの値は、ハッシュに似たキー/値ペアを使って保存されます。

```ruby
class ApplicationController < ActionController::Base

  private

  # キー付きのセッションに保存されたidでユーザーを検索する
  # :current_user_id はRailsアプリケーションでユーザーログインを扱う際の定番の方法。
  # ログインするとセッション値が設定され、
  # ログアウトするとセッション値が削除される。
  def current_user
    @_current_user ||= session[:current_user_id] &&
      User.find_by(id: session[:current_user_id])
  end
end
```

セッションに何かを保存したければ、ハッシュのようにキーにそれを割り当てます。

```ruby
class LoginsController < ApplicationController
  # "Create" a login, aka "log the user in"
  def create
    if user = User.authenticate(params[:username], params[:password])
      # セッションのuser idを保存し、
      # 今後のリクエストで使えるようにする
      session[:current_user_id] = user.id
      redirect_to root_url
    end
  end
end
```

セッションからデータの一部を削除したい場合は、そのキーバリューペアを削除します。

```ruby
class LoginsController < ApplicationController
  # ログインを削除する (=ログアウト)
  def destroy
    # セッションidからuser idを削除する
    session.delete(:current_user_id)
    # メモ化された現在のユーザーをクリアする
    @_current_user = nil
    redirect_to root_url
  end
end
```

セッション全体をリセットするには`reset_session`を使います。

### Flash

flashはセッションの中の特殊な部分であり、リクエストごとにクリアされます。つまりflashは「直後のリクエスト」でのみ参照可能になるという特徴を持ち、エラーメッセージをビューに渡したりするのに便利です。

flashのアクセス方法はセッションとほとんど同じで、ハッシュとしてアクセスします (これを[FlashHash](https://api.rubyonrails.org/classes/ActionDispatch/Flash/FlashHash.html) インスタンスと呼びます)。

例として、ログアウトする動作を扱ってみましょう。コントローラでflashを使うことで、次のリクエストで表示するメッセージを送信できます。

```ruby
class LoginsController < ApplicationController
  def destroy
    session.delete(:current_user_id)
    flash[:notice] = "You have successfully logged out."
    redirect_to root_url
  end
end
```

flashメッセージはリダイレクトの一部として割り当てることもできることにご注目ください。オプションとして`:notice`、`:alert`の他に、一般的な`:flash`も使えます。

```ruby
redirect_to root_url, notice: "You have successfully logged out."
redirect_to root_url, alert: "You're stuck here!"
redirect_to root_url, flash: { referral_code: 1234 }
```

この`destroy`アクションでは、アプリケーションの`root_url`にリダイレクトし、そこでメッセージを表示します。flashメッセージは、直前のアクションでflashメッセージにどのようなメッセージを設定していたかにかかわらず、次に行われるアクションだけですべて決まりますのでご注意ください。Railsアプリケーションのレイアウトでは、警告や通知をflashで表示するのが通例です。

```erb
<html>
  <!-- <head/> -->
  <body>
    <% flash.each do |name, msg| -%>
      <%= content_tag :div, msg, class: name %>
    <% end -%>

    <!-- 以下略 -->
  </body>
</html>
```

このように、アクションで通知(notice)や警告(alert)メッセージを設置すると、レイアウト側で自動的にそのメッセージが表示されます。

flashには、通知や警告に限らず、セッションに保存可能なものであれば何でも保存できます。

```erb
<% if flash[:just_signed_up] %>
  <p class="welcome">Welcome to our site!</p>
<% end %>
```

flashの値を別のリクエストにも引き継ぎたい場合は、`keep`メソッドを使います。

```ruby
class MainController < ApplicationController
  # このアクションはroot_urlに対応しており、このアクションに対する
  # すべてのリクエストをUsersController#indexにリダイレクトしたいとします。
  # あるアクションでflashを設定してこのindexアクションにリダイレクトすると、
  # 別のリダイレクトが発生した場合にはflashは消えてしまいます。
  # ここで'keep'を使うと別のリクエストでflashが消えないようになります。
  def index
    # すべてのflash値を保持する
    flash.keep

    # キーを指定して値を保持することもできる
    # flash.keep(:notice)
    redirect_to users_url
  end
end
```

#### `flash.now`

デフォルトでは、flashに値を追加すると直後のリクエストでその値を利用できますが、次のリクエストを待たずに同じリクエスト内でこれらのflash値にアクセスしたい場合があります。たとえば、`create`アクションに失敗してリソースが保存されなかった場合に、`new`テンプレートを直接描画するとします。このとき新しいリクエストは行われませんが、この状態でもflashを使ってメッセージを表示したいこともあります。このような場合、`flash.now`を使えば通常の`flash`と同じ要領でメッセージを表示できます。

```ruby
class ClientsController < ApplicationController
  def create
    @client = Client.new(params[:client])
    if @client.save
      # ...
    else
      flash.now[:error] = "Could not save client"
      render action: "new"
    end
  end
end
```

Cookie
-------

Webアプリケーションでは、cookieと呼ばれる少量のデータをクライアントのブラウザに保存できます。HTTPは「ステートレス」なプロトコルなので、基本的にリクエストとリクエストの間には何の関連もありませんが、cookieを使うとリクエスト同士の間で (あるいはセッション同士の間であっても) このデータが保持されるようになります。Railsでは`cookies`メソッドを使ってcookieに簡単にアクセスできます。アクセス方法はセッションの場合とよく似ていて、ハッシュのように動作します。

```ruby
class CommentsController < ApplicationController
  def new
    # cookieにコメント作者名が残っていたらフィールドに自動入力する
    @comment = Comment.new(author: cookies[:commenter_name])
  end

  def create
    @comment = Comment.new(params[:comment])
    if @comment.save
      flash[:notice] = "Thanks for your comment!"
      if params[:remember_name]
        # コメント作者名を保存する
        cookies[:commenter_name] = @comment.author
      else
        # コメント作者名がcookieに残っていたら削除する
        cookies.delete(:commenter_name)
      end
      redirect_to @comment.article
    else
      render action: "new"
    end
  end
end
```

セッションを削除する場合はキーに`nil`を指定することで削除しましたが、cookieを削除する場合はこの方法ではなく、`cookies.delete(:key)`を使ってください。

Railsでは、機密データ保存のために署名済みcookie jarと暗号化cookie jarを利用することもできます。署名済みcookie jarでは、暗号化した署名をcookie値に追加することで、cookieの改竄を防ぎます。暗号化cookie jarでは、署名の追加と共に、値自体を暗号化してエンドユーザーに読まれることのないようにします。
詳細については[APIドキュメント](https://api.rubyonrails.org/classes/ActionDispatch/Cookies.html)を参照してください。

これらの特殊なcookieではシリアライザを使って値を文字列に変換して保存し、読み込み時に逆変換(deserialize)を行ってRubyオブジェクトに戻しています。

利用するシリアライザを指定することもできます。

```ruby
Rails.application.config.action_dispatch.cookies_serializer = :json
```

新規アプリケーションのデフォルトシリアライザは`:json`です。既存のcookieが残っている旧アプリケーションとの互換性のため、`serializer`オプションに何も指定されていない場合は`:marshal`が使われます。

シリアライザのオプションに`:hybrid`を指定することもできます。これを指定すると、`Marshal`でシリアライズされた既存のcookieも透過的にデシリアライズし、保存時に`JSON`フォーマットで保存し直します。これは、既存のアプリケーションを`:json`シリアライザに移行するときに便利です。

`load`メソッドと`dump`メソッドに応答するカスタムのシリアライザを指定することもできます。

```ruby
Rails.application.config.action_dispatch.cookies_serializer = MyCustomSerializer
```

`:json`または`:hybrid`シリアライザを使う場合、一部のRubyオブジェクトがJSONとしてシリアライズされない可能性があることにご注意ください。たとえば、`Date`オブジェクトや`Time`オブジェクトはstringsとしてシリアライズされ、`Hash`のキーはstringに変換されます。

```ruby
class CookiesController < ApplicationController
  def set_cookie
    cookies.encrypted[:expiration_date] = Date.tomorrow # => Thu, 20 Mar 2014
    redirect_to action: 'read_cookie'
  end

  def read_cookie
    cookies.encrypted[:expiration_date] # => "2014-03-20"
  end
end
```

cookieには文字列や数字などの「単純なデータ」だけを保存することをお勧めします。cookieに複雑なオブジェクトを保存しなければならない場合は、後続のリクエストでcookieから値を読み出す場合の変換については自分で面倒を見る必要があります。

cookieセッションストアを使う場合、`session`や`flash`ハッシュについても同様のことが該当します。

XMLとJSONデータを出力する
---------------------------

ActionControllerのおかげで、`XML`データや`JSON`データの出力 (レンダリング) は非常に簡単に行えます。scaffoldを使って生成したコントローラは以下のようになっているはずです。

```ruby
class UsersController < ApplicationController
  def index
    @users = User.all
    respond_to do |format|
      format.html # index.html.erb
      format.xml  { render xml: @users }
      format.json { render json: @users }
    end
  end
end
```

上のコードでは、`render xml: @users.to_xml`ではなく`render xml: @users`となっていることにご注目ください。Railsは、オブジェクトが`String`型でない場合は自動的に`to_xml`を呼んでくれます。

フィルタ
-------

フィルタは、コントローラにあるアクションの「直前 (before)」、「直後 (after)」、あるいは「直前と直後の両方 (around)」に実行されるメソッドです。

フィルタは継承されるので、フィルタを`ApplicationController`で設定すればアプリケーションのすべてのコントローラでフィルタが有効になります。

「before系」フィルタは、リクエストサイクルを止めてしまう可能性があるのでご注意ください。「before系」フィルタのよくある使われ方として、ユーザーがアクションを実行する前にログインを要求するというのがあります。このフィルタメソッドは以下のような感じになるでしょう。

```ruby
class ApplicationController < ActionController::Base
  before_action :require_login

  private

  def require_login
    unless logged_in?
      flash[:error] = "You must be logged in to access this section"
      redirect_to new_login_url # halts request cycle
    end
  end
end
```

このメソッドはエラーメッセージをflashに保存し、ユーザーがログインしていない場合にはログインフォームにリダイレクトするというシンプルなものです。「before系」フィルタによってビューのレンダリングやリダイレクトが行われると、このアクションは実行されません。フィルタの実行後に実行されるようスケジュールされた追加のフィルタがある場合、これらもキャンセルされます。

この例ではフィルタを`ApplicationController`に追加したので、これを継承するすべてのコントローラが影響を受けます。つまり、アプリケーションのあらゆる機能についてログインが要求されることになります。当然ですが、アプリケーションのあらゆる画面で認証を要求してしまうと、認証に必要なログイン画面まで表示できなくなるという困った事態になってしまいます。したがって、このようにすべてのコントローラやアクションでログイン要求を設定すべきではありません。`skip_before_action`メソッドを使えば、特定のアクションでフィルタをスキップできます。


```ruby
class LoginsController < ApplicationController
  skip_before_action :require_login, only: [:new, :create]
end
```

上のようにすることで、`LoginsController`の`new`アクションと`create`アクションをこれまでどおり認証不要にできました。特定のアクションについてだけフィルタをスキップしたい場合には`:only`オプションを使います。逆に特定のアクションについてだけフィルタをスキップしたくない場合は`:except`オプションを使います。これらのオプションはフィルタの追加時にも使えるので、最初の場所で選択したアクションに対してだけ実行されるフィルタを追加することもできます。

NOTE: 同じフィルタを異なるオプションで複数回呼び出しても期待どおりに動作しません。最後に呼び出されたフィルタ定義によって、それまでのフィルタ定義は上書きされます。

### afterフィルタとaroundフィルタ

「before系」フィルタ以外に、アクションの実行後に実行されるフィルタや、実行前実行後の両方で実行されるフィルタを使うこともできます。

「after系」フィルタは「before系」フィルタと似ていますが、「after系」フィルタの場合アクションは既に実行済みであり、クライアントに送信されようとしている応答データにアクセスできる点が「before系」フィルタとは異なります。当然ながら、「after系」フィルタをどのように書いても、アクションの実行が中断するようなことはありません。ただし、「after系」フィルタは、アクションが成功した後にしか実行されず、リクエストサイクルの途中で例外が発生した場合は実行されませんのでご注意ください。

「around系」フィルタを使う場合は、フィルタ内のどこかで必ず`yield`を実行して、関連付けられたアクションを実行してやる義務が生じます。これはRackミドルウェアの動作と似ています。

たとえば、何らかの変更に際して承認ワークフローがあるWebサイトを考えてみましょう。管理者はこれらの変更内容を簡単にプレビューし、トランザクション内で承認できるとします。

```ruby
class ChangesController < ApplicationController
  around_action :wrap_in_transaction, only: :show

  private

  def wrap_in_transaction
    ActiveRecord::Base.transaction do
      begin
        yield
      ensure
        raise ActiveRecord::Rollback
      end
    end
  end
end
```

「around系」フィルタの場合、画面出力 (レンダリング) もその作業に含まれることにご注意ください。特に上の例では、ビュー自身がデータベースから (スコープなどを使って) 読み出しを行うとすると、その読み出しはトランザクション内で行われ、データがプレビューに表示されます。

あえて`yield`を実行せず、自分でレスポンスをビルドするという方法もあります。この場合、アクションは実行されません。

### その他のフィルタ使用法

最も一般的なフィルタの使用法は、privateメソッドを作成し、そのメソッドを`*_action`で追加することですが、同じ結果を得られるフィルタ使用法が他にも2とおりあります。

1番目は、`*_action` メソッドに対して直接ブロックを与えることです。このブロックはコントローラを引数として受け取ります。さきほどの`require_login`フィルタを書き換えて、ブロックを使うようにします。

```ruby
class ApplicationController < ActionController::Base
  before_action do |controller|
    unless controller.send(:logged_in?)
      flash[:error] = "You must be logged in to access this section"
      redirect_to new_login_url
    end
  end
end
```

このとき、フィルタで`send`メソッドを使っていることにご注意ください。その理由は、`logged_in?`メソッドはprivateであり、コントローラのスコープではフィルタが動作しないためです (訳注: `send`メソッドを使うとprivateメソッドを呼び出せます)。この方法は、特定のフィルタを実装する方法としては推奨されませんが、もっとシンプルな場合には役に立つことがあるかもしれません。

2番目の方法はクラスを使ってフィルタを扱うものです (実際には、正しいメソッドに応答するオブジェクトであれば何でも構いません)。他の2つの方法で実装すると読みやすくならず、再利用も困難になるような複雑な場合に有用です。例として、ログインフィルタをクラスで書き換えてみましょう。

```ruby
class ApplicationController < ActionController::Base
  before_action LoginFilter
end

class LoginFilter
  def self.before(controller)
    unless controller.send(:logged_in?)
      controller.flash[:error] = "You must be logged in to access this section"
      controller.redirect_to controller.new_login_url
    end
  end
end
```

繰り返しますが、この例はフィルタとして理想的なものではありません。その理由は、このフィルタがコントローラのスコープで動作せず、コントローラが引数として渡されるからです。このフィルタクラスには、フィルタと同じ名前のメソッドを実装する必要があります。従って、`before_action`フィルタの場合、クラスに`before`メソッドを実装しなければならない、という具合です。`around`メソッド内では`yield`を呼んでアクションを実行してやる必要があります。

リクエストフォージェリからの保護
--------------------------

クロスサイトリクエストフォージェリ（cross-site request forgery）は攻撃手法の一種です。邪悪なWebサイトがユーザーをだまし、攻撃目標となるWebサイトへの危険なリクエストを知らないうちに作成させるというものです。攻撃者は標的ユーザーに関する知識や権限を持っていなくても、目標サイトに対してデータの追加/変更/削除を行わせることができてしまいます。

この攻撃を防ぐために必要な手段の第一歩は、「create/update/destroyのような破壊的な操作に対して絶対にGETリクエストでアクセスできない」ようにすることです。WebアプリケーションがRESTful規則に従っていれば、これは守られているはずです。しかし、邪悪なWebサイトはGET以外のリクエストを目標サイトに送信することなら簡単にできてしまいます。リクエストフォージェリはまさにここを保護するためのものであり、文字どおりリクエストの偽造(forgery)から保護します。

具体的な保護方法は、推測不可能なトークンをすべてのリクエストに追加することです。このトークンはサーバーだけが知っています。これにより、リクエストに含まれているトークンが不正であればアクセスが拒否されます。

以下のようなフォームを試しに生成してみます。

```erb
<%= form_with model: @user, local: true do |form| %>
  <%= form.text_field :username %>
  <%= form.text_field :password %>
<% end %>
```

以下のようにトークンが隠しフィールドに追加されている様子がわかります。

```html
<form accept-charset="UTF-8" action="/users/1" method="post">
<input type="hidden"
       value="67250ab105eb5ad10851c00a5621854a23af5489"
       name="authenticity_token"/>
<!-- fields -->
</form>
```

Railsでは、[formヘルパー](form_helpers.html)を使って生成されたあらゆるフォームにトークンを追加するので、この攻撃を心配する必要はほとんどありません。formヘルパーを使わず手作りした場合や、別の理由でトークンが必要な場合には、`form_authenticity_token`メソッドでトークンを生成できます。

`form_authenticity_token`メソッドは、有効な認証トークンを生成します。このメソッドは、カスタムAjax呼び出しなどのように、Railsが自動的にトークンを追加してくれない箇所で使うのに便利です。

本ガイドの[セキュリティガイド](security.html)では、この話題を含む多くのセキュリティ問題について言及しており、いずれもWebアプリケーションを開発するうえで必読です。

`request`オブジェクトと`response`オブジェクト
--------------------------------

すべてのコントローラには、現在実行中のリクエストサイクルに関連するリクエストオブジェクトとレスポンスオブジェクトを指す、2つのアクセサメソッドがあります。`request`メソッドは`ActionDispatch::Request`クラスのインスタンスを含み、`response`メソッドは、今クライアントに戻されようとしている内容を表すレスポンスオブジェクトを返します。


### `request`オブジェクト

リクエストオブジェクトには、クライアントブラウザから返されるリクエストに関する有用な情報が多数含まれています。利用可能なメソッドをすべて知りたい場合は[Rails APIドキュメント](https://api.rubyonrails.org/classes/ActionDispatch/Request.html)と[Rackドキュメント](https://www.rubydoc.info/github/rack/rack/Rack/Request)を参照してください。その中から、このオブジェクトでアクセス可能なメソッドを紹介します。

| `request`のプロパティ                     | 目的                                                                          |
| ----------------------------------------- | -------------------------------------------------------------------------------- |
| host                                      | リクエストで使われるホスト名                                              |
| domain(n=2)                               | ホスト名の右 (TLD:トップレベルドメイン) から数えて`n`番目のセグメント            |
| format                                    | クライアントからリクエストされたContent-Type                                        |
| method                                    | リクエストで使われるHTTPメソッド                                            |
| get?, post?, patch?, put?, delete?, head? | HTTPメソッドがGET/POST/PATCH/PUT/DELETE/HEADのいずれかの場合にtrueを返す               |
| headers                                   | リクエストに関連付けられたヘッダーを含むハッシュを返す               |
| port                                      | リクエストで使われるポート番号 (整数)                                  |
| protocol                                  | プロトコル名に"://"を加えたものを返す ("http://"など) |
| query_string                              | URLの一部で使われるクエリ文字 ("?"より後の部分)                    |
| remote_ip                                 | クライアントのipアドレス                                                    |
| url                                       | リクエストで使われるURL全体                                             |

#### `path_parameters`、`query_parameters`、`request_parameters`

Railsは、リクエストに関連するすべてのパラメータを`params`ハッシュに集約してくれます。例えば、クエリ文字列や、POSTで送信されたパラメータがそうです。`request`オブジェクトには3つのアクセサがあり、パラメータの由来に応じたアクセスを行なうこともできます。`query_parameters`ハッシュにはクエリ文字列として送信されたパラメータが含まれます。`request_parameters`ハッシュにはPOSTのbodyの一部として送信されたパラメータが含まれます。`path_parameters`には、ルーティング機構によって特定のコントローラとアクションへのパスの一部であると認識されたパラメータが含まれます。

### `response`オブジェクト

`response`オブジェクトはアクションが実行されるときにビルドされ、クライアントに送り返されるデータをレンダリングします。responseオブジェクトを直接使うことは通常ありません。しかし、たとえば「after系」フィルタ内などで`response`オブジェクトを直接操作できると便利です。`response`オブジェクトのアクセサメソッドにセッターがあれば、それを利用して`response`オブジェクトの値を直接変更できます。利用可能なメソッドをすべて知りたい場合は[Rails APIドキュメント](https://api.rubyonrails.org/classes/ActionDispatch/Request.html)と[Rackドキュメント](https://www.rubydoc.info/github/rack/rack/Rack/Request)を参照してください。

| `response`のプロパティ | 目的                                                                                             |
| ---------------------- | --------------------------------------------------------------------------------------------------- |
| body                   | クライアントに送り返されるデータの文字列。多くの場合HTML。                  |
| status                 | レスポンスのステータスコード (200 OK、404 file not foundなど)|
| location               | リダイレクト時のリダイレクト先URL                                                  |
| content_type           | レスポンスのContent-Type                                                                   |
| charset                | レスポンスで使われる文字セット。デフォルトは"utf-8"。                                  |
| headers                | レスポンスで使われるヘッダー                                                                      |

#### カスタムヘッダーの設定

レスポンスでカスタムヘッダーを使いたい場合は、`response.headers`を利用できます。このヘッダー属性はハッシュであり、ヘッダ名と値がその中でマップされています。一部の値はRailsによって自動的に設定されます。ヘッダに追加・変更を行いたい場合は以下のように`response.headers`に割り当てます。

```ruby
response.headers["Content-Type"] = "application/pdf"
```

NOTE: 上の場合、直接`content_type`セッターを使う方がずっと自然です。

HTTP認証
--------------------

Railsには2とおりのHTTP認証機構がビルトインされています。

* BASIC認証
* ダイジェスト認証

### HTTP BASIC認証

HTTP BASIC認証は認証スキームの一種であり、非常に多くのブラウザおよびHTTPクライアントによってサポートされています。例として、Webアプリケーションに管理画面があり、ブラウザのHTTP BASIC認証ダイアログウィンドウでユーザー名とパスワードを入力しないとアクセスできないようにしたいとします。このビルトイン認証メカニズムはとても簡単に利用できます。必要なのは`http_basic_authenticate_with`メソッドだけです。

```ruby
class AdminsController < ApplicationController
  http_basic_authenticate_with name: "humbaba", password: "5baa61e4"
end
```

このとき、`AdminsController`を継承した名前空間付きのコントローラを作成することもできます。このフィルタは、該当するコントローラのすべてのアクションで実行されるので、それらをHTTP BASIC認証で保護することができます。

### HTTPダイジェスト認証

HTTPダイジェスト認証は、BASIC認証よりも高度な認証システムであり、暗号化されていない平文パスワードをネットワークに送信しなくて済む利点があります (BASIC認証も、HTTPS上で行えば安全になります)。Railsではダイジェスト認証も簡単に利用できます。必要なのは`authenticate_or_request_with_http_digest`メソッドだけです。

```ruby
class AdminsController < ApplicationController
  USERS = { "lifo" => "world" }

  before_action :authenticate

  private

    def authenticate
      authenticate_or_request_with_http_digest do |username|
        USERS[username]
      end
    end
end
```

上の例で示したように、`authenticate_or_request_with_http_digest`のブロックでは引数を1つ (ユーザー名) しか取りません。そしてブロックからはパスワードが返されます。`authenticate_or_request_with_http_digest`から`nil`または`false`が返された場合は、認証が失敗します。

ストリーミングとファイルダウンロード
----------------------------

HTMLを出力するのではなく、ユーザーにファイルを直接送信したい場合があります。`send_data`メソッドと`send_file`メソッドはRailsのすべてのコントローラで利用でき、いずれもストリームデータをクライアントに送信するために使います。`send_file`は、ディスク上のファイル名を取得したり、ファイルの内容をストリーミングしたりできる、便利なメソッドです。

クライアントにデータをストリーミングするには、`send_data`を使います。

```ruby
require "prawn"
class ClientsController < ApplicationController
  # クライアントに関する情報を含むPDFを生成し、
  # 返します。ユーザーはPDFをファイルダウンロードとして取得できます。
  def download_pdf
    client = Client.find(params[:id])
    send_data generate_pdf(client),
              filename: "#{client.name}.pdf",
              type: "application/pdf"
  end

  private

    def generate_pdf(client)
      Prawn::Document.new do
        text client.name, align: :center
        text "Address: #{client.address}"
        text "Email: #{client.email}"
      end.render
    end
end
```

上の例の`download_pdf`アクションから、privateメソッドが呼び出され、実際のPDF生成はprivateメソッド側で行われます。PDFは文字列として返されます。続いてこの文字列はクライアントに対してファイルダウンロードとしてストリーミング送信されます。このときに保存用のファイル名もクライアントに示されます。ストリーミング送信するファイルを、クライアント側でファイルとしてダウンロードして欲しくない (ファイルを保存して欲しくない) 場合があります。たとえば、HTMLページに埋め込める画像ファイルを撮影したとします。このときブラウザに対して、このファイルはダウンロード用ではないということを伝えるには、`:disposition`オプションに"inline"を指定します。逆のオプションは"attachment"で、こちらはストリーミングのデフォルト設定です。

### ファイルを送信する

サーバーのディスク上にすでにあるファイルを送信するには、`send_file`メソッドを使います。

```ruby
class ClientsController < ApplicationController
  # ディスク上に生成・保存済みのファイルをストリーミング送信する
  def download_pdf
    client = Client.find(params[:id])
    send_file("#{Rails.root}/files/clients/#{client.id}.pdf",
              filename: "#{client.name}.pdf",
              type: "application/pdf")
  end
end
```

ファイルは4KBずつ読み出され、ストリーミング送信されます。これは、巨大なファイルを一度にメモリに読み込まないようにするためです。分割読み出しは`:stream`オプションでオフにできます。`:buffer_size`オプションでブロックサイズを調整することもできます。

`:type`オプションが未指定の場合、`:filename`で取得したファイル名の拡張子から推測して与えられます。拡張子に該当するContent-TypeがRailsに登録されていない場合、`application/octet-stream`が使われます。

WARNING: サーバーのディスク上のファイルパスを指定するときに、(paramsやcookieなどの) クライアントから送信されたデータを使う場合は十分な注意が必要です。クライアントから邪悪なファイルパスが送り込まれ、開発者が意図しないファイルにアクセスされてしまうというセキュリティ上のリスクがあることを常に念頭に置いてください。

TIP: 静的なファイルをわざわざRails経由でストリーミング送信することはお勧めできません。ほとんどの場合、Webサーバーのpublicフォルダに置いてダウンロードさせれば済むことです。Railsを経由してストリーミングでダウンロードするよりも、ApacheなどのWebサーバーから直接ファイルをダウンロードする方が遥かに効率が高く、しかもRailsスタック全体を経由する不必要なリクエストを受け付けずに済みます。

### RESTfulなダウンロード

`send_data`だけでも問題なく利用できますが、真にRESTfulなアプリケーションを作ろうとしているのであれば、ファイルダウンロード専用にわざわざアクションを作成する必要は通常ありません。RESTという用語においては、上の例で使われているPDFファイルのようなものは、クライアントリソースを別の形で表現したものであると見なされます。Railsには、これに基づいた「RESTfulダウンロード」を簡単に実現するための洗練された手段も用意されています。PDFダウンロードをストリーミングとして扱わず、`show`アクションの一部として扱うように上の例を変更したものを以下に示します。

```ruby
class ClientsController < ApplicationController
  # ユーザーはリソース受信時にHTMLまたはPDFをリクエストできる
  def show
    @client = Client.find(params[:id])

    respond_to do |format|
      format.html
      format.pdf { render pdf: generate_pdf(@client) }
    end
  end
end
```

なお、この例が実際に動作するには、RailsのMIME typeにPDFを追加する必要があります。これを行なうには、`config/initializers/mime_types.rb`に以下を追加します。

```ruby
Mime::Type.register "application/pdf", :pdf
```

NOTE: Railsの設定ファイルは、起動時にしか読み込まれません(app/以下のファイルのようにリクエストごとに読み出されたりしません)。上の設定変更を反映するサーバーを再起動する必要があります。

これで、以下のようにURLに".pdf"を追加するだけでPDF版のclientを取得できます。

```bash
GET /clients/1.pdf
```

### 任意のデータをライブストリーミングする

Railsは、ファイル以外のものもストリーミング送信できます。実は`response`オブジェクトに含まれるものなら何でもストリーミング送信できます。`ActionController::Live`モジュールを使うと、ブラウザとの永続的な接続を作成することができます。これにより、いつでも好きなタイミングで任意のデータをブラウザに送信することができます。

#### ライブストリーミングを組み込む

コントローラクラスの内部に`ActionController::Live`を組み込むと、そのコントローラのすべてのアクションでデータをストリーミングできるようになります。このモジュールを以下のようにmixin(訳注: `include`で利用すること)できます。

```ruby
class MyController < ActionController::Base
  include ActionController::Live

  def stream
    response.headers['Content-Type'] = 'text/event-stream'
    100.times {
      response.stream.write "hello world\n"
      sleep 1
    }
  ensure
    response.stream.close
  end
end
```

上のコードでは、ブラウザとの間に永続的な接続を確立し、1秒おきに`"hello world\n"`を100個ずつ送信しています。

上の例には注意点がいくつもあります。レスポンスのストリームは確実に閉じる必要があります。ストリームを閉じ忘れると、ソケットが永久に開いたままになってしまいます。レスポンスストリームへの書き込みを行う前に、Content-Typeを`text/event-stream`に設定する必要もあります。その理由は、レスポンスがコミットされてしまうと (`response.committed?`が「truthy」な値を返したとき)、以後ヘッダーに書き込みできないからです。これは、レスポンスストリームに対して`write`または`commit`を行った場合に発生します。

#### 使用例

今あなたはカラオケマシンを開発中です。1人のユーザーが特定の曲の歌詞を見たいと思っています。それぞれの`Song`には特定の数の行があり、その行1つ1つに「曲が終わるまで後何拍あるか」を表す`num_beats`が記入されています。

歌詞を「カラオケスタイル」でユーザーに表示したいので、直前の歌詞を歌い終わってから次の歌詞を表示することになります。そこで、以下のように`ActionController::Live`を使います。

```ruby
class LyricsController < ActionController::Base
  include ActionController::Live

  def show
    response.headers['Content-Type'] = 'text/event-stream'
    song = Song.find(params[:id])

    song.each do |line|
      response.stream.write line.lyrics
      sleep line.num_beats
    end
  ensure
    response.stream.close
  end
end
```

上のコードでは、お客さんが直前の歌詞を歌い終わった場合にのみ、次の歌詞を送信しています。

#### ストリーミングを行う場合の考慮事項

任意のデータをストリーミング送信できることは、きわめて強力なツールとなります。これまでの例でご紹介したように、好きな時に好きなものをレスポンスストリームに流すことができます。ただし、以下の点についてご注意ください。

* レスポンスストリームが1つ作成されるたびに新しいスレッドが作成され、元のスレッドからスレッドローカルな変数がコピーされます。スレッドローカルな変数が増えすぎるとパフォーマンスに悪影響が生じます。また、スレッド数が多すぎても同様にパフォーマンスが低下します。
* レスポンスストリームを閉じることに失敗すると、対応するソケットが永久に開いたままになってしまいます。レスポンスストリームを使う場合は、`close`が確実に呼び出されるようにしてください。
* WEBrickサーバーはすべてのレスポンスをバッファリングするため、`ActionController::Live`をインクルードしても動作しません。このため、レスポンスを自動的にバッファリングしないWebサーバーを使う必要があります。

ログをフィルタする
-------------

Railsのログファイルは、環境ごとに`log`フォルダの下に出力されます。デバッグ時にアプリケーションで何が起こっているかをログで確認できると非常に便利なのですが、その一方で本番のアプリケーションでは顧客のパスワードのような重要な情報をログファイルに出力したくないこともあります。

### パラメータをフィルタする

Railsアプリケーションの設定ファイル`config.filter_parameters`に、特定のリクエストパラメータをログ出力時にフィルタする設定を追加することができます。フィルタされたパラメータはログ内で`[FILTERED]`という文字に置き換えられます。

```ruby
config.filter_parameters << :password
```

NOTE: 渡されるパラメータは、正規表現の「部分マッチ」によってフィルタされる点にご注意ください。Railsは適切なイニシャライザ（`initializers/filter_parameter_logging.rb`）にデフォルトで`:password`を追加し、アプリケーションの典型的な`password`パラメータや`password_confirmation`パラメータについて配慮します。

### リダイレクトをフィルタする

アプリケーションからのリダイレクト先URLのいくつかをログに出力したくない場合があります。
設定の`config.filter_redirect`オプションを使って、リダイレクト先をログに出力しないようにできます。

```ruby
config.filter_redirect << 's3.amazonaws.com'
```

フィルタしたいリダイレクト先は、文字列、正規表現、または両方を含む配列で指定できます。

```ruby
config.filter_redirect.concat ['s3.amazonaws.com', /private_path/]
```

マッチしたURLはログで`[FILTERED]`という文字に置き換えられます。

Rescue
------

どんなアプリケーションにもどこかにバグが潜んでいたり、適切に扱う必要のある例外をスローすることがあるものです。たとえば、データベースに既に存在しなくなったリソースに対してユーザーがアクセスした場合、Active Recordは`ActiveRecord::RecordNotFound`例外をスローします。

Railsのデフォルトの例外ハンドリングでは、例外の種類にかかわらず「500 Server Error」を表示します。リクエストがローカルのブラウザから行われた場合、詳細なトレースバックや追加情報が表示されますので、問題点を把握して対応することができます。リクエストがリモートブラウザの場合、Railsは「500 Server Error」というメッセージだけをユーザーに表示したり、ルーティングやレコードがない場合は「404 Not Found」と表示したりします。このままではあまりにそっけないので、エラーのキャッチ方法やユーザーへの表示方法をカスタマイズしたくなるものです。Railsアプリケーションでは、例外ハンドリングをさまざまなレベルで実行できます。

### デフォルトの500・404テンプレート

デフォルトでは、本番のRailsアプリケーションは404または500エラーメッセージを表示します（development環境の場合はあらゆるunhandled exceptionが発生します）。これらのメッセージは`public`フォルダ以下に置かれている静的なHTMLファイルです。それぞれ`404.html`および`500.html`という名前です。これらのファイルをカスタマイズして、情報を追加したりスタイルを整えたりできます。ただし、これらはあくまで静的なHTMLファイルなので、ERBやSCSSやCoffeeScriptはレイアウトに使えません。

### `rescue_from`

もう少し洗練された方法でエラーをキャッチしたい場合は、`rescue_from`を使えます。これは、特定の種類または複数の種類の例外を1つのコントローラ全体およびそのサブクラスで扱えるようにするものです。

`rescue_from`命令でキャッチできる例外が発生すると、ハンドラに例外オブジェクトが渡されます。このハンドラはメソッドか、`:with`オプション付きで渡された`Proc`オブジェクトのいずれかです。明示的に`Proc`オブジェクトを使う代りに、ブロックを直接使うこともできます。

`rescue_from`を使ってすべての`ActiveRecord::RecordNotFound`エラーを奪い取り、処理を行なう方法を以下に示します。

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :record_not_found

  private

    def record_not_found
      render plain: "404 Not Found", status: 404
    end
end
```

以前よりも洗練されましたが、もちろんこの例のままではエラーハンドリングは何も改良されていません。しかしこのようにすべての例外をキャッチできれば、後は好きなようにカスタマイズできます。たとえば、カスタム例外クラスを作成して、ユーザーがアクセス権を持たないアプリケーションの特定の箇所にアクセスした場合に例外をスローすることもできます。

```ruby
class ApplicationController < ActionController::Base
  rescue_from User::NotAuthorized, with: :user_not_authorized

  private

    def user_not_authorized
      flash[:error] = "You don't have access to this section."
      redirect_back(fallback_location: root_path)
    end
end

class ClientsController < ApplicationController
  # ユーザーがクライアントにアクセスする権限を持っているかどうかをチェックする
  before_action :check_authorization

  # このアクション内で認証周りを心配する必要がない
  def edit
    @client = Client.find(params[:id])
  end

  private

    # ユーザーが認証されていない場合は単に例外をスローする
    def check_authorization
      raise User::NotAuthorized unless current_user.admin?
    end
end
```

WARNING: `rescue_from`に`Exception`や`StandardError`を指定すると、Railsでの正しい例外ハンドリングが阻害されて深刻な副作用が生じる可能性があります。よほどの理由がない限り、この指定はおすすめできません。

NOTE: `ActiveRecord::RecordNotFound`エラーは、production環境ではすべて404エラーページが表示されます。この振る舞いをカスタマイズする必要がない限り、開発者がこのエラーを扱う必要はありません。

NOTE: `ApplicationController`クラスでは一部の例外についてrescueできないものがあります。その理由は、コントローラが初期化されてアクションが実行される前に発生する例外があるからです。

HTTPSプロトコルを強制する
--------------------

コントローラとのやりとりがHTTPSのみで行われるようにしたい場合は、環境設定の`config.force_ssl`で`ActionDispatch::SSL`ミドルウェアを有効にすることで行うべきです。
