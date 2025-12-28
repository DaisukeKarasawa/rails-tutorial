# 第1章: Railsの内部動作を理解する

## はじめに

この章では、Railsの内部動作を理解するために重要な4つのトピックを学びます。これらの知識は、中級・上級者としてRailsアプリケーションを開発する上で不可欠です。

- **Autoloading / Zeitwerk**: 定数解決、eager load、開発時リロードの仕組み
- **Rails Initialization Process**: 起動順序、initializerの置き場所、Railtie/Engineの理解
- **Configuring Rails Applications**: 環境差分、設定のスコープ、初期化と絡む設定
- **The Rails Command Line**: `bin/rails`、generator、runner、タスク運用

## 1.1 Autoloading / Zeitwerk（超重要）

### 参考資料
- [Rails Guides - Autoloading and Reloading Constants（英語）](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
- [Rails Guides - Autoloading and Reloading Constants（日本語）](https://railsguides.jp/autoloading_and_reloading_constants.html)

### Zeitwerkとは

Rails 6以降、定数の自動読み込み（autoloading）には**Zeitwerk**が使用されています。Zeitwerkは、ファイル名とディレクトリ構造から定数名を推測して自動的に読み込みます。

### 命名規則

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # User という定数が自動的に読み込まれる
end

# app/models/admin/user.rb
module Admin
  class User < ApplicationRecord
    # Admin::User という定数が自動的に読み込まれる
  end
end

# app/services/user_registration_service.rb
class UserRegistrationService
  # UserRegistrationService という定数が自動的に読み込まれる
end
```

### ディレクトリ構造と定数名の対応

```
app/
  models/
    user.rb                    → User
    admin/
      user.rb                  → Admin::User
  services/
    user_registration_service.rb → UserRegistrationService
  controllers/
    posts_controller.rb        → PostsController
    admin/
      posts_controller.rb      → Admin::PostsController
```

### 開発環境でのリロード

開発環境では、ファイルを変更すると自動的にリロードされます。

```ruby
# config/environments/development.rb
config.cache_classes = false  # デフォルト
config.reload_classes_only_on_change = true
```

### 本番環境でのEager Loading

本番環境では、起動時にすべての定数を事前に読み込みます（eager loading）。

```ruby
# config/environments/production.rb
config.cache_classes = true  # デフォルト
config.eager_load = true     # デフォルト
```

### カスタムオートロードパスの追加

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.load_defaults 7.0

    # カスタムオートロードパスを追加
    config.autoload_paths << "#{root}/lib"
    config.autoload_paths << "#{root}/app/services"
    config.autoload_paths << "#{root}/app/queries"
  end
end
```

### 名前空間の管理

```ruby
# app/models/concerns/authenticatable.rb
module Authenticatable
  extend ActiveSupport::Concern

  included do
    # ...
  end
end

# app/models/user.rb
class User < ApplicationRecord
  include Authenticatable
  # Authenticatable が自動的に読み込まれる
end
```

### よくある問題と解決方法

#### 問題1: 循環依存

```ruby
# 問題のあるコード
# app/models/user.rb
class User < ApplicationRecord
  has_many :posts
end

# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  # User がまだ読み込まれていない可能性がある
end
```

**解決方法**: 通常は問題ありませんが、必要に応じて明示的にrequireする

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user  # Railsが自動的に解決する
end
```

#### 問題2: 定数が見つからない

```ruby
# エラー: uninitialized constant UserRegistrationService
service = UserRegistrationService.new
```

**解決方法**: ファイル名と定数名が一致しているか確認

```ruby
# app/services/user_registration_service.rb
class UserRegistrationService
  # 正しい命名規則に従っている
end
```

### 手動での読み込み

```ruby
# 通常は不要だが、必要に応じて
require_dependency 'user_registration_service'
```

### 確認ポイント

```bash
# オートロードパスを確認
rails runner "puts Rails.application.config.autoload_paths"

# 定数が正しく読み込まれているか確認
rails runner "puts User"
rails runner "puts UserRegistrationService"
```

## 1.2 Rails Initialization Process

### 参考資料
- [Rails Guides - The Rails Initialization Process（英語）](https://guides.rubyonrails.org/initialization.html)
- [Rails Guides - The Rails Initialization Process（日本語）](https://railsguides.jp/initialization.html)

### 起動順序

Railsアプリケーションの起動は、以下の順序で行われます：

1. **bin/rails** が実行される
2. **config/boot.rb** が読み込まれる（Bundlerの設定）
3. **config/application.rb** が読み込まれる
4. **config/environment.rb** が読み込まれる
5. **Rails.application.initialize!** が呼ばれる
6. **config/initializers/** 内のファイルが順番に読み込まれる

### 起動プロセスの詳細

```ruby
# config/boot.rb
ENV['BUNDLE_GEMFILE'] ||= File.expand_path('../Gemfile', __dir__)
require 'bundler/setup' # Set up gems listed in the Gemfile.
```

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.load_defaults 7.0
    # ここでアプリケーションの設定を行う
  end
end
```

```ruby
# config/environment.rb
# Load the Rails application.
require_relative "application"

# Initialize the Rails application.
Rails.application.initialize!
```

### Initializersの作成

```ruby
# config/initializers/custom_config.rb
Rails.application.configure do
  config.custom_setting = 'value'
end

# または
Rails.application.config.custom_setting = 'value'
```

### Initializersの実行順序

Initializersはファイル名のアルファベット順に実行されます。

```ruby
# config/initializers/01_custom_config.rb  # 最初に実行
# config/initializers/02_another_config.rb
# config/initializers/99_final_config.rb   # 最後に実行
```

### Railtieの理解

Railtieは、Railsアプリケーションに機能を追加するための仕組みです。

```ruby
# lib/my_gem/railtie.rb
module MyGem
  class Railtie < Rails::Railtie
    # アプリケーション起動時に実行
    initializer "my_gem.configure" do |app|
      app.config.my_gem.setting = 'value'
    end

    # タスクを追加
    rake_tasks do
      load "tasks/my_gem.rake"
    end

    # ジェネレーターを追加
    generators do
      require "generators/my_gem_generator"
    end
  end
end
```

### Engineの理解

Engineは、Railsアプリケーションの機能を再利用可能な形にしたものです。

```ruby
# lib/my_engine/engine.rb
module MyEngine
  class Engine < ::Rails::Engine
    isolate_namespace MyEngine

    # ルーティングを追加
    config.routes.draw do
      resources :posts
    end
  end
end
```

### 起動フック

```ruby
# config/initializers/startup_hook.rb
Rails.application.config.after_initialize do
  # アプリケーション起動後に実行される処理
  puts "Application initialized!"
end
```

## 1.3 Configuring Rails Applications

### 参考資料
- [Rails Guides - Configuring Rails Applications（英語）](https://guides.rubyonrails.org/configuring.html)
- [Rails Guides - Configuring Rails Applications（日本語）](https://railsguides.jp/configuring.html)

### 環境別の設定

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.cache_classes = false
  config.eager_load = false
  config.consider_all_requests_local = true
  config.active_support.deprecation = :log
end

# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true
  config.consider_all_requests_local = false
  config.active_support.deprecation = :notify
end
```

### 設定のスコープ

```ruby
# config/application.rb - すべての環境に適用
module SampleApp
  class Application < Rails::Application
    config.time_zone = 'Tokyo'
    config.i18n.default_locale = :ja
  end
end

# config/environments/development.rb - 開発環境のみ
Rails.application.configure do
  config.log_level = :debug
end

# config/environments/production.rb - 本番環境のみ
Rails.application.configure do
  config.force_ssl = true
end
```

### カスタム設定の追加

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    # カスタム設定を追加
    config.custom_setting = 'default_value'
    config.api_key = ENV['API_KEY']
  end
end

# 使用例
Rails.application.config.custom_setting
Rails.application.config.api_key
```

### 環境変数の使用

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.database_url = ENV['DATABASE_URL']
    config.redis_url = ENV.fetch('REDIS_URL', 'redis://localhost:6379/0')
  end
end
```

### 設定の確認

```bash
# 設定を確認
rails runner "puts Rails.application.config.time_zone"
rails runner "puts Rails.application.config.custom_setting"

# 環境を確認
rails runner "puts Rails.env"
```

### 初期化と絡む設定

```ruby
# config/initializers/custom_config.rb
Rails.application.config.after_initialize do
  # 初期化後に実行される処理
  if Rails.env.production?
    # 本番環境でのみ実行
  end
end
```

## 1.4 The Rails Command Line

### 参考資料
- [Rails Guides - The Rails Command Line（英語）](https://guides.rubyonrails.org/command_line.html)
- [Rails Guides - The Rails Command Line（日本語）](https://railsguides.jp/command_line.html)

### bin/railsコマンド

```bash
# アプリケーションの情報を表示
rails --version
rails about

# ヘルプを表示
rails --help
rails generate --help
```

### ジェネレーター

```bash
# モデルの生成
rails generate model User name:string email:string
rails g model User name:string email:string  # 短縮形

# コントローラーの生成
rails generate controller Posts index show
rails g controller Posts index show

# マイグレーションの生成
rails generate migration AddEmailToUsers email:string
rails g migration AddEmailToUsers email:string

# ジェネレーターの一覧
rails generate
rails g
```

### カスタムジェネレーターの作成

```ruby
# lib/generators/service/service_generator.rb
class ServiceGenerator < Rails::Generators::NamedBase
  source_root File.expand_path('templates', __dir__)

  def create_service_file
    template 'service.rb.erb', "app/services/#{file_name}_service.rb"
  end
end
```

```erb
# lib/generators/service/templates/service.rb.erb
class <%= class_name %>Service
  def initialize
  end

  def call
    # TODO: 実装
  end
end
```

```bash
# カスタムジェネレーターの使用
rails generate service UserRegistration
```

### Rails Runner

```bash
# Rubyコードを実行
rails runner "puts User.count"
rails runner "User.all.each { |u| puts u.name }"

# ファイルを実行
rails runner script.rb

# 環境を指定
RAILS_ENV=production rails runner "puts User.count"
```

### Rails Console

```bash
# コンソールを起動
rails console
rails c  # 短縮形

# 環境を指定
RAILS_ENV=production rails console

# サンドボックスモード（変更をロールバック）
rails console --sandbox
```

### データベースコマンド

```bash
# データベースの作成
rails db:create

# マイグレーションの実行
rails db:migrate

# マイグレーションのロールバック
rails db:rollback

# データベースのリセット
rails db:reset

# シードデータの投入
rails db:seed

# データベースの状態を確認
rails db:version
rails db:migrate:status
```

### カスタムRakeタスクの作成

```ruby
# lib/tasks/custom.rake
namespace :custom do
  desc "カスタムタスクの説明"
  task :my_task => :environment do
    puts "カスタムタスクを実行"
    User.all.each do |user|
      puts user.name
    end
  end

  task :another_task, [:arg1, :arg2] => :environment do |t, args|
    puts "引数1: #{args[:arg1]}"
    puts "引数2: #{args[:arg2]}"
  end
end
```

```bash
# カスタムタスクの実行
rails custom:my_task
rails custom:another_task[value1,value2]

# すべてのタスクを確認
rails -T
rails -T custom
```

### サーバーの起動

```bash
# サーバーを起動
rails server
rails s  # 短縮形

# ポートを指定
rails server -p 3001

# バインディングを指定
rails server -b 0.0.0.0
```

### アセットのプリコンパイル

```bash
# アセットをプリコンパイル
rails assets:precompile

# アセットをクリーンアップ
rails assets:clobber
```

## 1.5 実践的な例

### カスタム設定の実装

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.load_defaults 7.0

    # カスタム設定
    config.posts_per_page = 25
    config.max_upload_size = 5.megabytes
  end
end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.page(params[:page]).per(Rails.application.config.posts_per_page)
  end
end
```

### カスタムInitializerの実装

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end
```

### カスタムRakeタスクの実装

```ruby
# lib/tasks/posts.rake
namespace :posts do
  desc "古い投稿をアーカイブ"
  task archive_old: :environment do
    old_posts = Post.where('created_at < ?', 1.year.ago)
    count = old_posts.update_all(status: 'archived')
    puts "#{count}件の投稿をアーカイブしました"
  end

  desc "投稿の統計を表示"
  task stats: :environment do
    puts "総投稿数: #{Post.count}"
    puts "公開済み: #{Post.published.count}"
    puts "下書き: #{Post.draft.count}"
  end
end
```

## 1.6 Active Support Core Extensions（Rails流Ruby拡張の前提）

### 参考資料
- [Rails Guides - Active Support Core Extensions（英語）](https://guides.rubyonrails.org/active_support_core_extensions.html)
- [Rails Guides - Active Support Core Extensions（日本語）](https://railsguides.jp/active_support_core_extensions.html)

### Active Supportとは

Active Supportは、Rubyの標準ライブラリを拡張し、Railsアプリケーションで便利なメソッドを提供するライブラリです。

### 文字列の拡張

```ruby
# 文字列の操作
"hello".pluralize        # => "hellos"
"posts".singularize      # => "post"
"hello_world".camelize   # => "HelloWorld"
"HelloWorld".underscore  # => "hello_world"
"hello world".titleize   # => "Hello World"

# 文字列の変換
"user".classify          # => "User"
"users".constantize      # => Users (クラス)
"hello".humanize         # => "Hello"
```

### 数値の拡張

```ruby
# 時間の計算
1.second
1.minute
1.hour
1.day
1.week
1.month
1.year

# 使用例
Time.current + 1.hour
Time.current - 2.days
1.week.ago
2.months.from_now
```

### 配列の拡張

```ruby
# 配列の操作
[1, 2, 3].to_sentence           # => "1, 2, and 3"
[1, 2, 3].to_sentence(locale: :ja)  # => "1、2、3"
[1, 2, 3].to_formatted_s(:db)   # => "1,2,3"

# 配列の分割
[1, 2, 3, 4, 5].in_groups_of(2) # => [[1, 2], [3, 4], [5, nil]]
[1, 2, 3, 4, 5].split(3)         # => [[1, 2], [4, 5]]
```

### ハッシュの拡張

```ruby
# ハッシュの操作
{ a: 1, b: 2 }.deep_merge({ b: 3, c: 4 })  # => { a: 1, b: 3, c: 4 }
{ a: 1, b: 2 }.stringify_keys              # => { "a" => 1, "b" => 2 }
{ "a" => 1, "b" => 2 }.symbolize_keys      # => { a: 1, b: 2 }
{ a: 1, b: 2 }.slice(:a, :c)               # => { a: 1 }
{ a: 1, b: 2 }.except(:b)                  # => { a: 1 }
```

### 日時の拡張

```ruby
# 日時の操作
Time.current.beginning_of_day
Time.current.end_of_day
Time.current.beginning_of_week
Time.current.end_of_week
Time.current.beginning_of_month
Time.current.end_of_month
Time.current.beginning_of_year
Time.current.end_of_year

# 日付の操作
Date.today.beginning_of_week
Date.today.end_of_week
Date.today.beginning_of_month
Date.today.end_of_month
```

### オブジェクトの拡張

```ruby
# オブジェクトの操作
user.presence              # => user (nilでなければuser、nilならnil)
user.blank?                # => false
user.present?              # => true
nil.blank?                 # => true
"".blank?                  # => true
[].blank?                  # => true
{}.blank?                  # => true

# オブジェクトの複製
user.dup                   # 浅いコピー
user.deep_dup              # 深いコピー
```

### 条件付き実行

```ruby
# 条件付き実行
user.try(:name)            # => user.name (userがnilでなければ)
nil.try(:name)             # => nil (エラーにならない)

# 条件付きメソッドチェーン
user&.posts&.first          # 安全なナビゲーション
```

### バリデーションの拡張

```ruby
# バリデーション
validates :email, presence: true
validates :age, numericality: { greater_than: 0 }
validates :url, format: { with: URI::regexp }
```

## 1.7 CurrentAttributes（便利だが事故りやすい論点）

### 参考資料
- [CurrentAttributesの賛否・注意点を知る（TechRacho）](https://techracho.bpsinc.jp/hachi8833/2024_01_25/43810)

### CurrentAttributesとは

CurrentAttributesは、リクエストスコープでグローバルにアクセス可能な属性を提供する機能です。現在のユーザーやリクエスト情報を簡単にアクセスできます。

### CurrentAttributesの実装

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user
  attribute :request_id
  attribute :ip_address
end
```

### 使用例

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_current_attributes

  private

  def set_current_attributes
    Current.user = current_user
    Current.request_id = request.uuid
    Current.ip_address = request.remote_ip
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  before_create :set_creator

  private

  def set_creator
    self.created_by = Current.user.id if Current.user
  end
end
```

### CurrentAttributesの利点

1. **簡潔なコード**: グローバルにアクセス可能
2. **リクエストスコープ**: 各リクエストで独立
3. **テストしやすい**: テストで簡単に設定可能

### CurrentAttributesの問題点と注意点

#### 1. スレッド安全性の問題

```ruby
# 問題のあるコード
class Current < ActiveSupport::CurrentAttributes
  attribute :user
end

# マルチスレッド環境で問題が発生する可能性
Thread.new do
  Current.user = user1
  # 他のスレッドと共有される可能性
end
end
```

#### 2. テストでの注意

```ruby
# テストでの注意
RSpec.describe Post do
  it 'creates post with current user' do
    user = create(:user)
    Current.user = user
    
    post = Post.create(title: 'Test', content: 'Content')
    
    expect(post.created_by).to eq(user.id)
    
    # テスト後にクリア
    Current.reset
  end
end
```

#### 3. バックグラウンドジョブでの使用

```ruby
# 問題のあるコード
class PostCreationJob < ApplicationJob
  def perform(post_id)
    # Current.userは設定されていない（リクエストスコープ外）
    post = Post.find(post_id)
    post.update(created_by: Current.user.id)  # => nil
  end
end

# 解決策: 明示的に渡す
class PostCreationJob < ApplicationJob
  def perform(post_id, user_id)
    Current.user = User.find(user_id)
    post = Post.find(post_id)
    post.update(created_by: Current.user.id)
  end
end
```

#### 4. モデルでの使用の注意

```ruby
# 注意: モデルでCurrentAttributesを使用する場合
class Post < ApplicationRecord
  before_create :set_creator

  private

  def set_creator
    # Current.userがnilの可能性がある
    if Current.user
      self.created_by = Current.user.id
    else
      # エラー処理
      raise "Current.user is not set"
    end
  end
end
```

### CurrentAttributesのベストプラクティス

1. **明示的に設定する**: コントローラーで必ず設定
2. **テストでクリアする**: テスト後に`Current.reset`を呼ぶ
3. **バックグラウンドジョブでは使わない**: 明示的に引数として渡す
4. **nilチェックを忘れない**: Current.userがnilの可能性を考慮

### CurrentAttributesの代替案

```ruby
# 代替案1: 明示的に渡す
class Post < ApplicationRecord
  def self.create_with_user(user, attributes)
    create(attributes.merge(created_by: user.id))
  end
end

# 代替案2: サービスオブジェクトを使用
class PostCreationService
  def initialize(user)
    @user = user
  end

  def call(post_params)
    Post.create(post_params.merge(created_by: @user.id))
  end
end
```

## 1.8 確認ポイント

この章の最後に、以下を確認してください：

1. **Zeitwerkが正しく動作している**
   - 定数が自動的に読み込まれる
   - ファイル名と定数名が一致している

2. **Initializersが正しく実行されている**
   - 設定が正しく読み込まれている

3. **カスタム設定が動作している**
   - 環境変数が正しく読み込まれている

4. **Railsコマンドが理解できている**
   - ジェネレーター、runner、タスクが使用できる

5. **Active Support Core Extensionsが理解できている**
   - 文字列、数値、配列、ハッシュの拡張メソッドが使用できる

6. **CurrentAttributesの注意点が理解できている**
   - スレッド安全性、テスト、バックグラウンドジョブでの注意点を理解している

## 1.9 参考資料

- [Rails Guides - Autoloading and Reloading Constants（英語）](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
- [Rails Guides - The Rails Initialization Process（英語）](https://guides.rubyonrails.org/initialization.html)
- [Rails Guides - Configuring Rails Applications（英語）](https://guides.rubyonrails.org/configuring.html)
- [Rails Guides - The Rails Command Line（英語）](https://guides.rubyonrails.org/command_line.html)
- [Rails Guides - Active Support Core Extensions（英語）](https://guides.rubyonrails.org/active_support_core_extensions.html)
- [CurrentAttributesの賛否・注意点を知る（TechRacho）](https://techracho.bpsinc.jp/hachi8833/2024_01_25/43810)

## 1.10 次の章へ

この章で、Railsの内部動作を理解しました。次の章では、基本的なアプリケーションを構築します。

**次の章**: [第2章: 基本的なアプリケーション構築](chapter02-basic-application.md)

## まとめ

この章では以下を学びました：

- **Autoloading / Zeitwerk**: 定数解決、eager load、開発時リロードの仕組み
- **Rails Initialization Process**: 起動順序、initializer、Railtie/Engine
- **Configuring Rails Applications**: 環境差分、設定のスコープ、カスタム設定
- **The Rails Command Line**: bin/rails、generator、runner、タスク運用
- **Active Support Core Extensions**: Rails流Ruby拡張の前提（文字列、数値、配列、ハッシュ、日時、オブジェクトの拡張）
- **CurrentAttributes**: 便利だが事故りやすい論点（利点、問題点、注意点、ベストプラクティス、代替案）

これらの知識は、Railsアプリケーションを深く理解し、効率的に開発する上で不可欠です。
