Active Storage の概要
=====================

このガイドはActive Recordモデルにファイルを添付する方法について説明します。

このガイドを読むと下記の内容が理解できるでしょう。

* 1つまたは複数のファイルを1つのレコードに添付する方法。
* 添付ファイルを消す方法
* 添付ファイルへのリンク方法
* バリアントを利用して画像を変形する方法
* PDFや動画に代表されるような非画像ファイルの生成方法
* あなたのアプリケーションサーバーを介して、ブラウザからストレージサービスに直接ファイルをアップロードする方法
* テスト中に保存されたファイルをクリーンアップする方法
* 追加のストレージサービスをサポートするための実装方法

--------------------------------------------------------------------------------


Active Storageについて
-----------------------

Active Storageは、Amazon S3、Google Cloud Storage、Microsoft Azure Storageなどのクラウドストレージサービスへのファイルのアップロードや、ファイルをActive Recordオブジェクトにアタッチする機能を提供します。
development環境とtest環境向けのローカルディスクベースのサービスを利用できるようになっており、ファイルを下位のサービスにミラーリングしてバックアップや移行に用いることもできます。

Active Storageを利用すると、アプリケーションにアップロードした画像の変形や、PDFや動画といった画像以外のアップロードファイルの画像表示の生成、任意のファイルからのメタデータ抽出が可能になります。

### 要件

Active Storageの多くの機能は、Railsによってインストールされないサードパーティソフトウェアに依存しているため、別途インストールが必要です。

* [libvips](https://github.com/libvips/libvips) v8.6以降または[ImageMagick](https://imagemagick.org/index.php): 画像解析や画像変形用
* [ffmpeg](http://ffmpeg.org/) v3.4以降: 動画や音声の解析や動画プレビュー用
* [poppler](https://poppler.freedesktop.org/)または[muPDF](https://mupdf.com/): PDFプレビュー用

画像分析や画像変形のために`image_processing` gemも必要です。`Gemfile`の`image_processing` gemをコメント解除するか、必要に応じて追加します。

```ruby
gem "image_processing", ">= 1.2"
```

TIP: ImageMagickは、libvipsに比べて知名度が高く普及も進んでいます。しかしlibvipsは[10倍高速かつメモリ消費も1/10です](https://github.com/libvips/libvips/wiki/Speed-and-memory-use)。JPEGファイルの場合、`libjpeg-dev`を`libjpeg-turbo-dev`に置き換えると[2〜7倍高速](https://libjpeg-turbo.org/About/Performance)になります。

WARNING: サードパーティのソフトウェアをインストールして使う前に、そのソフトウェアのライセンスを読んで理解しておきましょう。特にMuPDFはAGPLでライセンスされており、利用目的によっては商用ライセンスが必要です。

## セットアップ

Active Storageは、アプリケーションのデータベースで `active_storage_blobs`、`active_storage_variant_records`、`active_storage_attachments`という名前の3つのテーブルを使います。
新規アプリケーション作成した後（または既存のアプリケーションをRails 5.2にアップグレードした後）に、`rails active_storage:install`を実行して、これらのテーブルを作成するmigrationファイルを作成します。
migrationファイルを実行するには`rails db:migrate`をお使いください。

WARNING: `active_storage_attachments`は、使うモデルのクラス名を保存するポリモーフィックjoinテーブルです。モデルのクラス名を変更した場合は、このテーブルに対してマイグレーションを実行して背後の`record_type`をモデルの新しいクラス名に更新する必要があります。

WARNING: モデルの主キーに整数値ではなくUUIDを使っている場合は、生成されるマイグレーションファイルの`active_storage_attachments.record_id`と`active_storage_variant_records.id`のカラム型も変更する必要があります。

Active Storageのサービスは`config/storage.yml`で宣言します。アプリケーションが使うサービスごとに、名前と必要な構成を指定します。
次の例では、`local`、`test`、`amazon`という3つのサービスを宣言しています。

```yaml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

test:
  service: Disk
  root: <%= Rails.root.join("tmp/storage") %>

amazon:
  service: S3
  access_key_id: ""
  secret_access_key: ""
  bucket: ""
  region: "" # e.g. 'us-east-1'
```

利用するサービスをActive Storageに認識させるには、`Rails.application.config.active_storage.service`を設定します。
使うサービスは環境ごとに異なることもあるため、この設定を環境ごとに行うことをおすすめします。前述したローカルDiskサービスをdevelopment環境で使うには、`config/environments/development.rb`に以下を追加します。

```ruby
# ファイルをローカルに保存する
config.active_storage.service = :local
```

production環境でAmazon S3を利用するには、`config/environments/production.rb`に以下を追加します。

```ruby
# ファイルをAmazon S3に保存する
config.active_storage.service = :amazon
```

テスト時にテストサービスを利用するには、`config/environments/test.rb`に以下を追加します。

```ruby
# ローカルファイルシステム上のアップロード済みファイルを一時ディレクトリに保存する
config.active_storage.service = :test
```

内蔵されているサービスアダプタ(`Disk`や`S3`など)およびそれらに必要な設定について、詳しくは後述します。

NOTE: 環境固有の設定ファイルが優先されます。たとえばproduction環境では、`config/storage/production.yml`ファイルが存在すれば`config/storage.yml`ファイルよりも優先されます。

productionのデータ喪失リスクをさらに軽減するために、以下のようにバケット名に`Rails.env`を使うことをおすすめします。

```yaml
amazon:
  service: S3
  # ...
  bucket: your_own_bucket-<%= Rails.env %>

google:
  service: GCS
  # ...
  bucket: your_own_bucket-<%= Rails.env %>

azure:
  service: AzureStorage
  # ...
  container: your_container_name-<%= Rails.env %>
```

### Diskサービス

Diskサービスは`config/storage.yml`で宣言します。

```yaml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>
```

### Amazon S3サービス（およびS3互換API）

S3サービスは`config/storage.yml`で宣言します。

```yaml
amazon:
  service: S3
  access_key_id: ""
  secret_access_key: ""
  region: ""
  bucket: ""
```

クライアントやアップロードのオプションも指定できます。

```yaml
amazon:
  service: S3
  access_key_id: ""
  secret_access_key: ""
  region: ""
  bucket: ""
  http_open_timeout: 0
  http_read_timeout: 0
  retry_limit: 0
  upload:
    server_side_encryption: "" # 'aws:kms'または'AES256'
```

TIP: HTTPタイムアウトやリトライ上限数はアプリケーションに適した値を設定しましょう。特定の障害シナリオでは、デフォルトのAWSクライアント設定によってコネクションが数分間保持されてしまい、リクエストの待ち行列が発生する可能性があります。

`Gemfile`に[`aws-sdk-s3`](https://github.com/aws/aws-sdk-ruby) gemを追加します。

```ruby
gem "aws-sdk-s3", require: false
```

NOTE: Active Storageのコア機能では、`s3:ListBucket`、`s3:PutObject`、`s3:GetObject`、`s3:DeleteObject`という4つのパーミッションが必要です。[パブリックアクセス](#パブリックアクセス)の場合は`s3:PutObjectAcl`も必要です。ACLの設定といったアップロードオプションを追加で設定した場合は、この他にもパーミッションが必要になることがあります。

NOTE: 環境変数、標準SDKの設定ファイル、プロファイル、IAMインスタンスのプロファイルやタスクロールを使いたい場合は、上述の`access_key_id`、`secret_access_key`、`region`を省略できます。Amazon S3サービスでは、[AWS SDK documentation](https://docs.aws.amazon.com/sdk-for-ruby/v3/developer-guide/setup-config.html)に記載されている認証オプションをすべてサポートします。

DigitalOcean SpacesなどのS3互換オブジェクトストレージAPIに接続するには、`endpoint`を指定します。

```yaml
digitalocean:
  service: S3
  endpoint: https://nyc3.digitaloceanspaces.com
  access_key_id: ...
  secret_access_key: ...
  # ...その他のオプション
```

この他にもさまざまなオプションが利用できます。[AWS S3 Client](https://docs.aws.amazon.com/sdk-for-ruby/v3/api/Aws/S3/Client.html#initialize-instance_method)ドキュメントを参照してください。


### Microsoft Azure Storageサービス

Azure Storageサービスは`config/storage.yml`で宣言します。

``` yaml
azure:
  service: AzureStorage
  storage_account_name: ""
  storage_access_key: ""
  container: ""
```

`Gemfile`に[`azure-storage-blob`](https://github.com/Azure/azure-storage-ruby) gemを追加します。

``` ruby
gem "azure-storage-blob", require: false
```

### Google Cloud Storageサービス

Google Cloud Storageサービスは`config/storage.yml`で宣言します。

```yaml
google:
  service: GCS
  credentials: <%= Rails.root.join("path/to/keyfile.json") %>
  project: ""
  bucket: ""
```

keyfileパスの代わりに、credentialのハッシュを渡すこともできます。

```yaml
google:
  service: GCS
  credentials:
    type: "service_account"
    project_id: ""
    private_key_id: <%= Rails.application.credentials.dig(:gcs, :private_key_id) %>
    private_key: <%= Rails.application.credentials.dig(:gcs, :private_key).dump %>
    client_email: ""
    client_id: ""
    auth_uri: "https://accounts.google.com/o/oauth2/auth"
    token_uri: "https://accounts.google.com/o/oauth2/token"
    auth_provider_x509_cert_url: "https://www.googleapis.com/oauth2/v1/certs"
    client_x509_cert_url: ""
  project: ""
  bucket: ""
```

アップロードされたアセットに設定するCache-Controlメタデータをオプションで提供できます。

```yaml
google:
  service: GCS
  ...
  cache_control: "public, max-age=3600"
```

URLに署名する場合に、`credentials`の代わりに[IAM](https://cloud.google.com/storage/docs/access-control/signed-urls#signing-iam)をオプションで利用できます。これは、GKEアプリケーションをWorkload Identityで認証する場合に便利です。詳しくはGoogle Cloudのブログ記事『[Introducing Workload Identity: Better authentication for your GKE applications](https://cloud.google.com/blog/products/containers-kubernetes/introducing-workload-identity-better-authentication-for-your-gke-applications)』を参照してください。

```yaml
google:
  service: GCS
  ...
  iam: true
```

オプションで、URLに署名するときに特定のGSAを使います。IAMを使う場合は、GSAのメールを受け取るために[メタデータサーバー](https://cloud.google.com/compute/docs/storing-retrieving-metadata)にアクセスしますが、このメタデータサーバーは常に存在するとは限らず（ローカルテスト時など）、デフォルト以外のGSAを使いたい場合もあります。

```yaml
google:
  service: GCS
  ...
  iam: true
  gsa_email: "foobar@baz.iam.gserviceaccount.com"
```

`Gemfile`に[`google-cloud-storage`](https://github.com/GoogleCloudPlatform/google-cloud-ruby/tree/master/google-cloud-storage) gemを追加します。

``` ruby
gem "google-cloud-storage", "~> 1.11", require: false
```

### ミラーサービス

ミラーサービスを定義すると、複数のサービスを同期できます。ミラーサービスは、複数の下位サービスにアップロードや削除をレプリケーションします。

ミラーサービスは、production環境でサービス間の移行期で一時的に利用することを目的としています。新しいサービスへのミラーリングを開始し、既存のファイルを古いサービスから新しいサービスにコピーしてから、新しいサービスに全面的に移行できます。

NOTE: ミラーリングはアトミックではありません。プライマリサービスでアップロードに成功しても、サブサービスでは失敗する可能性があります。新しいサービスを開始する前に、すべてのファイルがコピー完了していることを確認してください。

上で説明したように、ミラーリングするサービスをそれぞれ定義します。ミラーサービスを定義するときは以下のように名前で参照します。

``` yaml
s3_west_coast:
  service: S3
  access_key_id: ""
  secret_access_key: ""
  region: ""
  bucket: ""

s3_east_coast:
  service: S3
  access_key_id: ""
  secret_access_key: ""
  region: ""
  bucket: ""

production:
  service: Mirror
  primary: s3_east_coast
  mirrors:
    - s3_west_coast
```

すべてのセカンダリサービスがアップロードを受信しますが、ダウンロードは常にプライマリサービスで行われます。

ミラーサービスはダイレクトアップロードと互換性があります。新しいファイルはプライマリサービスに直接アップロードされます。ダイレクトアップロードされたファイルをレコードにアタッチすると、バックグラウンドジョブがキューに入れられてセカンダリサービスにコピーされます。

### パブリックアクセス

Active Storageは、デフォルトでサービスにプライベートアクセスすることを前提としています。つまり、blobでは単一用途の署名済みURLを生成するということです。blobを一般公開したい場合は、アプリの `config/storage.yml`で以下のように`public: true`を指定します。

```yaml
gcs: &gcs
  service: GCS
  project: ""

private_gcs:
  <<: *gcs
  credentials: <%= Rails.root.join("path/to/private_keyfile.json") %>
  bucket: ""

public_gcs:
  <<: *gcs
  credentials: <%= Rails.root.join("path/to/public_keyfile.json") %>
  bucket: ""
  public: true
```

バケットがパブリックアクセス用に適切に設定されていることを必ず確認してください。ストレージサービスでパブリックな読み取りパーミッションを有効にする方法については、[Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/block-public-access-bucket.html)、[Google Cloud Storage](https://cloud.google.com/storage/docs/access-control/making-data-public#buckets)、[Microsoft Azure](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-manage-access-to-resources#set-container-public-access-level-in-the-azure-portal)のドキュメントをそれぞれ参照してください。Amazon S3では`s3:PutObjectAcl`パーミッションも必要です。

既存のアプリケーションを`public: true`に変更する場合は、バケット内のあらゆるファイルが一般公開されて読み取り可能になっていることを確認してから切り替えてください。

ファイルをレコードに添付する
-----------------------

### `has_one_attached`

[`has_one_attached`][]マクロは、レコードとファイルの間に1対1のマッピングを設定します。レコード1件ごとに1個のファイルを添付できます。

たとえば、アプリケーションに`User`モデルがあるとします。各userにavatarを持たせたい場合は、以下のように`User`モデルを定義します。

```ruby
class User < ApplicationRecord
  has_one_attached :avatar
end
```

Rails 6.0以降を使う場合は、以下のようにモデルのジェネレータコマンドを実行できます。

```ruby
bin/rails generate model User avatar:attachment
```

以下のように書くことでavatar付きのuserを作成できます。

```erb
<%= form.file_field :avatar %>
```

```ruby
class SignupController < ApplicationController
  def create
    user = User.create!(user_params)
    session[:user_id] = user.id
    redirect_to root_path
  end

  private
    def user_params
      params.require(:user).permit(:email_address, :password, :avatar)
    end
end
```

既存のuserにavatarを添付するには[`avatar.attach`][Attached::One#attach]を呼び出します。

```ruby
user.avatar.attach(params[:avatar])
```

`avatar.attached?`で特定のuserがavatarを持っているかどうかを調べられます。

```ruby
user.avatar.attached?
```

特定の添付ファイルについてはデフォルトのサービスを上書きしたい場合があります。以下のように`service`オプションを指定すると、添付ファイルごとに特定のサービスを設定できます。

```ruby
class User < ApplicationRecord
  has_one_attached :avatar, service: :s3
end
```

生成される添付可能オブジェクトで`variant`メソッドを呼ぶと、添付ファイルごとに特定のバリアント（サイズ違いの画像）を設定できます。

```ruby
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize: "100x100"
  end
end
```

アバターのサムネイルバリアントを取得するには`avatar.variant(:thumb)`を呼び出します。

```erb
<%= image_tag user.avatar.variant(:thumb) %>
```

[`has_one_attached`]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/Model.html#method-i-has_one_attached
[Attached::One#attach]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/One.html#method-i-attach
[Attached::One#attached?]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/One.html#method-i-attached-3F

### `has_many_attached`

[`has_many_attached`][]マクロは、レコードとファイルの間に1対多の関係を設定します。レコード1件ごとに、多数の添付ファイルを添付できます。

たとえば、アプリケーションに`Message`モデルがあるとします。メッセージごとに多数の画像を持たせるには、次のような`Message`モデルを定義します.

```ruby
class Message < ApplicationRecord
  has_many_attached :images
end
```

以下のように書くことで、画像付きのメッセージを作成できます。

```ruby
class MessagesController < ApplicationController
  def create
    message = Message.create!(message_params)
    redirect_to message
  end

  private
    def message_params
      params.require(:message).permit(:title, :content, images: [])
    end
end
```

[`images.attach`][Attached::Many#attach]を呼び出すと、既存のメッセージに新しい画像を追加できます。

```ruby
@message.images.attach(params[:images])
```

あるメッセージに何らかの画像がアタッチされているかどうかを調べるには、[`images.attached?`][Attached::Many#attached?]を呼び出します。

```ruby
@message.images.attached?
```

デフォルトサービスの上書きは、`has_one_attached`と同様に`service`オプションで行えます。

```ruby
class Message < ApplicationRecord
  has_many_attached :images, service: :s3
end
```

特定のvariantの設定も、`has_one_attached`と同様に、生成される添付可能オブジェクトで`variant`メソッドを呼ぶことで行なえます。

```ruby
class Message < ApplicationRecord
  has_many_attached :images do |attachable|
    attachable.variant :thumb, resize: "100x100"
  end
end
```

[`has_many_attached`]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/Model.html#method-i-has_many_attached
[Attached::Many#attach]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/Many.html#method-i-attach
[Attached::Many#attached?]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/Many.html#method-i-attached-3F

### File/IO Objectsをアタッチする

HTTPリクエスト経由では配信されないファイルをアタッチする必要が生じる場合があります。たとえば、ディスク上で生成したファイルやユーザーが送信したURLからダウンロードしたファイルをアタッチしたい場合や、モデルのテストでfixtureファイルをアタッチしたい場合などが考えられます。これを行うには、以下のようにオープンIOオブジェクトとファイル名を1つ以上含むハッシュを渡します。

```ruby
@message.images.attach(io: File.open('/path/to/file'), filename: 'file.pdf')
```

可能であれば、`content_type:`も指定しておきましょう。Active Storageは、渡されたデータからファイルのcontent_typeの判定を試みますが、判定できない場合は指定のcontent_typeにフォールバックします。

```ruby
@message.images.attach(io: File.open('/path/to/file'), filename: 'file.pdf', content_type: 'application/pdf')
```

以下のように`content_type`に`identify: false`を渡すと、content_typeの推測をバイパスできます。

```ruby
@message.images.attach(
  io: File.open('/path/to/file'),
  filename: 'file.pdf',
  content_type: 'application/pdf',
  identify: false
)
```

`content_type:`を指定せず、Active Storageがファイルのcontent_typeを自動的に判別できない場合は、デフォルトで`application/octet-stream`が設定されます。

ファイルを削除する
-----------------------------

添付ファイルをモデルから削除するには、添付ファイルに対して[`purge`][Attached::One#purge]を呼び出します。
Active Jobを使うようにアプリケーションが設定されている場合は、バックグラウンドで削除を実行できます。purgeすると、blobとファイルがストレージサービスから削除されます。

```ruby
# avatarと実際のリソースファイルを同期的に破棄します。
user.avatar.purge

# Active Jobを介して、関連付けられているモデルと実際のリソースファイルを非同期で破棄します。
user.avatar.purge_later
```

[Attached::One#purge]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/One.html#method-i-purge
[Attached::One#purge_later]: https://api.rubyonrails.org/classes/ActiveStorage/Attached/One.html#method-i-purge_later

ファイルを配信する
-------------

Active Storageは「リダイレクト」と「プロキシ」という2種類のファイル配信をサポートしています。

WARNING: Active Storageのすべてのコントローラは、デフォルトでpublicアクセスできます。生成されるURLは推測が困難ですが、設計上は永続的なURLになります。ファイルをより高度なレベルで保護する必要がある場合は、[認証済みコントローラ](#認証済みコントローラ)の実装を検討してください。

### リダイレクトモード

[`url_for`][ActionView::RoutingUrlFor#url_for]ビューヘルパーにblobを渡すとblobのパーマネントURLを生成できます。生成されるURLでは、そのblogの[`RedirectController`][`ActiveStorage::Blobs::RedirectController`]にルーティングされる[`signed_id`][ActiveStorage::Blob#signed_id]が使われます。

```ruby
url_for(user.avatar)
# => /rails/active_storage/blobs/:signed_id/my-avatar.png
```

`RedirectController`は、サービスの実際のエンドポイントにリダイレクトします。この間接参照によってサービスURLと実際のURLが切り離され、たとえば添付ファイルを別サービスにミラーリングして可用性を高めることが可能になります。リダイレクトのHTTP有効期限は5分です。

ダウンロードリンクを作成するには、`rails_blob_path`や`rails_blob_url`ヘルパーを使います。このヘルパーでは[`Content-Disposition`ヘッダー](https://developer.mozilla.org/ja/docs/Web/HTTP/Headers/Content-Disposition)を指定できます。

```ruby
rails_blob_path(user.avatar, disposition: "attachment")
```

WARNING: XSS（[クロスサイトスクリプティング](https://ja.wikipedia.org/wiki/%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%B5%E3%82%A4%E3%83%88%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%97%E3%83%86%E3%82%A3%E3%83%B3%E3%82%B0)）攻撃を防ぐため、Active Storageは特定の種類のファイルについて`Content-Disposition`ヘッダーを強制的に"attachment"に設定します。この振る舞いを変更する場合は、[Active Storageの設定方法](configuring.html#active-storageを設定する)で利用可能な設定オプションを参照してください。

バックグラウンドジョブやcronジョブなど、コントローラやビューのコンテキストの外でリンクを作成する必要がある場合は、以下のような方法で`rails_blob_path`にアクセスできます。

```ruby
Rails.application.routes.url_helpers.rails_blob_path(user.avatar, only_path: true)
```

[ActionView::RoutingUrlFor#url_for]: https://api.rubyonrails.org/classes/ActionView/RoutingUrlFor.html#method-i-url_for
[ActiveStorage::Blob#signed_id]: https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-signed_id

### プロキシモード

ファイルをプロキシすることもオプションで可能です。これは、リクエストに応じてアプリケーションサーバーがファイルデータをストレージサービスからダウンロードするということです。プロキシモードは、CDN上のファイルを配信する場合に便利です。

以下のように、Active Storageがデフォルトでプロキシを利用するように設定できます。

```ruby
# config/initializers/active_storage.rb
Rails.application.config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

特定の添付ファイルを明示的にプロキシしたい場合は、`rails_storage_proxy_path`や`rails_storage_proxy_url`という形式のURLヘルパーを利用できます。

```erb
<%= image_tag rails_storage_proxy_path(@user.avatar) %>
```

#### Active Storageの手前にCDNを配置する

また、Active Storageの添付ファイルにCDNを使うには、URLをプロキシモードで生成してアプリで提供し、CDNで追加設定を行わずに添付ファイルがCDNでキャッシュされるようにする必要があります。Active Storageのデフォルトのプロキシコントローラは、レスポンスをキャッシュするようにCDNに指示するHTTPヘッダーを設定するので、すぐに利用できます。

また、生成されるURLがアプリのホストではなくCDNのホストを使うようにする必要もあります。これを行う方法は複数ありますが、一般にはアプリの`config/routes.rb`ファイルを調整して、添付ファイルやそのバリエーションのURLが正しく生成されるようにします。たとえば以下を追加できます。

```ruby
# config/routes.rb
direct :cdn_image do |model, options|
  if model.respond_to?(:signed_id)
    route_for(
      :rails_service_blob_proxy,
      model.signed_id,
      model.filename,
      options.merge(host: ENV['CDN_HOST'])
    )
  else
    signed_blob_id = model.blob.signed_id
    variation_key  = model.variation.key
    filename       = model.blob.filename

    route_for(
      :rails_blob_representation_proxy,
      signed_blob_id,
      variation_key,
      filename,
      options.merge(host: ENV['CDN_HOST'])
    )
  end
end
```

続いて以下のようにルーティングを生成します。

```erb
<%= cdn_image_url(user.avatar.variant(resize_to_limit: [128, 128])) %>
```

### 認証済みコントローラ

Active Storageのすべてのコントローラは、デフォルトでpublicアクセスできます。生成されるURLではプレーンな[`signed_id`][ActiveStorage::Blob#signed_id]が使われ、推測は困難ですが、URLは永続的です。blobのURLを知っている人であれば、 `ApplicationController`の`before_action`でログインを必須にしていてもblobのURLにアクセス可能です。より高度なレベルの保護が必要な場合は、[`ActiveStorage::Blobs::RedirectController`][]、[`ActiveStorage::Blobs::ProxyController`][]、[`ActiveStorage::Representations::RedirectController`][]、[`ActiveStorage::Representations::ProxyController`][]をベースに独自の認証済みコントローラを実装できます。

あるアカウントがアプリケーションのロゴにアクセスすることだけを許可するには、以下のようにします。

```ruby
# config/routes.rb
resource :account do
  resource :logo
end
```

```ruby
# app/controllers/logos_controller.rb
class LogosController < ApplicationController
  # ApplicationController経由で
  # AuthenticateとSetCurrentAccountをincludeする

  def show
    redirect_to Current.account.logo.url
  end
end
```

```erb
<%= image_tag account_logo_path %>
```

このとき、一般からアクセス可能なURLでファイルにアクセスされるのを防ぐために、以下のようにActive Storageのデフォルトルーティングを無効にするとよいでしょう。

```ruby
config.active_storage.draw_routes = false
```

[`ActiveStorage::Blobs::RedirectController`]: https://api.rubyonrails.org/classes/ActiveStorage/Blobs/RedirectController.html
[`ActiveStorage::Blobs::ProxyController`]: https://api.rubyonrails.org/classes/ActiveStorage/Blobs/ProxyController.html
[`ActiveStorage::Representations::RedirectController`]: https://api.rubyonrails.org/classes/ActiveStorage/Representations/RedirectController.html
[`ActiveStorage::Representations::ProxyController`]: https://api.rubyonrails.org/classes/ActiveStorage/Representations/ProxyController.html

ファイルをダウンロードする
-----------------

アップロードしたblobに対して処理を行う（別フォーマットへの変換など）必要が生じることがあります。`ActiveStorage::Blob#download`を用いてblobのバイナリデータをメモリに読み込めます。

```ruby
binary = user.avatar.download
```

場合によっては、blobをディスク上のファイルとしてダウンロードし、外部プログラム（ウイルススキャナーやメディアコンバーターなど）で処理できるようにしたいことがあります。ActiveStorageの[`open`][Blob#open]メソッドでblobをディスク上のtempfileにダウンロードできます。

```ruby
message.video.open do |file|
  system '/path/to/virus/scanner', file.path
  # ...
end
```

重要なのは、このファイルは`after_create`コールバックの時点ではアクセスできず、`after_create_commit`コールバックでのみアクセス可能になることです。

[Blob#download]: https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-download
[Blob#open]: https://api.rubyonrails.org/classes/ActiveStorage/Blob.html#method-i-open

ファイルを解析する
---------------

Active Storageは、Active Jobにジョブをキューイングしてアップロードされるとファイルを解析します。解析されたファイルのメタデータハッシュには、`analyzed: true`などの追加情報が保存されます。[`analyzed?`][]を呼び出すことで、blobが解析済みかどうかをチェックできます。

画像解析では、幅（`width`）と高さ（`height`）の属性が提供されます。

動画解析では、幅（`width`）と高さ（`height`）のほかに、再生時間（`duration`）、角度（`angle`）、アスペクト比（ `display_aspect_ratio`）、動画の存在を表す`video`（boolean）と音声の存在を表す`audio`（boolean）も提供されます。

音声解析では、再生時間（`duration`）とビットレート（`bit_rate`）の属性が提供されます。

[`analyzed?`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob/Analyzable.html#method-i-analyzed-3F

画像、動画、PDFを表示する
---------------

Active Storageは、ファイルのさまざまな表示方法をサポートしています。

添付ファイルで[`representation`][]を呼び出すと、画像バリアントの表示や、動画やPDFのプレビュー表示が行えます。

[`representable?`]を呼び出せば、`representation`を呼び出す前に添付ファイルが表示可能かどうかをチェックできます。

ファイルフォーマットによってはActive Storageですぐにプレビューを表示できないものもあるので（Wordドキュメントなど）、`representable?`が`false`を返す場合は、ファイルを[リンク形式でダウンロード](#ファイルを配信する)させるとよいでしょう。

```erb
<ul>
  <% @message.files.each do |file| %>
    <li>
      <% if file.representable? %>
        <%= image_tag file.representation(resize_to_limit: [100, 100]) %>
      <% else %>
        <%= link_to rails_blob_path(file, disposition: "attachment") do %>
          <%= image_tag "placeholder.png", alt: "Download file" %>
        <% end %>
      <% end %>
    </li>
  <% end %>
</ul>
```

`representation`の内部では、画像に対して`variant`メソッドを呼び出し、プレビュー可能なファイルであれば`preview`メソッドを呼び出します。これらのメソッドを直接呼ぶことも可能です。

[`representable?`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob/Representable.html#method-i-representable-3F
[`representation`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob/Representable.html#method-i-representation

### 遅延読み込みとイミディエイト読み込み

Active Storageは、デフォルトで表示処理を遅延します。

```ruby
image_tag file.representation(resize_to_limit: [100, 100])
```

上のコードで生成される`<img>`タグには、[`ActiveStorage::Representations::RedirectController`][]を指す`src`属性が追加されます。ブラウザがこのコントローラにリクエストを送信すると、リモートサービスへの`302`リダイレクトが返されます（[プロキシモード](#プロキシモード)の場合はファイルのコンテンツが返されます）。

ファイルを遅延読み込みすることで、[単一用途URL](#パブリックアクセス)のような機能を使っても最初のページ読み込みが遅くならないようにできます。

遅延読み込みはほとんどのケースに適しています。

画像をただちに表示するURLを生成したい場合は、以下のように`.processed.url`を呼び出せます。

```ruby
image_tag file.representation(resize_to_limit: [100, 100]).processed.url
```

Active Storageのバリアントトラッカーは、リクエストされた表示処理が以前行われていた場合にレコードをデータベースに保存することで、パフォーマンスを向上させます。つまり上のコードは、S3などのリモートサービスへのAPI呼び出しを1度だけ行い、バリアントが保存されると以後はそれを使います。バリアントトラッカーは自動的に実行されますが、`config.active_storage.track_variants`設定で無効にできます。

上のコード例を用いてひとつのページ内で多数の画像をレンダリングすると、バリアントレコードの読み込みで「N+1クエリ問題」が発生する可能性があります。N+1クエリ問題を避けるには、以下のように[`ActiveStorage::Attachment`][]で名前付きスコープをお使いください。

```ruby
message.images.with_all_variant_records.each do |file|
  image_tag file.representation(resize_to_limit: [100, 100]).processed.url
end
```

[`ActiveStorage::Representations::RedirectController`]: https://api.rubyonrails.org/classes/ActiveStorage/Representations/RedirectController.html
[`ActiveStorage::Attachment`]: https://api.rubyonrails.org/classes/ActiveStorage/Attachment.html

画像を変形する
----------------

画像を変形することで、画像を任意のサイズで表示できるようになります。

サイズ違いの画像を作成するには、添付ファイルで[`variant`][]を呼び出します。このメソッドには、バリアントプロセッサでサポートされている任意の変形処理を渡せます。

ブラウザがバリアントのURLにアクセスすると、Active Storageは元のblobを指定のフォーマットに遅延変形し、新しいサービスのある場所へリダイレクトします。

バリアントがリクエストされると、Active Storageは画像フォーマットに応じて自動的に変形処理を適用します。

1. content typeが可変（`config.active_storage.variable_content_types`の設定に基づく）で、Web画像を考慮しない場合（`config.active_storage.web_image_content_types`の設定に基づく）は、PNGに変換される。

2. `quality`が指定されていない場合は、その画像のデフォルトの画像品質がバリアントプロセッサで使われる。

Active StorageのデフォルトのバリアントプロセッサはMiniMagickですが、[Vips][]も指定可能です。Vipsに切り替えるには、`config/application.rb`に以下の設定を追加します。

```ruby
config.active_storage.variant_processor = :vips
```

MiniMagickとVipsの互換性は完全ではないため、MiniMagickを利用している既存のアプリケーションをVipsに移行すると、フォーマット固有のオプションを使っている場合は以下のように若干の変更が必要になります。

```erb
<!-- MiniMagick -->
<%= image_tag user.avatar.variant(resize_to_limit: [100, 100], format: :jpeg, sampling_factor: "4:2:0", strip: true, interlace: "JPEG", colorspace: "sRGB", quality: 80) %>

<!-- Vips -->
<%= image_tag user.avatar.variant(resize_to_limit: [100, 100], format: :jpeg, saver: { subsample_mode: "on", strip: true, interlace: true, quality: 80 }) %>
```

[`variant`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob/Representable.html#method-i-variant
[Vips]: https://www.rubydoc.info/gems/ruby-vips/Vips/Image

ファイルのプレビュー
-----------------------

画像でないファイルの中にはプレビューできるものもあります（画像として表示されます）。たとえば、動画ファイルの最初のフレームを抽出してプレビューできます。Active Storageでは、動画とPDFドキュメントについてすぐ使えるプレビュー機能をサポートしています。遅延生成されるプレビューへのリンクを作成するには、以下のように添付ファイルの[`preview`][]メソッドを使います。

```erb
<%= image_tag message.video.preview(resize_to_limit: [100, 100]) %>
```

別のフォーマットのサポートを追加するには、独自のプレビューアを追加します。詳しくは[`ActiveStorage::Preview`][]ドキュメントを参照してください。

[`preview`]: https://api.rubyonrails.org/classes/ActiveStorage/Blob/Representable.html#method-i-preview
[`ActiveStorage::Preview`]: https://api.rubyonrails.org/classes/ActiveStorage/Preview.html

ダイレクトアップロード
--------------------------

Active Storageは、付属のJavaScriptライブラリを用いて、クライアントからクラウドへのダイレクトアップロードをサポートします。

### 利用法

1. アプリケーションのJavaScriptバンドルに`activestorage.js`を追記します。

    アセットパイプラインを使う場合は以下のようにします。

    ```js
    //= require activestorage
    ```

    npmパッケージを使う場合は以下のようにします。

    ```js
    import * as ActiveStorage from "@rails/activestorage"
    ActiveStorage.start()
    ```

2. [`file_field`](form_helpers.html#ファイルのアップロード)に`direct_upload: true`を追加します。

    ```erb
    <%= form.file_field :attachments, multiple: true, direct_upload: true %>
    ```

    `FormBuilder`を使っていない場合は、以下のようにdata属性を直接追加します。

    ```erb
    <input type=file data-direct-upload-url="<%= rails_direct_uploads_url %>" />
    ```

3. サードパーティのストレージサービスにCORSを設定して、ダイレクトアップロードのリクエストを許可します。

4. 以上で完了です。アップロードはフォーム送信時に開始されます。

### CORS（Cross-Origin Resource Sharing）を設定する

サードパーティへのダイレクトアップロードを使えるようにするには、そのサービスで自分のアプリからのクロスオリジンリクエストを許可する必要があります。お使いのサービスのCORSドキュメントを参照してください。

* [S3](https://docs.aws.amazon.com/ja_jp/AmazonS3/latest/userguide/ManageCorsUsing.html)
* [Google Cloud Storage](https://cloud.google.com/storage/docs/configuring-cors)
* [Azure Storage](https://docs.microsoft.com/ja-jp/rest/api/storageservices/cross-origin-resource-sharing--cors--support-for-the-azure-storage-services)

以下を許可します。

* 自分のアプリがアクセスされるすべてのオリジン
* `PUT`リクエストメソッド
* 以下のヘッダー
  * `Origin`
  * `Content-Type`
  * `Content-MD5`
  * `Content-Disposition`（Azure Storageでは不要）
  * `x-ms-blob-content-disposition`（Azure Storageのみ必要）
  * `x-ms-blob-type`（Azure Storageのみ必要）
  * `Cache-Control`（GCSでは`cache_control`が設定されている場合のみ必要）

Diskサービスはアプリのオリジンを共有するので、CORS設定は不要です。

#### 設定例: S3のCORS

```json
[
  {
    "AllowedHeaders": [
      "*"
    ],
    "AllowedMethods": [
      "PUT"
    ],
    "AllowedOrigins": [
      "https://www.example.com"
    ],
    "ExposeHeaders": [
      "Origin",
      "Content-Type",
      "Content-MD5",
      "Content-Disposition"
    ],
    "MaxAgeSeconds": 3600
  }
]
```

#### 設定例: Google Cloud StorageのCORS

```json
[
  {
    "origin": ["https://www.example.com"],
    "method": ["PUT"],
    "responseHeader": ["Origin", "Content-Type", "Content-MD5", "Content-Disposition"],
    "maxAgeSeconds": 3600
  }
]
```

#### 設定例: Azure StorageのCORS

```xml
<Cors>
  <CorsRule>
    <AllowedOrigins>https://www.example.com</AllowedOrigins>
    <AllowedMethods>PUT</AllowedMethods>
    <AllowedHeaders>Origin, Content-Type, Content-MD5, x-ms-blob-content-disposition, x-ms-blob-type</AllowedHeaders>
    <MaxAgeInSeconds>3600</MaxAgeInSeconds>
  </CorsRule>
</Cors>
```

### ダイレクトアップロードのJavascriptイベント

| イベント名 | イベントの対象 | イベントデータ（`event.detail`） | 説明 |
| --- | --- | --- | --- |
| `direct-uploads:start` | `<form>` | なし | ダイレクトアップロードフィールドのファイルを含むフォームが送信された。 |
| `direct-upload:initialize` | `<input>` | `{id, file}` | フォーム送信後のすべてのファイルにディスパッチされる。 |
| `direct-upload:start` | `<input>` | `{id, file}` | 直接アップロードが開始されている。 |
| `direct-upload:before-blob-request` | `<input>` | `{id, file, xhr}` | アプリケーションにダイレクトアップロードメタデータを要求する前。 |
| `direct-upload:before-storage-request` | `<input>` | `{id, file, xhr}` | ファイルを保存するリクエストを出す前。 |
| `direct-upload:progress` | `<input>` | `{id, file, progress}` | ファイルを保存する要求が進行中。 |
| `direct-upload:error` | `<input>` | `{id, file, error}` | エラーが発生した。 このイベントがキャンセルされない限り、`alert`が表示される。 |
| `direct-upload:end` | `<input>` | `{id, file}` | ダイレクトアップロードが終了した。 |
| `direct-uploads:end` | `<form>` | なし | すべてのダイレクトアップロードが終了した。 |

### 例

上記のイベントを用いて、アップロードの進行状況をプログレスバー表示できます。

![direct-uploads](https://user-images.githubusercontent.com/5355/28694528-16e69d0c-72f8-11e7-91a7-c0b8cfc90391.gif)

以下は、アップロードされたファイルをフォームに表示するコードです。

```js
// direct_uploads.js

addEventListener("direct-upload:initialize", event => {
  const { target, detail } = event
  const { id, file } = detail
  target.insertAdjacentHTML("beforebegin", `
    <div id="direct-upload-${id}" class="direct-upload direct-upload--pending">
      <div id="direct-upload-progress-${id}" class="direct-upload__progress" style="width: 0%"></div>
      <span class="direct-upload__filename"></span>
    </div>
  `)
  target.previousElementSibling.querySelector(`.direct-upload__filename`).textContent = file.name
})

addEventListener("direct-upload:start", event => {
  const { id } = event.detail
  const element = document.getElementById(`direct-upload-${id}`)
  element.classList.remove("direct-upload--pending")
})

addEventListener("direct-upload:progress", event => {
  const { id, progress } = event.detail
  const progressElement = document.getElementById(`direct-upload-progress-${id}`)
  progressElement.style.width = `${progress}%`
})

addEventListener("direct-upload:error", event => {
  event.preventDefault()
  const { id, error } = event.detail
  const element = document.getElementById(`direct-upload-${id}`)
  element.classList.add("direct-upload--error")
  element.setAttribute("title", error)
})

addEventListener("direct-upload:end", event => {
  const { id } = event.detail
  const element = document.getElementById(`direct-upload-${id}`)
  element.classList.add("direct-upload--complete")
})
```

以下のスタイルを追加します。

```css
/* direct_uploads.css */

.direct-upload {
  display: inline-block;
  position: relative;
  padding: 2px 4px;
  margin: 0 3px 3px 0;
  border: 1px solid rgba(0, 0, 0, 0.3);
  border-radius: 3px;
  font-size: 11px;
  line-height: 13px;
}

.direct-upload--pending {
  opacity: 0.6;
}

.direct-upload__progress {
  position: absolute;
  top: 0;
  left: 0;
  bottom: 0;
  opacity: 0.2;
  background: #0076ff;
  transition: width 120ms ease-out, opacity 60ms 60ms ease-in;
  transform: translate3d(0, 0, 0);
}

.direct-upload--complete .direct-upload__progress {
  opacity: 0.4;
}

.direct-upload--error {
  border-color: red;
}

input[type=file][data-direct-upload-url][disabled] {
  display: none;
}
```

### ライブラリやフレームワークとの統合

ダイレクトアップロード機能をJavaScriptフレームワークから利用したい場合や、ドラッグアンドドロップをカスタマイズしたい場合は、`DirectUpload`クラスを利用して行えます。選択したライブラリからファイルを1件受信したら、`DirectUpload`をインスタンス化してそのインスタンスの`create`メソッドを呼び出します。`create`には、アップロード完了時に呼び出すコールバックを1つ渡せます。

```js
import { DirectUpload } from "@rails/activestorage"

const input = document.querySelector('input[type=file]')

// ファイルドロップへのバインド: 親要素のondropか、
// Dropzoneなどのライブラリを使う
const onDrop = (event) => {
  event.preventDefault()
  const files = event.dataTransfer.files;
  Array.from(files).forEach(file => uploadFile(file))
}

// 通常のファイル選択へのバインド
input.addEventListener('change', (event) => {
  Array.from(input.files).forEach(file => uploadFile(file))
  // 選択されたファイルをここで入力からクリアしてもよい
  input.value = null
})

const uploadFile = (file) => {
  // フォームではfile_field direct_upload: trueが必要
  // （これでdata-direct-upload-url、
  // data-direct-upload-token、
  // data-direct-upload-attachment-nameを提供する）
  const url = input.dataset.directUploadUrl
  const token = input.dataset.directUploadToken
  const attachmentName = input.dataset.directUploadAttachmentName
  const upload = new DirectUpload(file, url, token, attachmentName)

  upload.create((error, blob) => {
    if (error) {
      // エラーハンドリングをここに書く
    } else {
      // 適切な名前のhidden inputをblob.signed_idの値とともにフォームに追加する
      // これによりblob idが通常のアップロードフローで転送される
      const hiddenField = document.createElement('input')
      hiddenField.setAttribute("type", "hidden");
      hiddenField.setAttribute("value", blob.signed_id);
      hiddenField.name = input.name
      document.querySelector('form').appendChild(hiddenField)
    }
  })
}
```

ファイルアップロードの進行状況をトラッキングする必要がある場合は、`DirectUpload`コンストラクタに5番目のパラメータを渡せます。`DirectUpload`はアップロード中にオブジェクトの`directUploadWillStoreFileWithXHR`メソッドを呼び出すので、以後はXHRで独自のプログレスハンドラをバインドできるようになります。

```js
import { DirectUpload } from "@rails/activestorage"

class Uploader {
  constructor(file, url, token, attachmentName) {
    this.upload = new DirectUpload(file, url, token, attachmentName, this)
  }

  upload(file) {
    this.upload.create((error, blob) => {
      if (error) {
        // エラーハンドリングをここに書く
      } else {
      // 適切な名前のhidden inputをblob.signed_idの値とともにフォームに追加する
      }
    })
  }

  directUploadWillStoreFileWithXHR(request) {
    request.upload.addEventListener("progress",
      event => this.directUploadDidProgress(event))
  }

  directUploadDidProgress(event) {
    // event.loadedとevent.totalでプログレスバーを更新する
  }
}
```

NOTE: [ダイレクトアップロード](#ダイレクトアップロード)では、ファイルがアップロードされたにもかかわらずレコードにまったくアタッチされないことがあります。[アタッチされなかったアップロードを破棄する](#アタッチされなかったアップロードを破棄する)を参照してください。

テスト
-------------------------------------------

結合テストやコントローラのテストでファイルのアップロードをテストするには、[`fixture_file_upload`][]を使います。
Railsは、ファイルを他のパラメータと同様に扱います。

```ruby
class SignupController < ActionDispatch::IntegrationTest
  test "can sign up" do
    post signup_path, params: {
      name: "David",
      avatar: fixture_file_upload("david.png", "image/png")
    }

    user = User.order(:created_at).last
    assert user.avatar.attached?
  end
end
```

[`fixture_file_upload`]: https://api.rubyonrails.org/classes/ActionDispatch/TestProcess/FixtureFile.html

テスト中に作成したファイルを破棄する
-----------------------------------------------

#### システムテスト

システムテストでは、トランザクションをロールバックすることでテストデータをクリーンアップしますが、`destroy`はオブジェクトに対して呼び出されないため、添付ファイルはそのままでは決してクリーンアップされません。
添付ファイルを破棄したい場合は、`after_teardown`コールバックで行えます。このコールバックを実行すると、テスト中に作成されたすべてのコネクションを確実に完了するので、Active Storageでファイルが見つからないというエラーは表示されなくなります。

```ruby
class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  # ...
  def after_teardown
    super
    FileUtils.rm_rf(ActiveStorage::Blob.service.root)
  end
  # ...
end
```

[並列テスト][]と`DiskService`を利用している場合は、Active Storage用の独自のフォルダをプロセスごとに設定する必要があります。これにより、`teardown`コールバックが呼ばれたときに、関連するプロセスのファイルだけが削除されるようになります。

```ruby
class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  # ...
  parallelize_setup do |i|
    ActiveStorage::Blob.service.root = "#{ActiveStorage::Blob.service.root}-#{i}"
  end
  # ...
end
```

システムテストで、添付ファイルを持つモデルの削除を検証し、かつActive Jobを使っている場合は、test環境で以下のようにインラインキューアダプタを使うように設定してください（purgeジョブが未来のいつかではなく、ただちに実行されるようにするため）。

```ruby
# インラインジョブ処理を用いてpurgeをただちに行う
config.active_job.queue_adapter = :inline
```

[並列テスト]: https://railsguides.jp/testing.html#%E4%B8%A6%E5%88%97%E3%83%86%E3%82%B9%E3%83%88

#### 結合テスト

システムテストの場合と同様、結合テスト（integration test）の場合もアップロードしたファイルの自動クリーンアップは行われません。アップロードしたファイルをクリーンアップしたい場合は、`teardown`コールバックで行えます。

```ruby
class ActionDispatch::IntegrationTest
  def after_teardown
    super
    FileUtils.rm_rf(ActiveStorage::Blob.service.root)
  end
end
```

[並列テスト][]と`DiskService`を利用している場合は、Active Storage用の独自のフォルダをプロセスごとに設定する必要があります。これにより、`teardown`コールバックが呼ばれたときに、関連するプロセスのファイルだけが削除されるようになります。

```ruby
class ActionDispatch::IntegrationTest
  parallelize_setup do |i|
    ActiveStorage::Blob.service.root = "#{ActiveStorage::Blob.service.root}-#{i}"
  end
end
```

[並列テスト]: https://railsguides.jp/testing.html#%E3%83%91%E3%83%A9%E3%83%AC%E3%83%AB%E3%83%86%E3%82%B9%E3%83%88

### フィクスチャに添付ファイルを追加する

既存の[フィクスチャ][]に添付ファイルを追加できます。最初に、独立したストレージサービスを作成します。

```yml
# config/storage.yml

test_fixtures:
  service: Disk
  root: <%= Rails.root.join("tmp/storage_fixtures") %>
```

上の設定は、Active Storageにフィクスチャファイルの「アップロード」先を伝えるためのものなので、一時ディレクトリを使う必要があります。通常の`test`サービスと別のディレクトリを指定することで、フィクスチャファイルとテスト中にアップロードされるファイルが分けられます。

次にActive Storageクラスで使うフィクスチャファイルを作成します。

```yml
# active_storage/attachments.yml
david_avatar:
  name: avatar
  record: david (User)
  blob: david_avatar_blob
```

```yml
# active_storage/blobs.yml
david_avatar_blob: <%= ActiveStorage::FixtureSet.blob filename: "david.png", service_name: "test_fixtures" %>
```

次に、フィクスチャディレクトリ（デフォルトのパスは `test/fixtures/files`）に、`filename:`に対応するファイルを置きます。詳しくは[`ActiveStorage::FixtureSet`][]のドキュメントを参照してください。

セットアップがすべて完了したら、テストで添付ファイルにアクセスできるようになります。

```ruby
class UserTest < ActiveSupport::TestCase
  def test_avatar
    avatar = users(:david).avatar

    assert avatar.attached?
    assert_not_nil avatar.download
    assert_equal 1000, avatar.byte_size
  end
end
```

#### フィクスチャをクリーンアップする

テストでアップロードされたファイルは[各テストが終わるたびに](#テスト中に作成したファイルを破棄する)クリーンアップされますが、フィクスチャファイルのクリーンアップはテスト完了時に1度だけ行えば十分です。

並列テストを使っている場合は、`parallelize_teardown`を呼び出します。

```ruby
class ActiveSupport::TestCase
  # ...
  parallelize_teardown do |i|
    FileUtils.rm_rf(ActiveStorage::Blob.services.fetch(:test_fixtures).root)
  end
  # ...
end
```

並列テストを実行していない場合は、`Minitest.after_run`を使うか、利用しているテストフレームワークの同等なメソッド（RSpecの`after(:suite)`など）を使います。

```ruby
# test_helper.rb

Minitest.after_run do
  FileUtils.rm_rf(ActiveStorage::Blob.services.fetch(:test_fixtures).root)
end
```

[フィクスチャ]: testing.html#フィクスチャのしくみ
[`ActiveStorage::FixtureSet`]: https://api.rubyonrails.org/classes/ActiveStorage/FixtureSet.html

その他のクラウドサービスのサポートを実装する
---------------------------------

これら以外のクラウドサービスをサポートする必要がある場合は、サービスを実装する必要があります。
各サービスは、ファイルをアップロードしてクラウドにダウンロードするのに必要なメソッドを実装することで、[`ActiveStorage::Service`](https://api.rubyonrails.org/classes/ActiveStorage/Service.html)を拡張します 。

アタッチされなかったアップロードを破棄する
--------------------------

アップロードされたファイルがレコードにまったくアタッチされないことがあります。これは[ダイレクトアップロード](#ダイレクトアップロード)を使っている場合に発生する可能性があります。[`scope :unattached`](https://github.com/rails/rails/blob/8ef5bd9ced351162b673904a0b77c7034ca2bc20/activestorage/app/models/active_storage/blob.rb#L49)を使うことでアタッチされなかったレコードをクエリで調べられます。以下は[カスタムrakeタスク](command_line.html#カスタムrakeタスク)を使った例です。

```ruby
namespace :active_storage do
  desc "Purges unattached Active Storage blobs. Run regularly."
  task purge_unattached: :environment do
    ActiveStorage::Blob.unattached.where("active_storage_blobs.created_at <= ?", 2.days.ago).find_each(&:purge_later)
  end
end
```

WARNING: `ActiveStorage::Blob.unattached`で生成されるクエリは、大規模なデータベースを使うアプリケーションでは時間がかかり、ユーザーの混乱を招く可能性があります。
