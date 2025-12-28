# 第3章: Active Recordの高度な活用

## はじめに

この章では、Active Recordの基礎から高度な機能まで、包括的に学びます。[Rails Guides - Active Record Basics](https://guides.rubyonrails.org/active_record_basics.html)から始めて、Query Interface、Associations、Migrations、Validations、Callbacks、Multiple Databases、Active Record Encryptionまで、実践的な内容を扱います。

第2章で作成した基本的なアプリケーションを拡張しながら、Active Recordの全機能を実践します。

## 3.1 Active Record Basics（基礎を正しい前提に戻す）

### 参考資料
- [Rails Guides - Active Record Basics（英語）](https://guides.rubyonrails.org/active_record_basics.html)
- [Rails Guides - Active Record Basics（日本語）](https://railsguides.jp/active_record_basics.html)

### Active Recordとは

Active Recordは、**ORM（Object-Relational Mapping）パターン**の実装です。データベースのテーブルをRubyのクラスとして扱い、レコードをオブジェクトとして操作できます。

### 命名規則

```ruby
# テーブル名: posts（複数形、スネークケース）
# モデル名: Post（単数形、パスカルケース）
class Post < ApplicationRecord
  # postsテーブルに対応
end

# テーブル名: user_profiles（複数形、スネークケース）
# モデル名: UserProfile（単数形、パスカルケース）
class UserProfile < ApplicationRecord
  # user_profilesテーブルに対応
end
```

### CRUD操作の基礎

```ruby
# Create（作成）
post = Post.new(title: "Hello", content: "World")
post.save
# または
post = Post.create(title: "Hello", content: "World")

# Read（読み取り）
post = Post.find(1)
post = Post.find_by(title: "Hello")
posts = Post.where(published: true)
all_posts = Post.all

# Update（更新）
post.update(title: "Updated")
post.update_columns(title: "Updated")  # バリデーションをスキップ

# Delete（削除）
post.destroy  # コールバックを実行
post.delete   # コールバックをスキップ
```

### 属性の操作

```ruby
# 属性の読み取り
post.title
post[:title]
post.read_attribute(:title)

# 属性の書き込み
post.title = "New Title"
post[:title] = "New Title"
post.write_attribute(:title, "New Title")

# 属性の変更確認
post.title_changed?
post.title_was
post.changes
post.changed_attributes
```

### バリデーションとエラー

```ruby
# バリデーションの実行
post.valid?
post.invalid?

# エラーの確認
post.errors
post.errors.full_messages
post.errors[:title]
```

## 3.2 Active Record Query Interface（N+1、eager loading、locking、EXPLAIN）

### 参考資料
- [Rails Guides - Active Record Query Interface（英語）](https://guides.rubyonrails.org/active_record_querying.html)
- [Rails Guides - Active Record Query Interface（日本語）](https://railsguides.jp/active_record_querying.html)

### 参考資料
- [Rails Guides - Active Record Querying（英語）](https://guides.rubyonrails.org/active_record_querying.html)
- [Rails Guides - Active Record Querying（日本語）](https://railsguides.jp/active_record_querying.html)
- [Rails API - ActiveRecord::QueryMethods](https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html)

### whereメソッドの高度な使い方

```ruby
# 基本的なwhere
Post.where(published: true)

# 条件の組み合わせ
Post.where(published: true).where('created_at > ?', 1.week.ago)

# OR条件
Post.where(published: true).or(Post.where(featured: true))

# NOT条件
Post.where.not(published: false)

# 範囲検索
Post.where(created_at: 1.week.ago..Time.current)

# 部分一致検索（ILIKE - PostgreSQL）
Post.where('title ILIKE ?', "%#{params[:q]}%")

# 配列での検索
Post.where(id: [1, 2, 3])
Post.where(status: ['published', 'draft'])
```

### joins、includes、eager_load、preloadの違いと挙動整理

#### 基本的な違い

```ruby
# joins - JOINのみ、関連オブジェクトはロードしない
Post.joins(:user).where(users: { active: true })
# => SELECT "posts".* FROM "posts" INNER JOIN "users" ON "users"."id" = "posts"."user_id" WHERE "users"."active" = true
# 注意: post.user にアクセスすると追加のクエリが発行される（N+1問題）

# includes - 事前にロード（LEFT OUTER JOINまたは別クエリ）
Post.includes(:user).all
# ケース1: where条件がない場合 → 別クエリで取得
# => SELECT "posts".* FROM "posts"
# => SELECT "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3)

# ケース2: where条件がある場合 → LEFT OUTER JOINで取得
Post.includes(:user).where(users: { active: true }).references(:users)
# => SELECT "posts"."id" AS t0_r0, "posts"."title" AS t0_r1, "users"."id" AS t0_r2, ...
#    FROM "posts" LEFT OUTER JOIN "users" ON "users"."id" = "posts"."user_id"
#    WHERE "users"."active" = true

# eager_load - 常にLEFT OUTER JOINで一度に取得
Post.eager_load(:user).all
# => SELECT "posts"."id" AS t0_r0, "posts"."title" AS t0_r1, "users"."id" AS t0_r2, ...
#    FROM "posts" LEFT OUTER JOIN "users" ON "users"."id" = "posts"."user_id"

# preload - 常に別々のクエリで取得
Post.preload(:user).all
# => SELECT "posts".* FROM "posts"
# => SELECT "users".* FROM "users" WHERE "users"."id" IN (1, 2, 3)
```

#### 使い分けの判断基準

**includesを使う場合:**
- 関連オブジェクトにアクセスする可能性がある
- where条件がない、または柔軟にクエリを変更したい
- Railsが自動的に最適な方法を選択してくれる

**eager_loadを使う場合:**
- 関連オブジェクトの条件でフィルタリングしたい
- 常にLEFT OUTER JOINで取得したい
- クエリ数を最小限にしたい

**preloadを使う場合:**
- 関連オブジェクトにアクセスする可能性がある
- 必ず別クエリで取得したい（JOINを避けたい）
- 大量のデータを扱う場合（JOINの結果が大きくなりすぎるのを防ぐ）

**joinsを使う場合:**
- 関連オブジェクトの条件でフィルタリングしたいが、関連オブジェクト自体は不要
- 集計クエリなど、関連オブジェクトのデータは不要

#### 実践的な例

```ruby
# ケース1: 関連オブジェクトにアクセスする
posts = Post.includes(:user).all
posts.each { |post| puts post.user.name }  # 追加クエリなし

# ケース2: 関連オブジェクトの条件でフィルタリング
posts = Post.eager_load(:user).where(users: { active: true })
# または
posts = Post.includes(:user).where(users: { active: true }).references(:users)

# ケース3: 関連オブジェクトは不要、条件のみ使用
posts = Post.joins(:user).where(users: { active: true })
posts.each { |post| puts post.title }  # userにアクセスしない

# ケース4: 大量のデータを扱う場合
posts = Post.preload(:user).limit(10000)
# JOINの結果が大きくなりすぎるのを防ぐ
```

#### パフォーマンスの比較

```ruby
# ベンチマーク
require 'benchmark'

Benchmark.bm do |x|
  x.report('includes') do
    100.times { Post.includes(:user).all.each { |p| p.user.name } }
  end

  x.report('eager_load') do
    100.times { Post.eager_load(:user).all.each { |p| p.user.name } }
  end

  x.report('preload') do
    100.times { Post.preload(:user).all.each { |p| p.user.name } }
  end
end
```

### ネストした関連のEager Loading

```ruby
# 複数レベルの関連を一度にロード
Post.includes(user: :profile).all
Post.includes(comments: :user).all

# 条件付きEager Loading
Post.includes(:user).where(users: { active: true }).references(:users)

# 複数の関連を同時にロード
Post.includes(:user, :comments, :tags).all
```

### 参考資料（詳細な挙動整理）
- [includes/preload/eager_load の挙動整理（日本語）](https://tech.stmn.co.jp/entry/2020/11/30/145159)

### select、pluck、idsの使い分け

```ruby
# select - 必要なカラムのみ取得（オブジェクトを作成）
Post.select(:id, :title, :created_at).all

# pluck - 配列として取得（オブジェクトを作成しない）
Post.pluck(:id, :title)
# => [[1, "Post 1"], [2, "Post 2"]]

# ids - IDのみ取得
Post.ids
# => [1, 2, 3]
```

### スコープの定義と活用

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user

  # クラスメソッドとして定義
  scope :published, -> { where(published: true) }
  scope :recent, -> { order(created_at: :desc) }
  scope :by_user, ->(user) { where(user: user) }

  # 条件付きスコープ
  scope :recent_published, -> { published.recent }

  # 引数を受け取るスコープ
  scope :created_after, ->(date) { where('created_at > ?', date) }

  # デフォルトスコープ（注意して使用）
  default_scope { order(created_at: :desc) }
end

# 使用例
Post.published
Post.recent
Post.by_user(current_user)
Post.published.recent
Post.created_after(1.week.ago)
```

### サブクエリの活用

```ruby
# サブクエリを使用した検索
Post.where(user_id: User.where(active: true).select(:id))

# EXISTS句
Post.where('EXISTS (?)', User.where('users.id = posts.user_id').where(active: true))
```

### find_by_sqlの使用

```ruby
# 複雑なクエリを直接実行
Post.find_by_sql("
  SELECT posts.*, COUNT(comments.id) as comments_count
  FROM posts
  LEFT JOIN comments ON comments.post_id = posts.id
  GROUP BY posts.id
  HAVING COUNT(comments.id) > 5
")
```

### Locking（ロック）

```ruby
# 悲観的ロック（Pessimistic Locking）
post = Post.lock.find(1)
# SELECT * FROM posts WHERE id = 1 FOR UPDATE

# 楽観的ロック（Optimistic Locking）
# lock_versionカラムが必要
class Post < ApplicationRecord
  # lock_versionカラムが自動的に追加される
end

post1 = Post.find(1)
post2 = Post.find(1)

post1.update(title: "Updated by 1")
post2.update(title: "Updated by 2")  # ActiveRecord::StaleObjectError
```

### EXPLAIN（クエリの分析）

```ruby
# クエリプランを確認
Post.where(published: true).explain
# => EXPLAIN for: SELECT "posts".* FROM "posts" WHERE "posts"."published" = $1
#    Seq Scan on posts  (cost=0.00..10.00 rows=100 width=100)
#      Filter: (published = true)

# 詳細な分析
ActiveRecord::Base.connection.execute("EXPLAIN ANALYZE SELECT * FROM posts WHERE published = true")
```

### バッチ処理

```ruby
# find_each - バッチサイズを指定して処理
Post.find_each(batch_size: 1000) do |post|
  process_post(post)
end

# find_in_batches - バッチごとに配列として取得
Post.find_in_batches(batch_size: 1000) do |batch|
  batch.each { |post| process_post(post) }
end

# in_batches - より柔軟なバッチ処理
Post.in_batches(of: 1000) do |relation|
  relation.update_all(processed: true)
end
```

### 存在確認とカウント

```ruby
# exists? - 存在確認のみ（レコードを取得しない）
Post.exists?(1)
Post.exists?(title: "Hello")
Post.where(published: true).exists?

# count - カウントのみ
Post.count
Post.where(published: true).count

# size - キャッシュを考慮
posts = Post.where(published: true)
posts.size  # 必要に応じてクエリを実行
posts.load
posts.size  # キャッシュから取得
```

## 3.3 Active Record Associations（STI/Delegated Types含む）

### 参考資料
- [Rails Guides - Active Record Associations（英語）](https://guides.rubyonrails.org/association_basics.html)
- [Rails Guides - Active Record Associations（日本語）](https://railsguides.jp/association_basics.html)
- [Rails API - ActiveRecord::Associations](https://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html)

### has_many :throughの実装

コメント機能を追加して、ユーザーが投稿にコメントできるようにします。

```ruby
# マイグレーション
rails generate model Comment content:text post:references user:references
rails db:migrate
```

```ruby
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user

  validates :content, presence: true, length: { maximum: 1000 }
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  has_many :comments, dependent: :destroy
  has_many :commenters, through: :comments, source: :user
end
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :posts, dependent: :destroy
  has_many :comments, dependent: :destroy
  has_many :commented_posts, through: :comments, source: :post
end
```

### 多対多の関連（has_and_belongs_to_many）

タグ機能を追加します。

```ruby
# マイグレーション
rails generate model Tag name:string
rails generate migration CreateJoinTablePostsTags posts tags
rails db:migrate
```

```ruby
# app/models/tag.rb
class Tag < ApplicationRecord
  has_and_belongs_to_many :posts

  validates :name, presence: true, uniqueness: true
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_and_belongs_to_many :tags
end
```

### ポリモーフィック関連

いいね機能を追加します。

```ruby
# マイグレーション
rails generate model Like user:references likeable:references{polymorphic}
rails db:migrate
```

```ruby
# app/models/like.rb
class Like < ApplicationRecord
  belongs_to :user
  belongs_to :likeable, polymorphic: true

  validates :user_id, uniqueness: { scope: [:likeable_type, :likeable_id] }
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  has_many :likes, as: :likeable, dependent: :destroy
  has_many :likers, through: :likes, source: :user
end
```

### 関連付けのオプション

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # 条件付き関連
  has_many :published_comments, -> { where(approved: true) }, class_name: 'Comment'

  # 順序付け
  has_many :comments, -> { order(created_at: :desc) }

  # 依存関係
  has_many :comments, dependent: :destroy
  has_many :attachments, dependent: :delete_all

  # 外部キーの指定
  belongs_to :author, class_name: 'User', foreign_key: 'user_id'

  # カウンターキャッシュ
  belongs_to :user, counter_cache: true
end
```

### Single Table Inheritance（STI）

```ruby
# マイグレーション
rails generate migration CreateMessages content:text type:string
rails db:migrate
```

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  # typeカラムでサブクラスを区別
end

# app/models/text_message.rb
class TextMessage < Message
  # type = 'TextMessage'
end

# app/models/image_message.rb
class ImageMessage < Message
  # type = 'ImageMessage'
  has_one_attached :image
end

# 使用例
TextMessage.create(content: "Hello")
ImageMessage.create(content: "Photo", image: file)
Message.all  # すべてのメッセージを取得
```

### Delegated Types

```ruby
# マイグレーション
rails generate migration CreateEntries entryable:references{polymorphic} title:string
rails db:migrate
```

```ruby
# app/models/entry.rb
class Entry < ApplicationRecord
  delegated_type :entryable, types: %w[ Message Article ]
end

# app/models/entryable.rb
class Entryable < ApplicationRecord
  has_one :entry, as: :entryable, touch: true
end

# app/models/message.rb
class Message < Entryable
end

# app/models/article.rb
class Article < Entryable
end

# 使用例
Entry.create(entryable: Message.new(content: "Hello"), title: "New Message")
Entry.create(entryable: Article.new(body: "Content"), title: "New Article")
```

## 3.4 Active Record Validations（strict/条件付き/独自バリデーション）

### 参考資料
- [Rails Guides - Active Record Validations（英語）](https://guides.rubyonrails.org/active_record_validations.html)
- [Rails Guides - Active Record Validations（日本語）](https://railsguides.jp/active_record_validations.html)

### カスタムバリデーション

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validate :title_must_be_unique_for_user
  validate :content_must_not_contain_spam

  private

  def title_must_be_unique_for_user
    if user.posts.where(title: title).where.not(id: id).exists?
      errors.add(:title, 'は既に使用されています')
    end
  end

  def content_must_not_contain_spam
    spam_words = ['spam', 'advertisement']
    if content.present? && spam_words.any? { |word| content.downcase.include?(word) }
      errors.add(:content, 'にスパムが含まれています')
    end
  end
end
```

### 条件付きバリデーション

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true, if: :published?
  validates :content, presence: true, unless: :draft?

  def published?
    status == 'published'
  end

  def draft?
    status == 'draft'
  end
end
```

### バリデーションのグループ化

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  with_options if: :published? do
    validates :title, presence: true
    validates :content, presence: true
    validates :published_at, presence: true
  end
end
```

### Strictバリデーション

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # strict: true を指定すると、バリデーションエラー時に例外を発生
  validates :title, presence: true, strict: true
  # または
  validates! :title, presence: true
end

# 使用例
begin
  Post.create(title: "")  # ActiveRecord::RecordInvalid
rescue ActiveRecord::RecordInvalid => e
  puts e.message
end
```

### 独自バリデータークラス

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    unless value =~ URI::MailTo::EMAIL_REGEXP
      record.errors.add(attribute, :invalid, options.merge(value: value))
    end
  end
end

# app/models/user.rb
class User < ApplicationRecord
  validates :email, email: true
end
```

### バリデーションコンテキスト

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true, on: :create
  validates :content, presence: true, on: :update
  validates :slug, uniqueness: true, on: :publish
end

# 使用例
post = Post.new
post.valid?(:create)  # titleのバリデーションを実行
post.valid?(:publish)  # slugのバリデーションを実行
```

## 3.5 Active Record Callbacks（after_commit/after_rollback、トランザクション境界）

### 参考資料
- [Rails Guides - Active Record Callbacks（英語）](https://guides.rubyonrails.org/active_record_callbacks.html)
- [Rails Guides - Active Record Callbacks（日本語）](https://railsguides.jp/active_record_callbacks.html)

### コールバックの実装

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  before_validation :set_default_status
  after_create :notify_creation
  after_update :notify_update, if: :saved_change_to_title?
  before_destroy :check_deletable

  private

  def set_default_status
    self.status ||= 'draft'
  end

  def notify_creation
    PostCreatedNotificationJob.perform_later(id)
  end

  def notify_update
    PostUpdatedNotificationJob.perform_later(id)
  end

  def check_deletable
    if comments.any?
      errors.add(:base, 'コメントがあるため削除できません')
      throw :abort
    end
  end
end
```

### トランザクションコールバック

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # トランザクションがコミットされた後に実行
  after_commit :notify_creation, on: :create
  after_commit :notify_update, on: :update
  after_commit :notify_destroy, on: :destroy

  # トランザクションがロールバックされた後に実行
  after_rollback :log_rollback

  private

  def notify_creation
    PostCreatedNotificationJob.perform_later(id)
  end

  def notify_update
    PostUpdatedNotificationJob.perform_later(id)
  end

  def notify_destroy
    PostDestroyedNotificationJob.perform_later(id)
  end

  def log_rollback
    Rails.logger.warn("Post transaction rolled back: #{id}")
  end
end
```

### トランザクション境界の理解

```ruby
# 問題のあるコード
class Post < ApplicationRecord
  after_create :send_notification

  def send_notification
    # この時点ではトランザクションがまだコミットされていない
    # 外部サービスへの通知が失敗する可能性がある
    ExternalService.notify(self)
  end
end

# 正しいコード
class Post < ApplicationRecord
  after_commit :send_notification, on: :create

  def send_notification
    # トランザクションがコミットされた後に実行される
    ExternalService.notify(self)
  end
end
```

### コールバックの条件

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  # 条件付きコールバック
  after_update :notify_if_published, if: :published?

  # 複数の条件
  after_save :log_change, if: :saved_change_to_title?, unless: :skip_logging?

  private

  def notify_if_published
    # published? が true の場合のみ実行
  end

  def log_change
    # titleが変更された場合のみ実行
  end
end
```

## 3.6 Active Record Migrations（安全なスキーマ変更・運用の原則、ロールバック戦略）

### 参考資料
- [Rails Guides - Active Record Migrations（英語）](https://guides.rubyonrails.org/active_record_migrations.html)
- [Rails Guides - Active Record Migrations（日本語）](https://railsguides.jp/active_record_migrations.html)

### 複雑なマイグレーション

```ruby
# db/migrate/xxxxxx_add_full_text_search_to_posts.rb
class AddFullTextSearchToPosts < ActiveRecord::Migration[7.0]
  def up
    # PostgreSQLの全文検索インデックスを追加
    execute <<-SQL
      CREATE INDEX posts_full_text_search_idx ON posts
      USING gin(to_tsvector('japanese', title || ' ' || content));
    SQL
  end

  def down
    execute "DROP INDEX IF EXISTS posts_full_text_search_idx;"
  end
end
```

### データマイグレーション

```ruby
# db/migrate/xxxxxx_migrate_post_status.rb
class MigratePostStatus < ActiveRecord::Migration[7.0]
  def up
    Post.find_each do |post|
      if post.published
        post.update_column(:status, 'published')
      else
        post.update_column(:status, 'draft')
      end
    end
  end

  def down
    Post.find_each do |post|
      post.update_column(:published, post.status == 'published')
    end
  end
end
```

### スキーマの変更

```ruby
# db/migrate/xxxxxx_change_posts_content_to_text.rb
class ChangePostsContentToText < ActiveRecord::Migration[7.0]
  def change
    change_column :posts, :content, :text
  end
end
```

### 安全なスキーマ変更の原則

#### 1. ゼロダウンタイムデプロイ

```ruby
# ステップ1: 新しいカラムを追加（NULL許可）
class AddStatusToPosts < ActiveRecord::Migration[7.0]
  def change
    add_column :posts, :status, :string, null: true
  end
end

# ステップ2: データを移行
class MigratePostStatus < ActiveRecord::Migration[7.0]
  def up
    Post.find_each do |post|
      post.update_column(:status, post.published? ? 'published' : 'draft')
    end
  end
end

# ステップ3: NOT NULL制約を追加
class AddNotNullToPostStatus < ActiveRecord::Migration[7.0]
  def change
    change_column_null :posts, :status, false
  end
end
```

#### 2. インデックスの追加

```ruby
# 同時実行ロックを避けるため、CONCURRENTLYを使用（PostgreSQL）
class AddIndexToPostsTitle < ActiveRecord::Migration[7.0]
  disable_ddl_transaction!

  def change
    add_index :posts, :title, algorithm: :concurrently
  end
end
```

#### 3. 外部キー制約の追加

```ruby
# 既存データを確認してから追加
class AddForeignKeyToComments < ActiveRecord::Migration[7.0]
  def change
    # 孤立レコードがないことを確認
    execute <<-SQL
      DELETE FROM comments WHERE post_id NOT IN (SELECT id FROM posts);
    SQL

    add_foreign_key :comments, :posts, on_delete: :cascade
  end
end
```

### ロールバック戦略

```ruby
# 安全なロールバックを実装
class AddColumnToPosts < ActiveRecord::Migration[7.0]
  def up
    add_column :posts, :new_column, :string
    # データ移行
    Post.find_each do |post|
      post.update_column(:new_column, calculate_value(post))
    end
  end

  def down
    # ロールバック時の処理
    remove_column :posts, :new_column
  end
end
```

### データマイグレーションのベストプラクティス

```ruby
# バッチ処理で安全に実行
class MigratePostData < ActiveRecord::Migration[7.0]
  def up
    Post.find_each do |post|
      post.update_column(:status, 'published') if post.published?
    end
  end

  def down
    Post.find_each do |post|
      post.update_column(:published, post.status == 'published')
    end
  end
end
```

### マイグレーションの検証

```ruby
# マイグレーション実行前の検証
class AddIndexToPosts < ActiveRecord::Migration[7.0]
  def up
    # インデックスが既に存在しないことを確認
    unless index_exists?(:posts, :title)
      add_index :posts, :title
    end
  end

  def down
    if index_exists?(:posts, :title)
      remove_index :posts, :title
    end
  end
end
```

## 3.7 Multiple Databases（読み書き分離、接続切替、シャーディング）

### 参考資料
- [Rails Guides - Multiple Databases（英語）](https://guides.rubyonrails.org/active_record_multiple_databases.html)
- [Rails Guides - Multiple Databases（日本語）](https://railsguides.jp/active_record_multiple_databases.html)

### データベース設定

```yaml
# config/database.yml
production:
  primary:
    <<: *default
    database: sample_app_production
    migrations_paths: db/migrate
  replica:
    <<: *default
    database: sample_app_production_replica
    replica: true
    migrations_paths: db/migrate
```

### 読み書き分離

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to database: { writing: :primary, reading: :replica }
end

# 使用例
# 読み取り専用（レプリカから読み取り）
Post.connected_to(role: :reading) do
  @posts = Post.all
end

# 書き込み（プライマリに書き込み）
Post.connected_to(role: :writing) do
  @post = Post.create(title: "New Post")
end
```

### 自動的な読み書き分離

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.active_record.reads = true
  end
end

# 自動的に読み取りはレプリカ、書き込みはプライマリ
Post.all  # レプリカから読み取り
Post.create(...)  # プライマリに書き込み
```

### シャーディング

```yaml
# config/database.yml
production:
  primary:
    <<: *default
    database: sample_app_production
  shard_one:
    <<: *default
    database: sample_app_production_shard_one
  shard_two:
    <<: *default
    database: sample_app_production_shard_two
```

```ruby
# app/models/application_record.rb
class ApplicationRecord < ActiveRecord::Base
  self.abstract_class = true

  connects_to shards: {
    default: { writing: :primary },
    shard_one: { writing: :shard_one },
    shard_two: { writing: :shard_two }
  }
end

# 使用例
Post.connected_to(shard: :shard_one) do
  @posts = Post.all
end
```

## 3.8 Active Record Encryption（PII保護・移行・共存パターン）

### 参考資料
- [Rails Guides - Active Record Encryption（英語）](https://guides.rubyonrails.org/active_record_encryption.html)
- [Rails Guides - Active Record Encryption（日本語）](https://railsguides.jp/active_record_encryption.html)

### 暗号化の設定

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.active_record.encryption.primary_key = ENV['AR_ENCRYPTION_PRIMARY_KEY']
    config.active_record.encryption.deterministic_key = ENV['AR_ENCRYPTION_DETERMINISTIC_KEY']
    config.active_record.encryption.key_derivation_salt = ENV['AR_ENCRYPTION_KEY_DERIVATION_SALT']
  end
end
```

### 属性の暗号化

```ruby
# app/models/user.rb
class User < ApplicationRecord
  # 暗号化された属性
  encrypts :email
  encrypts :phone_number, deterministic: true  # 検索可能

  # 暗号化された属性の検索
  User.find_by(email: "user@example.com")  # 暗号化された値で検索
end
```

### 移行パターン

```ruby
# 既存のデータを暗号化するマイグレーション
class EncryptUserEmails < ActiveRecord::Migration[7.0]
  def up
    User.find_each do |user|
      user.update_column(:email, user.email)  # 暗号化される
    end
  end
end
```

### 共存パターン

```ruby
# 暗号化と非暗号化の両方をサポート
class User < ApplicationRecord
  # 暗号化された属性
  encrypts :email, previous: { deterministic: true }

  # 移行期間中は両方の形式をサポート
  def email
    super || read_attribute(:email)
  end
end
```

## 3.9 実践的な例：検索機能の実装

### 全文検索の実装

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  def self.full_text_search(query)
    return all if query.blank?

    where(
      "to_tsvector('japanese', title || ' ' || content) @@ plainto_tsquery('japanese', ?)",
      query
    )
  end
end
```

### 複合検索の実装

```ruby
# app/queries/posts_query.rb
class PostsQuery
  def initialize(relation = Post.all)
    @relation = relation
  end

  def call
    @relation
      .then { |r| apply_includes(r) }
      .then { |r| apply_filters(r) }
      .then { |r| apply_search(r) }
      .then { |r| apply_sorting(r) }
      .then { |r| apply_pagination(r) }
  end

  def with_user(user)
    @user = user
    self
  end

  def with_tags(tag_ids)
    @tag_ids = tag_ids
    self
  end

  def search(query)
    @search_query = query
    self
  end

  def order_by(field, direction = :desc)
    @order_field = field
    @order_direction = direction
    self
  end

  def page(page_number)
    @page = page_number
    self
  end

  private

  def apply_filters(relation)
    relation = relation.where(user: @user) if @user
    relation = relation.joins(:tags).where(tags: { id: @tag_ids }) if @tag_ids.present?
    relation
  end

  def apply_search(relation)
    return relation unless @search_query

    relation.full_text_search(@search_query)
  end

  # ... 既存のメソッド ...
end
```

## 3.10 確認ポイント

この章の最後に、以下を確認してください：

1. **Active Record Basicsが理解できている**
   - CRUD操作が正しく実行できる
   - 属性の操作が理解できている

2. **Query Interfaceが理解できている**
   - locking、EXPLAINが使用できる
   - バッチ処理が適切に実装されている

3. **Associationsが正しく実装されている**
   - STI、Delegated Typesが理解できている

4. **Validationsが動作している**
   - strictバリデーションが機能している
   - 独自バリデーターが作成できる

5. **Callbacksが動作している**
   - after_commit/after_rollbackが理解できている
   - トランザクション境界が理解できている

6. **Migrationsが安全に実行できる**
   - ゼロダウンタイムデプロイが理解できている
   - ロールバック戦略が理解できている

7. **Multiple Databasesが設定できる**
   - 読み書き分離が実装できる

8. **Active Record Encryptionが設定できる**
   - PII保護が実装できる

## 3.11 参考資料

- [Rails Guides - Active Record Basics（英語）](https://guides.rubyonrails.org/active_record_basics.html)
- [Rails Guides - Active Record Query Interface（英語）](https://guides.rubyonrails.org/active_record_querying.html)
- [Rails Guides - Active Record Associations（英語）](https://guides.rubyonrails.org/association_basics.html)
- [Rails Guides - Active Record Migrations（英語）](https://guides.rubyonrails.org/active_record_migrations.html)
- [Rails Guides - Active Record Validations（英語）](https://guides.rubyonrails.org/active_record_validations.html)
- [Rails Guides - Active Record Callbacks（英語）](https://guides.rubyonrails.org/active_record_callbacks.html)
- [Rails Guides - Multiple Databases（英語）](https://guides.rubyonrails.org/active_record_multiple_databases.html)
- [Rails Guides - Active Record Encryption（英語）](https://guides.rubyonrails.org/active_record_encryption.html)

## 3.12 次の章へ

この章で、Active Recordの包括的な機能を学びました。次の章では、Action ControllerとAction Viewの詳細を学びます。

**次の章**: [第4章: Action ControllerとAction Viewの詳細](chapter04-action-controller-view.md)

## まとめ

この章では以下を学びました：

- **Active Record Basics**: 基礎を正しい前提に戻す
- **Query Interface**: N+1、eager loading、locking、EXPLAIN
- **Associations**: STI、Delegated Typesを含む設計パターン
- **Migrations**: 安全なスキーマ変更・運用の原則、ロールバック戦略
- **Validations**: strict/条件付き/独自バリデーション
- **Callbacks**: after_commit/after_rollback、トランザクション境界
- **Multiple Databases**: 読み書き分離、接続切替、シャーディング
- **Active Record Encryption**: PII保護・移行・共存パターン
