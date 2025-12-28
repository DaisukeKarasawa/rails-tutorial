# 第12章: 本番環境のデプロイと運用

## はじめに

この章では、作成したアプリケーションを本番環境にデプロイし、運用する方法を学びます。Pumaの設定、Docker化、本番環境へのデプロイ、モニタリング、エラートラッキングなどを実装しながら、実践的な運用手法を学びます。

## 12.1 Pumaの設定とパフォーマンスチューニング

### 参考資料
- [Rails Guides - Tuning Performance for Deployment（英語）](https://guides.rubyonrails.org/tuning_performance_for_deployment.html)
- [Heroku Dev Center - Deploying Rails Applications with the Puma Web Server](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server)
- [Judo Scale - Puma Default Threads Changed](https://judoscale.com/blog/puma-default-threads-changed)

### Pumaとは

Pumaは、Rails 5以降のデフォルトWebサーバーです。マルチスレッドとマルチプロセスをサポートしています。

### Pumaの設定

```ruby
# config/puma.rb
max_threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
min_threads_count = ENV.fetch("RAILS_MIN_THREADS") { max_threads_count }
threads min_threads_count, max_threads_count

worker_timeout 3600 if ENV.fetch("RAILS_ENV", "development") == "development"

port ENV.fetch("PORT") { 3000 }

environment ENV.fetch("RAILS_ENV") { "development" }

pidfile ENV.fetch("PIDFILE") { "tmp/pids/server.pid" }

workers ENV.fetch("WEB_CONCURRENCY") { 2 }

preload_app!

plugin :tmp_restart
```

### スレッドとワーカーの理解

#### スレッド（Threads）

- **min_threads**: 最小スレッド数（デフォルト: 0、Rails 7.1以降は5）
- **max_threads**: 最大スレッド数（デフォルト: 5）
- 各スレッドは独立してリクエストを処理
- I/O待機中は他のスレッドが処理を続行可能

#### ワーカー（Workers）

- **workers**: プロセス数（デフォルト: 0、本番環境では2以上推奨）
- 各ワーカーは独立したプロセス
- CPUコア数に応じて設定

### 環境変数による設定

```bash
# スレッド数の設定
export RAILS_MAX_THREADS=5
export RAILS_MIN_THREADS=5

# ワーカー数の設定
export WEB_CONCURRENCY=2

# ポートの設定
export PORT=3000
```

### 本番環境での推奨設定

```ruby
# config/puma.rb（本番環境用）
workers ENV.fetch("WEB_CONCURRENCY") { 2 }
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 5 }
threads threads_count, threads_count

preload_app!

rackup      DefaultRackup
port        ENV.fetch("PORT") { 3000 }
environment ENV.fetch("RAILS_ENV") { "production" }

on_worker_boot do
  ActiveRecord::Base.establish_connection
end

before_fork do
  ActiveRecord::Base.connection_pool.disconnect!
end
```

### メモリ使用量の最適化

```ruby
# config/puma.rb
# メモリリークを防ぐため、定期的にワーカーを再起動
worker_timeout 3600

# メモリ制限を設定（systemdの場合）
# LimitNOFILE=65536
```

### Pumaのモニタリング

```ruby
# config/initializers/puma_monitor.rb
if Rails.env.production?
  require 'puma/plugin'
  
  Puma::Plugin.create do
    def start(launcher)
      Thread.new do
        loop do
          sleep 60
          stats = launcher.stats
          Rails.logger.info("Puma stats: #{stats}")
        end
      end
    end
  end
end
```

## 12.2 Error Reporting in Rails Applications（Rails標準のエラーレポート基盤）

### 参考資料
- [Rails Guides - Error Reporting in Rails Applications（英語）](https://guides.rubyonrails.org/error_reporting.html)
- [Rails Guides - Error Reporting in Rails Applications（日本語）](https://railsguides.jp/error_reporting.html)

### Rails.errorの使用

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    
    Rails.error.handle do
      if @post.save
        redirect_to @post
      else
        render :new
      end
    end
  end
end
```

### エラーハンドラーの登録

```ruby
# config/initializers/error_reporting.rb
Rails.error.subscribe(ErrorSubscriber.new)

class ErrorSubscriber
  def report(error, handled:, severity:, context:, source: nil)
    # Sentryに送信
    Sentry.capture_exception(error, extra: context) if severity == :error
    
    # ログに記録
    Rails.logger.error("Error: #{error.message}")
    Rails.logger.error(error.backtrace.join("\n"))
  end
end
```

### エラーコンテキストの追加

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    Rails.error.set_context(user_id: current_user.id, action: 'create_post')
    
    @post = current_user.posts.build(post_params)
    # ...
  end
end
```

### エラーの分類

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    Rails.error.handle(severity: :warning) do
      # 警告レベルのエラー
      process_post
    end
  rescue StandardError => e
    Rails.error.report(e, severity: :error, context: { user_id: current_user.id })
    raise
  end
end
```

## 12.3 Docker化

### Dockerfileの作成

```dockerfile
# Dockerfile
FROM ruby:3.1.0

RUN apt-get update -qq && apt-get install -y \
  build-essential \
  libpq-dev \
  nodejs \
  yarn \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

RUN bundle exec rails assets:precompile

EXPOSE 3000

CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

### .dockerignore

```
.git
.gitignore
README.md
.env
.env.local
log/*
tmp/*
node_modules
coverage
.sass-cache
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: sample_app_production

  redis:
    image: redis:7
    volumes:
      - redis_data:/data

  web:
    build: .
    command: bundle exec puma -C config/puma.rb
    volumes:
      - .:/app
    ports:
      - "3000:3000"
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/sample_app_production
      REDIS_URL: redis://redis:6379/0
      RAILS_ENV: production
      SECRET_KEY_BASE: ${SECRET_KEY_BASE}

  sidekiq:
    build: .
    command: bundle exec sidekiq
    volumes:
      - .:/app
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/sample_app_production
      REDIS_URL: redis://redis:6379/0
      RAILS_ENV: production

volumes:
  postgres_data:
  redis_data:
```

## 12.4 本番環境の設定

### 環境変数の設定

```ruby
# config/environments/production.rb
Rails.application.configure do
  config.cache_classes = true
  config.eager_load = true
  config.consider_all_requests_local = false
  config.public_file_server.enabled = ENV['RAILS_SERVE_STATIC_FILES'].present?
  config.force_ssl = true
  config.log_level = :info
  config.log_formatter = ::Logger::Formatter.new
  config.active_record.dump_schema_after_migration = false
end
```

### シークレットキーの管理

```bash
# シークレットキーの生成
rails secret

# .env.production に追加
SECRET_KEY_BASE=your_secret_key_here
```

## 12.5 ヘルスチェックエンドポイント

### ヘルスチェックコントローラー

```ruby
# app/controllers/health_controller.rb
class HealthController < ApplicationController
  def show
    checks = {
      database: database_check,
      redis: redis_check
    }

    if checks.values.all?
      render json: { status: 'ok', checks: checks }, status: :ok
    else
      render json: { status: 'error', checks: checks }, status: :service_unavailable
    end
  end

  private

  def database_check
    ActiveRecord::Base.connection.execute('SELECT 1')
    true
  rescue StandardError
    false
  end

  def redis_check
    Redis.current.ping == 'PONG'
  rescue StandardError
    false
  end
end
```

### ルーティング

```ruby
# config/routes.rb
Rails.application.routes.draw do
  get '/health', to: 'health#show'
  # ... 既存のルーティング ...
end
```

## 12.6 エラートラッキング（Sentry + Rails.error）

### Sentryのセットアップ

```ruby
# Gemfile
gem 'sentry-ruby'
gem 'sentry-rails'
```

```bash
bundle install
```

```ruby
# config/initializers/sentry.rb
Sentry.init do |config|
  config.dsn = ENV['SENTRY_DSN']
  config.breadcrumbs_logger = [:active_support_logger, :http_logger]
  config.traces_sample_rate = 0.5
  config.environment = Rails.env
end
```

### Rails.errorとの統合

```ruby
# config/initializers/error_reporting.rb
Rails.error.subscribe(SentryErrorSubscriber.new)

class SentryErrorSubscriber
  def report(error, handled:, severity:, context:, source: nil)
    return unless severity == :error || severity == :fatal
    
    Sentry.with_scope do |scope|
      scope.set_context(:rails_error, context)
      scope.set_tag(:source, source) if source
      Sentry.capture_exception(error)
    end
  end
end
```

### エラーハンドリング

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  rescue_from StandardError, with: :handle_error

  private

  def handle_error(exception)
    Rails.error.report(exception, 
      severity: :error,
      context: {
        user_id: current_user&.id,
        controller: controller_name,
        action: action_name
      }
    )
    
    raise exception unless Rails.env.production?
    
    render json: {
      error: 'Internal Server Error',
      message: 'An error occurred. Please try again later.'
    }, status: :internal_server_error
  end
end
```

## 12.7 ログ管理

### ログの構造化

```ruby
# config/initializers/log_formatter.rb
class StructuredLogFormatter < Logger::Formatter
  def call(severity, time, progname, msg)
    {
      timestamp: time.iso8601,
      level: severity,
      message: msg,
      pid: Process.pid
    }.to_json + "\n"
  end
end

Rails.logger.formatter = StructuredLogFormatter.new if Rails.env.production?
```

## 12.8 バックアップ戦略

### バックアップタスク

```ruby
# lib/tasks/backup.rake
namespace :db do
  desc 'Backup database'
  task backup: :environment do
    timestamp = Time.current.strftime('%Y%m%d_%H%M%S')
    backup_file = "backup_#{timestamp}.sql"
    
    system("pg_dump #{ENV['DATABASE_URL']} > #{backup_file}")
    
    puts "Backup completed: #{backup_file}"
  end
end
```

## 12.9 デプロイの実行

### ビルドと起動

```bash
# Dockerイメージのビルド
docker-compose build

# データベースのマイグレーション
docker-compose run web rails db:migrate

# アプリケーションの起動
docker-compose up -d
```

## 12.10 確認ポイント

この章の最後に、以下を確認してください：

1. **Dockerコンテナが正常に起動する**
   ```bash
   docker-compose ps
   ```

2. **ヘルスチェックが正常に動作する**
   ```bash
   curl http://localhost:3000/health
   ```

3. **アプリケーションが正常に動作する**
   - ブラウザでアクセスして確認

## 12.11 参考資料

- [Rails Guides - Tuning Performance for Deployment（英語）](https://guides.rubyonrails.org/tuning_performance_for_deployment.html)
- [Rails Guides - Error Reporting in Rails Applications（英語）](https://guides.rubyonrails.org/error_reporting.html)
- [Heroku Dev Center - Deploying Rails Applications with the Puma Web Server](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server)
- [Judo Scale - Puma Default Threads Changed](https://judoscale.com/blog/puma-default-threads-changed)

## 12.12 次の章へ

この章で、本番環境へのデプロイと運用の基礎を学びました。次の章では、実践的な開発手法とワークフローを学びます。

**次の章**: [第13章: 実践的な開発手法とワークフロー](chapter13-practical-development.md)

## まとめ

この章では以下を学びました：

- **Pumaの設定**: スレッドとワーカーの理解、環境変数による設定
- **パフォーマンスチューニング**: メモリ使用量の最適化、モニタリング
- **Error Reporting**: Rails標準のエラーレポート基盤、Rails.errorの使用
- **Docker化**: コンテナ化とdocker-compose
- **本番環境設定**: 環境変数、シークレットキー
- **ヘルスチェック**: エンドポイントの実装
- **エラートラッキング**: Sentry + Rails.errorの統合
- **ログ管理**: 構造化ログ
- **バックアップ**: データベースバックアップ
