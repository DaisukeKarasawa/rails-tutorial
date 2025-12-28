# 第7章: パフォーマンス最適化とスケーリング

## はじめに

この章では、第2章と第3章で作成したアプリケーションのパフォーマンスを最適化します。N+1問題の解決、データベースクエリの最適化、キャッシュ戦略、バックグラウンドジョブなどを実装しながら、実践的な最適化手法を学びます。

[Evil Martians](https://evilmartians.com/)の実践的なTipsも参考にしながら、本番環境で実際に使える最適化手法を学びます。

## 7.1 N+1問題の特定と解決

### 問題の確認

現在の`PostsController#index`でN+1問題が発生している可能性があります。確認してみましょう。

```ruby
# app/controllers/posts_controller.rb
def index
  @posts = Post.order(created_at: :desc).page(params[:page])
end
```

このコードでは、各投稿のユーザー情報を取得する際に、投稿の数だけクエリが発行されます。

### 問題の検出

```bash
# 開発環境でクエリログを確認
rails server
# ブラウザで投稿一覧ページにアクセス
# ログでクエリの数を確認
```

### Eager Loadingの実装

```ruby
# app/queries/posts_query.rb
class PostsQuery
  # ... 既存のコード ...

  private

  def apply_includes(relation)
    relation.includes(:user)  # 既に実装済み
  end
end
```

### カウンターキャッシュの追加

投稿数が多い場合、ユーザーの投稿数を表示する際にN+1問題が発生します。

```ruby
# マイグレーション
rails generate migration AddPostsCountToUsers posts_count:integer
```

```ruby
# db/migrate/xxxxxx_add_posts_count_to_users.rb
class AddPostsCountToUsers < ActiveRecord::Migration[7.0]
  def change
    add_column :users, :posts_count, :integer, default: 0, null: false
  end
end
```

```bash
rails db:migrate
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user, counter_cache: true
  # ... 既存のコード ...
end
```

```ruby
# 既存データの更新
User.find_each do |user|
  User.reset_counters(user.id, :posts)
end
```

## 7.2 データベースクエリの最適化

### インデックスの追加

```ruby
# マイグレーション
rails generate migration AddIndexesToPosts
```

```ruby
# db/migrate/xxxxxx_add_indexes_to_posts.rb
class AddIndexesToPosts < ActiveRecord::Migration[7.0]
  def change
    add_index :posts, :user_id
    add_index :posts, :created_at
    add_index :posts, [:user_id, :created_at]
  end
end
```

```bash
rails db:migrate
```

### selectの使用

必要なカラムのみ取得するようにします。

```ruby
# app/queries/posts_query.rb
class PostsQuery
  def apply_includes(relation)
    relation.includes(:user).select('posts.*, users.name as user_name')
  end
end
```

### pluckの使用

IDのみが必要な場合。

```ruby
# app/controllers/posts_controller.rb
def index
  post_ids = PostsQuery.new.call.pluck(:id)
  @posts = Post.where(id: post_ids).includes(:user).map(&:decorate)
end
```

## 7.3 Caching with Rails（キャッシュ戦略の全体像）

### 参考資料
- [Rails Guides - Caching with Rails（英語）](https://guides.rubyonrails.org/caching_with_rails.html)
- [Rails Guides - Caching with Rails（日本語）](https://railsguides.jp/caching_with_rails.html)

### キャッシュストアの選択

```ruby
# config/environments/development.rb
# 開発環境: メモリキャッシュ
config.cache_store = :memory_store, { size: 64.megabytes }

# config/environments/production.rb
# 本番環境: Redis
config.cache_store = :redis_cache_store, {
  url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0'),
  namespace: 'sample_app:cache',
  expires_in: 1.hour,
  reconnect_attempts: 3
}
```

### フラグメントキャッシュ（Fragment Caching）

```erb
<!-- app/views/posts/index.html.erb -->
<% cache @posts do %>
  <% @posts.each do |post| %>
    <% cache post do %>
      <article>
        <h2><%= link_to post.title, post %></h2>
        <p><%= post.truncated_content %></p>
        <p>
          投稿者: <%= post.author_name %>
          投稿日時: <%= post.formatted_created_at %>
        </p>
      </article>
    <% end %>
  <% end %>
<% end %>

<!-- 条件付きキャッシュ -->
<% cache_if user_signed_in?, [post, 'authenticated'] do %>
  <%= render post %>
<% end %>

<!-- コレクションキャッシュ -->
<%= render partial: 'posts/post', collection: @posts, cached: true %>
```

### SQLキャッシュ（Query Cache）

```ruby
# 同じクエリが同じリクエスト内で複数回実行される場合、2回目以降はキャッシュから取得
Post.find(1)  # データベースにクエリ
Post.find(1)  # キャッシュから取得（クエリなし）

# リクエストが終了するとキャッシュはクリアされる
```

### 低レベルキャッシュ（Low-Level Caching）

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def posts_count_cached
    Rails.cache.fetch("user:#{id}:posts_count", expires_in: 1.hour) do
      posts.count
    end
  end

  def statistics
    Rails.cache.fetch("user:#{id}:statistics", expires_in: 30.minutes) do
      {
        posts_count: posts.count,
        comments_count: comments.count,
        likes_count: posts.sum(:likes_count)
      }
    end
  end
end
```

### キャッシュキーの設計

```ruby
# モデルベースのキャッシュキー
cache_key = [user, user.updated_at.to_i]

# カスタムキャッシュキー
cache_key = "user:#{user.id}:stats:#{Date.today}"

# 複合キー
cache_key = "posts:user:#{user.id}:page:#{page}:sort:#{sort}"
```

### キャッシュの無効化

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  after_update :expire_cache
  after_destroy :expire_cache
  after_touch :expire_cache

  private

  def expire_cache
    # 特定のキャッシュを削除
    Rails.cache.delete("post:#{id}")
    Rails.cache.delete("user:#{user_id}:posts")
    
    # パターンマッチで削除（Redisの場合）
    # Rails.cache.delete_matched("user:#{user_id}:*")
    
    # キャッシュバージョンを更新
    user.touch  # userのupdated_atが更新され、キャッシュキーが変わる
  end
end
```

### キャッシュバージョニング

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  def cache_version
    "#{id}-#{updated_at.to_i}"
  end
end

# 使用例
Rails.cache.fetch("post:#{post.cache_version}") do
  # キャッシュ内容
end
```

### キャッシュのテスト

```ruby
# spec/models/post_spec.rb
RSpec.describe Post do
  describe 'cache expiration' do
    it 'expires cache on update' do
      post = create(:post)
      Rails.cache.write("post:#{post.id}", 'cached_value')
      
      post.update(title: 'Updated')
      
      expect(Rails.cache.read("post:#{post.id}")).to be_nil
    end
  end
end
```

## 7.4 Active Support Instrumentation（計測・可観測性）

### 参考資料
- [Rails Guides - Active Support Instrumentation（英語）](https://guides.rubyonrails.org/active_support_instrumentation.html)
- [Rails Guides - Active Support Instrumentation（日本語）](https://railsguides.jp/active_support_instrumentation.html)
- [Rails API - ActiveSupport::Notifications](https://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)

### Instrumentationとは

Instrumentationは、アプリケーションの動作を計測・監視するための仕組みです。

### 標準的なイベント

```ruby
# Active Recordイベント
ActiveSupport::Notifications.subscribe('sql.active_record') do |name, start, finish, id, payload|
  duration = finish - start
  Rails.logger.info("SQL: #{payload[:sql]} (#{duration}s)")
end

# Action Controllerイベント
ActiveSupport::Notifications.subscribe('process_action.action_controller') do |name, start, finish, id, payload|
  duration = finish - start
  Rails.logger.info("Controller: #{payload[:controller]}##{payload[:action]} (#{duration}s)")
end
```

### カスタムイベントの作成

```ruby
# app/services/post_creation_service.rb
class PostCreationService
  def call
    ActiveSupport::Notifications.instrument('post.creation', post_id: post.id) do
      # 処理
      create_post
    end
  end
end

# イベントの購読
ActiveSupport::Notifications.subscribe('post.creation') do |name, start, finish, id, payload|
  duration = finish - start
  Rails.logger.info("Post created: #{payload[:post_id]} in #{duration}s")
end
```

### イベントの計測

```ruby
# config/initializers/instrumentation.rb
ActiveSupport::Notifications.subscribe('process_action.action_controller') do |name, start, finish, id, payload|
  duration = finish - start
  
  # スロークエリの検出
  if payload[:db_runtime] && payload[:db_runtime] > 0.1
    Rails.logger.warn("Slow query detected: #{payload[:controller]}##{payload[:action]} (#{payload[:db_runtime]}s)")
  end
  
  # メトリクスの記録
  StatsD.timing('rails.request.duration', duration * 1000)
  StatsD.increment('rails.request.count')
end
```

### イベントのフィルタリング

```ruby
# 特定のイベントのみ購読
ActiveSupport::Notifications.subscribe(/^sql\.active_record$/) do |name, start, finish, id, payload|
  # SQLイベントのみ処理
end

# 条件付き購読
ActiveSupport::Notifications.subscribe('process_action.action_controller') do |name, start, finish, id, payload|
  next unless payload[:controller] == 'PostsController'
  # PostsControllerのイベントのみ処理
end
```

### パフォーマンス計測

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  around_action :measure_performance

  private

  def measure_performance
    start_time = Time.current
    yield
    duration = Time.current - start_time
    
    ActiveSupport::Notifications.instrument('posts.index', duration: duration) do
      # 処理
    end
  end
end
```

## 7.5 Active Job Basics（標準APIとデフォルトキュー基盤）

### 参考資料
- [Rails Guides - Active Job Basics（英語）](https://guides.rubyonrails.org/active_job_basics.html)
- [Rails Guides - Active Job Basics（日本語）](https://railsguides.jp/active_job_basics.html)

### Active Jobとは

Active Jobは、バックグラウンドジョブを定義・実行するための統一されたインターフェースです。Sidekiq、Delayed Job、Resqueなどのバックグラウンドジョブアダプターを抽象化します。

### ジョブの作成

```bash
rails generate job PostCreatedNotification
```

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :default

  def perform(post_id)
    post = Post.find(post_id)
    # 処理を実装
    UserMailer.post_created_notification(post).deliver_now
  end
end
```

### ジョブの実行

```ruby
# 非同期で実行（バックグラウンド）
PostCreatedNotificationJob.perform_later(post.id)

# 同期的に実行（即座に実行）
PostCreatedNotificationJob.perform_now(post.id)

# 指定した時刻に実行
PostCreatedNotificationJob.set(wait: 1.hour).perform_later(post.id)
PostCreatedNotificationJob.set(wait_until: 1.day.from_now).perform_later(post.id)
```

### キューアダプターの設定

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    # デフォルト: :async（開発環境）
    # 本番環境では: :sidekiq, :delayed_job, :resque など
    config.active_job.queue_adapter = :sidekiq
  end
end

# 環境別の設定
# config/environments/development.rb
config.active_job.queue_adapter = :async

# config/environments/production.rb
config.active_job.queue_adapter = :sidekiq
```

### デフォルトキュー基盤（Rails 7.1以降）

Rails 7.1以降、Solid Queueがデフォルトのキューアダプターとして利用可能になりました。

```ruby
# Gemfile
gem 'solid_queue'
```

```bash
bundle install
rails generate solid_queue:install
rails db:migrate
```

```ruby
# config/environments/production.rb
config.active_job.queue_adapter = :solid_queue
```

### キューの指定

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :notifications  # デフォルトキュー以外を指定
end

# 実行時にキューを指定
PostCreatedNotificationJob.set(queue: :high_priority).perform_later(post.id)
```

### リトライとエラーハンドリング

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :default
  
  # リトライ設定
  retry_on StandardError, wait: 5.seconds, attempts: 3
  
  # 特定のエラーはリトライしない
  discard_on ActiveRecord::RecordNotFound
  
  def perform(post_id)
    post = Post.find(post_id)
    # 処理
  rescue StandardError => e
    # エラーハンドリング
    Rails.logger.error("Job failed: #{e.message}")
    raise  # リトライするために再発生
  end
end
```

### コールバック

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :default
  
  before_perform :log_start
  after_perform :log_completion
  around_perform :measure_duration
  
  def perform(post_id)
    # 処理
  end
  
  private
  
  def log_start
    Rails.logger.info("Starting job: #{self.class.name}")
  end
  
  def log_completion
    Rails.logger.info("Completed job: #{self.class.name}")
  end
  
  def measure_duration
    start_time = Time.current
    yield
    duration = Time.current - start_time
    Rails.logger.info("Job took #{duration} seconds")
  end
end
```

### ジョブの引数

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :default
  
  # 複数の引数を受け取る
  def perform(post_id, user_id, options = {})
    post = Post.find(post_id)
    user = User.find(user_id)
    
    # オプションの処理
    if options[:send_email]
      UserMailer.post_created_notification(post, user).deliver_now
    end
  end
end

# 使用例
PostCreatedNotificationJob.perform_later(post.id, user.id, send_email: true)
```

### ジョブのテスト

```ruby
# spec/jobs/post_created_notification_job_spec.rb
RSpec.describe PostCreatedNotificationJob, type: :job do
  describe '#perform' do
    it 'sends notification email' do
      post = create(:post)
      
      expect {
        PostCreatedNotificationJob.perform_now(post.id)
      }.to change { ActionMailer::Base.deliveries.count }.by(1)
    end
    
    it 'enqueues job' do
      post = create(:post)
      
      expect {
        PostCreatedNotificationJob.perform_later(post.id)
      }.to have_enqueued_job(PostCreatedNotificationJob)
    end
  end
end
```

### ジョブの優先度

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :default
  
  # 優先度の設定（アダプターによって異なる）
  def priority
    10  # 高い優先度
  end
end
```

### ジョブのキャンセル

```ruby
# ジョブのキャンセル（アダプターによって異なる）
# Sidekiqの場合
job = PostCreatedNotificationJob.perform_later(post.id)
Sidekiq::Queue.new.find_job(job.job_id)&.delete
```

## 7.6 Sidekiqのセットアップと実装

### Sidekiqのセットアップ

```ruby
# Gemfile
gem 'sidekiq'
```

```bash
bundle install
```

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end
```

### メール送信ジョブの作成

```ruby
# app/jobs/post_created_notification_job.rb
class PostCreatedNotificationJob < ApplicationJob
  queue_as :default

  def perform(post_id)
    post = Post.find(post_id)
    # 将来的にフォロワーに通知を送る機能を追加
    # UserMailer.post_created_notification(post).deliver_later
  end
end
```

### Service Objectの更新

```ruby
# app/services/post_creation_service.rb
class PostCreationService
  # ... 既存のコード ...

  private

  def create_post
    post = user.posts.create!(post_params)
    PostCreatedNotificationJob.perform_later(post.id)
    post
  end
end
```

### Sidekiqの起動

```bash
# 別のターミナルで
bundle exec sidekiq
```

### SidekiqとActive Jobの統合

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :sidekiq
  end
end

# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end

Sidekiq.configure_client do |config|
  config.redis = { url: ENV.fetch('REDIS_URL', 'redis://localhost:6379/0') }
end
```

## 7.7 データベースインデックスの最適化

### クエリの分析

```ruby
# app/controllers/posts_controller.rb
def index
  @posts = PostsQuery.new.call
  # クエリの分析
  ActiveRecord::Base.logger = Logger.new(STDOUT) if Rails.env.development?
end
```

### 複合インデックスの追加

```ruby
# db/migrate/xxxxxx_add_composite_index_to_posts.rb
class AddCompositeIndexToPosts < ActiveRecord::Migration[7.0]
  def change
    add_index :posts, [:user_id, :created_at], order: { created_at: :desc }
  end
end
```

## 7.8 クエリ最適化のためのロギング

### 参考資料
- [Evil Martians - Rails Query Optimizations（英語）](https://evilmartians.com/chronicles/rails-query-optimizations)

### クエリログの設定

```ruby
# config/environments/development.rb
Rails.application.configure do
  # クエリログを有効化
  config.log_level = :debug
  
  # クエリの実行時間を表示
  ActiveRecord::Base.logger = Logger.new(STDOUT)
end
```

### クエリの分析と最適化

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    # クエリの実行時間を測定
    start_time = Time.current
    @posts = PostsQuery.new.call
    query_time = Time.current - start_time
    
    Rails.logger.info("Posts query took #{query_time} seconds")
    
    # クエリの数をカウント
    query_count = 0
    ActiveSupport::Notifications.subscribe('sql.active_record') do |*args|
      query_count += 1
    end
    
    @posts.to_a  # クエリを実行
    
    Rails.logger.info("Total queries: #{query_count}")
  end
end
```

### クエリの可視化

```ruby
# Gemfile
group :development do
  gem 'rack-mini-profiler'
  gem 'flamegraph'
end
```

```ruby
# config/initializers/rack_mini_profiler.rb
if Rails.env.development?
  Rack::MiniProfiler.config.position = 'bottom-right'
  Rack::MiniProfiler.config.start_hidden = false
end
```

### スロークエリの検出

```ruby
# config/initializers/query_monitor.rb
if Rails.env.development?
  ActiveSupport::Notifications.subscribe('sql.active_record') do |name, start, finish, id, payload|
    duration = finish - start
    
    if duration > 0.1  # 100ms以上かかるクエリ
      Rails.logger.warn("Slow query detected: #{payload[:sql]} (#{duration}s)")
    end
  end
end
```

## 7.9 Active Record重い画面の対処Tips

### 参考資料
- [Evil Martians - 5 Tips for ActiveRecord Dashboards（英語）](https://evilmartians.com/chronicles/5-tips-for-activerecord-dashboards)

### Tip 1: 必要なカラムのみ取得

```ruby
# 悪い例: すべてのカラムを取得
@posts = Post.all

# 良い例: 必要なカラムのみ取得
@posts = Post.select(:id, :title, :created_at, :user_id).all
```

### Tip 2: 集計をデータベースで実行

```ruby
# 悪い例: Rubyで集計
posts = Post.all
total_comments = posts.sum { |post| post.comments.count }

# 良い例: データベースで集計
total_comments = Post.joins(:comments).count
# または
total_comments = Post.left_joins(:comments).group('posts.id').count.size
```

### Tip 3: カウンターキャッシュの活用

```ruby
# マイグレーション
rails generate migration AddCommentsCountToPosts comments_count:integer
```

```ruby
# db/migrate/xxxxxx_add_comments_count_to_posts.rb
class AddCommentsCountToPosts < ActiveRecord::Migration[7.0]
  def change
    add_column :posts, :comments_count, :integer, default: 0, null: false
  end
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :comments, dependent: :destroy, counter_cache: true
end

# 使用例
@posts = Post.select(:id, :title, :comments_count).all
# comments.count のクエリが不要
```

### Tip 4: ビューでの計算を避ける

```ruby
# 悪い例: ビューで計算
# <% @posts.each do |post| %>
#   <p>Comments: <%= post.comments.count %></p>
# <% end %>

# 良い例: 事前に計算
@posts = Post.includes(:comments).select('posts.*, COUNT(comments.id) as comments_count')
             .left_joins(:comments)
             .group('posts.id')
             .all

# ビューで使用
# <% @posts.each do |post| %>
#   <p>Comments: <%= post.comments_count %></p>
# <% end %>
```

### Tip 5: ページネーションの最適化

```ruby
# Gemfile
gem 'kaminari'
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    # オフセットベースのページネーション（遅い）
    # @posts = Post.offset(params[:page].to_i * 25).limit(25)
    
    # カーソルベースのページネーション（速い）
    @posts = Post.where('id > ?', params[:cursor].to_i)
                 .order(:id)
                 .limit(25)
                 .includes(:user)
  end
end
```

### Tip 6: バックグラウンドでの集計

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # 集計結果をキャッシュ
  def self.statistics
    Rails.cache.fetch('posts_statistics', expires_in: 1.hour) do
      {
        total: count,
        published: where(published: true).count,
        recent: where('created_at > ?', 1.week.ago).count
      }
    end
  end
end
```

```ruby
# app/jobs/update_statistics_job.rb
class UpdateStatisticsJob < ApplicationJob
  queue_as :default

  def perform
    statistics = {
      total_posts: Post.count,
      total_users: User.count,
      total_comments: Comment.count,
      updated_at: Time.current
    }
    
    Rails.cache.write('dashboard_statistics', statistics, expires_in: 1.hour)
  end
end
```

### Tip 7: 部分的なデータ読み込み

```ruby
# 重い画面では、必要なデータのみ読み込む
class DashboardsController < ApplicationController
  def show
    # すべてのデータを読み込まない
    @recent_posts = Post.recent.limit(10).includes(:user)
    @popular_posts = Post.popular.limit(10).includes(:user)
    @statistics = Post.statistics  # キャッシュから取得
  end
end
```

## 7.10 パフォーマンス測定

### ベンチマークの追加

```ruby
# lib/tasks/benchmark.rake
namespace :benchmark do
  desc 'Benchmark posts index'
  task posts_index: :environment do
    require 'benchmark'
    
    Benchmark.bm do |x|
      x.report('without includes') do
        100.times { Post.order(created_at: :desc).limit(25).to_a }
      end
      
      x.report('with includes') do
        100.times { Post.includes(:user).order(created_at: :desc).limit(25).to_a }
      end
    end
  end
end
```

```bash
rails benchmark:posts_index
```

## 7.11 確認ポイント

この章の最後に、以下を確認してください：

1. **N+1問題が解決されている**
   - クエリログで確認
   - パフォーマンスが改善されている

2. **キャッシュが動作している**
   - Redisが起動している
   - キャッシュが有効に機能している

3. **バックグラウンドジョブが動作している**
   - Sidekiqが起動している
   - ジョブが正常に処理されている

## 7.12 参考資料

- [Rails Guides - Active Job Basics（英語）](https://guides.rubyonrails.org/active_job_basics.html)
- [Rails Guides - Caching with Rails（英語）](https://guides.rubyonrails.org/caching_with_rails.html)
- [Rails Guides - Active Support Instrumentation（英語）](https://guides.rubyonrails.org/active_support_instrumentation.html)
- [Rails API - ActiveSupport::Notifications](https://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)
- [Evil Martians - 5 Tips for ActiveRecord Dashboards（英語）](https://evilmartians.com/chronicles/5-tips-for-activerecord-dashboards)
- [Evil Martians - Rails Query Optimizations（英語）](https://evilmartians.com/chronicles/rails-query-optimizations)
- [includes/preload/eager_load の挙動整理（日本語）](https://tech.stmn.co.jp/entry/2020/11/30/145159)

## 7.13 次の章へ

この章で、アプリケーションのパフォーマンスを最適化しました。次の章では、セキュリティを強化します。

**次の章**: [第8章: セキュリティの深掘りとベストプラクティス](chapter08-security.md)

## まとめ

この章では以下を学びました：

- **N+1問題の解決**: Eager Loading、カウンターキャッシュ
- **クエリ最適化**: インデックス、select、pluck
- **Caching with Rails**: fragment/SQL cache等、キャッシュ戦略の全体像
- **Active Support Instrumentation**: 計測・可観測性の入口
- **Active Job Basics**: 標準APIとデフォルトキュー基盤（Solid Queue）
- **バックグラウンドジョブ**: Sidekiqの実装とActive Jobとの統合
- **クエリ最適化のためのロギング**: スロークエリの検出、クエリの可視化
- **Active Record重い画面の対処**: 必要なカラムのみ取得、集計の最適化、ページネーションの最適化
- **パフォーマンス測定**: ベンチマークの実施
