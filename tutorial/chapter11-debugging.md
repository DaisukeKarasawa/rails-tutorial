# 第11章: デバッグ

## はじめに

この章では、[Rails Guides - Debugging Rails Applications](https://guides.rubyonrails.org/debugging_rails_applications.html)を参考に、Railsアプリケーションのデバッグ方法を学びます。ログ、例外、スタックトレース、デバッガの使い所を理解し、効率的にデバッグできるようになります。

## 11.1 Debugging Rails Applications（公式ガイドの全体像）

### 参考資料
- [Rails Guides - Debugging Rails Applications（英語）](https://guides.rubyonrails.org/debugging_rails_applications.html)
- [Rails Guides - Debugging Rails Applications（日本語）](https://railsguides.jp/debugging_rails_applications.html)

### デバッグの基本原則

1. **ログを活用する**: 問題の原因を特定するための情報を記録
2. **例外を理解する**: エラーメッセージとスタックトレースを読む
3. **デバッガを使う**: 実行中のコードの状態を確認
4. **段階的に絞り込む**: 問題の範囲を特定してから詳細を調査

## 11.2 ログの詳細な使い方

### ログレベルの設定

```ruby
# config/environments/development.rb
Rails.application.configure do
  config.log_level = :debug  # :debug, :info, :warn, :error, :fatal
end

# config/environments/production.rb
Rails.application.configure do
  config.log_level = :info  # 本番環境では通常:info以上
end
```

### ログの出力先

```ruby
# config/environments/development.rb
Rails.application.configure do
  # 標準出力に出力
  config.logger = ActiveSupport::Logger.new(STDOUT)
  
  # ファイルに出力
  config.logger = ActiveSupport::Logger.new(Rails.root.join('log', 'development.log'))
  
  # 複数の出力先に出力
  config.logger = ActiveSupport::Logger.new(
    Rails.root.join('log', 'development.log')
  )
  config.logger.extend(ActiveSupport::Logger.broadcast(
    ActiveSupport::Logger.new(STDOUT)
  ))
end
```

### ログの出力方法

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    # デバッグレベル
    Rails.logger.debug "Creating post with params: #{post_params.inspect}"
    Rails.logger.debug { "User: #{current_user.inspect}" }  # ブロック形式（遅延評価）
    
    @post = current_user.posts.build(post_params)
    
    if @post.save
      # 情報レベル
      Rails.logger.info "Post created: #{@post.id}"
      redirect_to @post
    else
      # 警告レベル
      Rails.logger.warn "Post creation failed: #{@post.errors.full_messages}"
      # エラーレベル
      Rails.logger.error "Post creation error: #{@post.errors.full_messages.join(', ')}"
      render :new
    end
  end
end
```

### タグ付きログ

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.log_tags = [
      :request_id,
      -> request { Time.current },
      -> request { request.remote_ip }
    ]
  end
end
```

### 構造化ログ

```ruby
# config/initializers/log_formatter.rb
class StructuredLogFormatter < Logger::Formatter
  def call(severity, time, progname, msg)
    {
      timestamp: time.iso8601,
      level: severity,
      message: msg,
      pid: Process.pid,
      thread_id: Thread.current.object_id
    }.to_json + "\n"
  end
end

Rails.logger.formatter = StructuredLogFormatter.new
```

### ログのフィルタリング

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password,
  :password_confirmation,
  :credit_card,
  :ssn
]
```

## 11.3 例外とスタックトレースの理解

### 例外の種類

```ruby
# StandardError - 通常のエラー
begin
  Post.find(999)
rescue StandardError => e
  Rails.logger.error "Error: #{e.message}"
  Rails.logger.error e.backtrace.join("\n")
end

# ActiveRecord::RecordNotFound - レコードが見つからない
begin
  Post.find(999)
rescue ActiveRecord::RecordNotFound => e
  Rails.logger.warn "Post not found: #{e.message}"
  redirect_to posts_path, alert: '投稿が見つかりませんでした'
end

# ActionController::ParameterMissing - パラメータが不足
begin
  params.require(:post)
rescue ActionController::ParameterMissing => e
  Rails.logger.error "Missing parameter: #{e.message}"
  render json: { error: e.message }, status: :bad_request
end
```

### スタックトレースの読み方

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    begin
      @post = current_user.posts.build(post_params)
      @post.save!
    rescue => e
      # スタックトレースをログに出力
      Rails.logger.error "Exception: #{e.class}"
      Rails.logger.error "Message: #{e.message}"
      Rails.logger.error "Backtrace:"
      e.backtrace.each do |line|
        Rails.logger.error "  #{line}"
      end
      
      # ユーザーには分かりやすいメッセージを表示
      flash[:error] = '投稿の作成に失敗しました'
      render :new
    end
  end
end
```

### 例外の再発生

```ruby
# app/services/post_creation_service.rb
class PostCreationService
  def call
    begin
      create_post
    rescue ActiveRecord::RecordInvalid => e
      # ログに記録
      Rails.logger.error "Validation failed: #{e.message}"
      # 再発生させて上位で処理
      raise
    rescue StandardError => e
      # 予期しないエラーをログに記録
      Rails.logger.error "Unexpected error: #{e.message}"
      Rails.error.report(e, severity: :error)
      # 再発生
      raise
    end
  end
end
```

### カスタム例外

```ruby
# app/exceptions/post_creation_error.rb
class PostCreationError < StandardError
  attr_reader :post, :errors

  def initialize(post, errors)
    @post = post
    @errors = errors
    super("Post creation failed: #{errors.join(', ')}")
  end
end

# 使用例
raise PostCreationError.new(@post, @post.errors.full_messages)
```

## 11.4 デバッガの使い所

### debug gemの使用

```ruby
# Gemfile
group :development, :test do
  gem 'debug'
end
```

```bash
bundle install
```

### ブレークポイントの設定

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    
    # ブレークポイントを設定
    binding.break  # または debugger
    
    # 変数の値を確認
    # @post の状態を確認
    # post_params の内容を確認
    
    if @post.save
      redirect_to @post
    else
      render :new
    end
  end
end
```

### デバッガーのコマンド

```ruby
# デバッガー内でのコマンド
continue  # c - 実行を続ける
next      # n - 次の行に進む（メソッド呼び出しをスキップ）
step      # s - メソッドの中に入る
finish    # fin - 現在のメソッドを終了
where     # bt - スタックトレースを表示
up        # u - スタックフレームを上に移動
down      # d - スタックフレームを下に移動
frame     # f - 特定のフレームに移動
list      # l - 現在のコードを表示
break     # b - ブレークポイントを設定
delete    # del - ブレークポイントを削除
info      # i - 変数やメソッドの情報を表示
eval      # p - 式を評価して表示
```

### 条件付きブレークポイント

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.all
    
    # 条件付きブレークポイント
    binding.break if @posts.count > 100
    
    # または
    binding.break(pre: -> { @posts.count > 100 })
  end
end
```

### デバッガーでの変数の確認

```ruby
# デバッガー内で実行
# 変数の値を確認
p @post
p @post.title
p @post.errors.full_messages

# メソッドの実行
p @post.valid?
p @post.save

# 式の評価
p Post.count
p current_user.posts.count
```

### デバッガーの使い所

1. **変数の状態を確認したい時**
   ```ruby
   binding.break
   # 変数の値を確認
   ```

2. **メソッドの実行フローを追いたい時**
   ```ruby
   binding.break
   # step でメソッドの中に入る
   ```

3. **条件分岐の動作を確認したい時**
   ```ruby
   binding.break if condition
   # 条件が満たされた時だけ停止
   ```

4. **エラーが発生する直前の状態を確認したい時**
   ```ruby
   begin
     # 処理
   rescue => e
     binding.break
     # エラー発生時の状態を確認
   end
   ```

## 11.5 byebugの使用（従来の方法）

```ruby
# Gemfile
group :development, :test do
  gem 'debug'
end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    
    debugger  # ここでデバッガーが起動
    
    if @post.save
      redirect_to @post
    else
      render :new
    end
  end
end
```

### デバッガーのコマンド

```ruby
# デバッガー内でのコマンド
continue  # 実行を続ける
next      # 次の行に進む
step      # メソッドの中に入る
finish    # 現在のメソッドを終了
where     # スタックトレースを表示
up        # スタックフレームを上に移動
down      # スタックフレームを下に移動
eval      # 式を評価
```

## 11.6 エラーページの活用

### better_errorsの使用

```ruby
# Gemfile
group :development do
  gem 'better_errors'
  gem 'binding_of_caller'
end
```

```bash
bundle install
```

エラーが発生すると、詳細な情報が表示されます。

### better_errorsの設定

```ruby
# config/environments/development.rb
if defined?(BetterErrors)
  BetterErrors.application_root = File.expand_path('..', __dir__)
  BetterErrors.maximum_variable_inspect_size = 1000
end
```

## 11.7 クエリのデバッグ

### クエリログの確認

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    ActiveRecord::Base.logger = Logger.new(STDOUT)
    @posts = Post.includes(:user).all
  end
end
```

### クエリの分析

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    queries = []
    
    ActiveSupport::Notifications.subscribe('sql.active_record') do |*args|
      queries << args.last[:sql]
    end
    
    @posts = Post.includes(:user).all
    
    puts "Total queries: #{queries.count}"
    queries.each { |q| puts q }
  end
end
```

## 11.8 パフォーマンスのデバッグ

### rack-mini-profilerの使用

```ruby
# Gemfile
group :development do
  gem 'rack-mini-profiler'
end
```

```bash
bundle install
```

ページの下部にパフォーマンス情報が表示されます。

### メモリプロファイリング

```ruby
# Gemfile
group :development do
  gem 'memory_profiler'
end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    report = MemoryProfiler.report do
      @posts = Post.includes(:user).all
    end
    
    report.pretty_print
  end
end
```

## 11.9 テストのデバッグ

### テストの実行

```bash
# 特定のテストを実行
bundle exec rspec spec/models/post_spec.rb

# 詳細な出力
bundle exec rspec spec/models/post_spec.rb --format documentation

# 失敗したテストのみ再実行
bundle exec rspec --only-failures
```

### テスト内でのデバッグ

```ruby
# spec/models/post_spec.rb
RSpec.describe Post do
  it 'validates title presence' do
    post = Post.new
    debugger  # デバッガーが起動
    expect(post).not_to be_valid
  end
end
```

## 11.10 デバッグのベストプラクティス

### 1. ログを適切に使用する

```ruby
# 良い例: 適切なログレベルを使用
Rails.logger.debug "Debug information"
Rails.logger.info "Important information"
Rails.logger.warn "Warning message"
Rails.logger.error "Error message"

# 悪い例: すべてをerrorレベルで出力
Rails.logger.error "Debug information"  # 不適切
```

### 2. 例外を適切に処理する

```ruby
# 良い例: 適切な例外処理
begin
  risky_operation
rescue SpecificError => e
  # 特定のエラーを処理
  handle_specific_error(e)
rescue StandardError => e
  # 一般的なエラーを処理
  handle_general_error(e)
ensure
  # クリーンアップ
  cleanup
end
```

### 3. デバッガーを効果的に使う

```ruby
# 良い例: 問題の原因を特定するために使用
binding.break if condition  # 条件付きブレークポイント

# 悪い例: すべての場所にブレークポイントを設定
binding.break  # 不要な場所に設定
```

### 4. スタックトレースを読む

```ruby
# スタックトレースの読み方
# 1. エラーメッセージを確認
# 2. スタックトレースの最初の数行を確認（実際のエラー発生箇所）
# 3. アプリケーションコードの部分を確認
# 4. ライブラリの部分は通常無視して良い
```

## 11.11 確認ポイント

この章の最後に、以下を確認してください：

1. **デバッガーが動作している**
   - debug gemでブレークポイントが機能する
   - デバッガーコマンドが理解できている

2. **ログが正しく出力されている**
   - 適切なログレベルが設定されている
   - 必要な情報がログに記録されている
   - 構造化ログが機能している

3. **例外が適切に処理されている**
   - スタックトレースが読める
   - 例外の種類を理解している

4. **パフォーマンス情報が確認できる**
   - rack-mini-profilerが動作している
   - クエリログが確認できる

## 11.12 参考資料

- [Rails Guides - Debugging Rails Applications（英語）](https://guides.rubyonrails.org/debugging_rails_applications.html)
- [Rails Guides - Debugging Rails Applications（日本語）](https://railsguides.jp/debugging_rails_applications.html)

## 11.13 次の章へ

この章で、デバッグの方法を学びました。次の章では、本番環境へのデプロイを学びます。

**次の章**: [第12章: 本番環境のデプロイと運用](chapter12-production-deployment.md)

## まとめ

この章では以下を学びました：

- **Debugging Rails Applications**: ログ、例外、スタックトレース、デバッガの使い所
- **ログの詳細な使い方**: ログレベル、出力先、タグ付きログ、構造化ログ、フィルタリング
- **例外とスタックトレース**: 例外の種類、スタックトレースの読み方、例外の再発生、カスタム例外
- **デバッガの使い所**: debug gem、ブレークポイント、デバッガーコマンド、条件付きブレークポイント、変数の確認
- **エラーページ**: better_errorsの設定
- **クエリのデバッグ**: クエリログ、クエリ分析
- **パフォーマンスのデバッグ**: rack-mini-profiler、メモリプロファイリング
- **テストのデバッグ**: テスト実行、テスト内デバッグ
- **デバッグのベストプラクティス**: ログの適切な使用、例外の適切な処理、デバッガーの効果的な使用、スタックトレースの読み方