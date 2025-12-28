# 第1章: 基本的なアプリケーション構築

## はじめに

この章では、第0章で作成したアプリケーションに基本的な機能を追加します。ユーザー認証、投稿機能、基本的なCRUD操作を実装しながら、Railsの基本的な使い方を学びます。

## 1.1 ユーザーモデルの作成

### ユーザーモデルの生成

```bash
rails generate model User name:string email:string
rails db:migrate
```

### バリデーションの追加

```ruby
# app/models/user.rb
class User < ApplicationRecord
  validates :name, presence: true, length: { maximum: 50 }
  validates :email, presence: true, 
                    length: { maximum: 255 },
                    format: { with: URI::MailTo::EMAIL_REGEXP },
                    uniqueness: { case_sensitive: false }
end
```

### コンソールでの確認

```bash
rails console
```

```ruby
# ユーザーの作成
user = User.create(name: "Example User", email: "example@example.com")

# バリデーションの確認
user.valid?
user.errors.full_messages

# 無効なユーザーの作成
invalid_user = User.create(name: "", email: "invalid")
invalid_user.valid?
invalid_user.errors.full_messages
```

## 1.2 Deviseによる認証機能の実装

### Deviseのインストール

```ruby
# Gemfile
gem 'devise'
```

```bash
bundle install
rails generate devise:install
```

### 設定ファイルの確認

`config/initializers/devise.rb`が生成されます。必要に応じて設定を調整します。

### ユーザーモデルへのDevise追加

既存のUserモデルにDeviseを追加します：

```bash
rails generate devise User
```

既存のUserモデルがある場合、マイグレーションファイルを確認して、必要なカラムを追加します：

```ruby
# db/migrate/xxxxxx_add_devise_to_users.rb
class AddDeviseToUsers < ActiveRecord::Migration[7.0]
  def self.up
    change_table :users do |t|
      ## Database authenticatable
      t.string :encrypted_password, null: false, default: ""

      ## Recoverable
      t.string   :reset_password_token
      t.datetime :reset_password_sent_at

      ## Rememberable
      t.datetime :remember_created_at

      ## Trackable
      t.integer  :sign_in_count, default: 0, null: false
      t.datetime :current_sign_in_at
      t.datetime :last_sign_in_at
      t.string   :current_sign_in_ip
      t.string   :last_sign_in_ip

      ## Confirmable
      # t.string   :confirmation_token
      # t.datetime :confirmed_at
      # t.datetime :confirmation_sent_at
      # t.string   :unconfirmed_email

      ## Lockable
      # t.integer  :failed_attempts, default: 0, null: false
      # t.string   :unlock_token
      # t.datetime :locked_at
    end

    add_index :users, :email,                unique: true
    add_index :users, :reset_password_token, unique: true
    # add_index :users, :confirmation_token,   unique: true
    # add_index :users, :unlock_token,         unique: true
  end

  def self.down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

```bash
rails db:migrate
```

### Userモデルの更新

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :trackable

  validates :name, presence: true, length: { maximum: 50 }
end
```

### ルーティングの確認

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users
  root 'home#index'
end
```

### ビューの生成（オプション）

```bash
rails generate devise:views
```

## 1.3 投稿（Post）モデルの作成

### Postモデルの生成

```bash
rails generate model Post title:string content:text user:references
rails db:migrate
```

### アソシエーションの設定

```ruby
# app/models/user.rb
class User < ApplicationRecord
  devise :database_authenticatable, :registerable,
         :recoverable, :rememberable, :validatable,
         :trackable

  validates :name, presence: true, length: { maximum: 50 }
  
  has_many :posts, dependent: :destroy
end
```

```ruby
# app/models/post.rb
class Post < ApplicationRecord
  belongs_to :user
  
  validates :title, presence: true, length: { maximum: 255 }
  validates :content, presence: true, length: { maximum: 10000 }
end
```

## 1.4 Postsコントローラーの作成

### コントローラーの生成

```bash
rails generate controller Posts index show new create edit update destroy
```

### コントローラーの実装

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]

  def index
    @posts = Post.includes(:user).order(created_at: :desc).page(params[:page])
  end

  def show
  end

  def new
    @post = current_user.posts.build
  end

  def create
    @post = current_user.posts.build(post_params)
    
    if @post.save
      redirect_to @post, notice: '投稿が作成されました。'
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    authorize @post
  end

  def update
    authorize @post
    
    if @post.update(post_params)
      redirect_to @post, notice: '投稿が更新されました。'
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    authorize @post
    @post.destroy
    redirect_to posts_path, notice: '投稿が削除されました。'
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def post_params
    params.require(:post).permit(:title, :content)
  end
end
```

### ルーティングの追加

```ruby
# config/routes.rb
Rails.application.routes.draw do
  devise_for :users
  resources :posts
  root 'posts#index'
end
```

## 1.5 ビューの作成

### レイアウトファイルの更新

```erb
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
  <head>
    <title>Sample App</title>
    <meta name="viewport" content="width=device-width,initial-scale=1">
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>

    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body>
    <nav>
      <div>
        <%= link_to "Sample App", root_path %>
        <% if user_signed_in? %>
          <%= link_to "新規投稿", new_post_path %>
          <%= link_to "ログアウト", destroy_user_session_path, data: { turbo_method: :delete } %>
          <span>こんにちは、<%= current_user.name %>さん</span>
        <% else %>
          <%= link_to "ログイン", new_user_session_path %>
          <%= link_to "新規登録", new_user_registration_path %>
        <% end %>
      </div>
    </nav>

    <% if notice %>
      <p class="notice"><%= notice %></p>
    <% end %>
    <% if alert %>
      <p class="alert"><%= alert %></p>
    <% end %>

    <%= yield %>
  </body>
</html>
```

### 投稿一覧ページ

```erb
<!-- app/views/posts/index.html.erb -->
<h1>投稿一覧</h1>

<% @posts.each do |post| %>
  <article>
    <h2><%= link_to post.title, post %></h2>
    <p><%= truncate(post.content, length: 100) %></p>
    <p>
      投稿者: <%= post.user.name %>
      投稿日時: <%= post.created_at.strftime("%Y年%m月%d日 %H:%M") %>
    </p>
  </article>
<% end %>

<%= paginate @posts if defined?(Kaminari) %>
```

### 投稿詳細ページ

```erb
<!-- app/views/posts/show.html.erb -->
<article>
  <h1><%= @post.title %></h1>
  <p><%= simple_format(@post.content) %></p>
  <p>
    投稿者: <%= @post.user.name %>
    投稿日時: <%= @post.created_at.strftime("%Y年%m月%d日 %H:%M") %>
  </p>

  <% if user_signed_in? && current_user == @post.user %>
    <%= link_to "編集", edit_post_path(@post) %>
    <%= link_to "削除", post_path(@post), 
                data: { turbo_method: :delete, turbo_confirm: "削除しますか？" } %>
  <% end %>
</article>

<%= link_to "一覧に戻る", posts_path %>
```

### 新規投稿フォーム

```erb
<!-- app/views/posts/new.html.erb -->
<h1>新規投稿</h1>

<%= form_with model: @post, local: true do |form| %>
  <% if @post.errors.any? %>
    <div>
      <h2><%= pluralize(@post.errors.count, "件のエラー") %>があります:</h2>
      <ul>
        <% @post.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :title, "タイトル" %>
    <%= form.text_field :title %>
  </div>

  <div>
    <%= form.label :content, "内容" %>
    <%= form.text_area :content, rows: 10 %>
  </div>

  <div>
    <%= form.submit "投稿する" %>
  </div>
<% end %>

<%= link_to "一覧に戻る", posts_path %>
```

### 編集フォーム

```erb
<!-- app/views/posts/edit.html.erb -->
<h1>投稿を編集</h1>

<%= form_with model: @post, local: true do |form| %>
  <% if @post.errors.any? %>
    <div>
      <h2><%= pluralize(@post.errors.count, "件のエラー") %>があります:</h2>
      <ul>
        <% @post.errors.full_messages.each do |message| %>
          <li><%= message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div>
    <%= form.label :title, "タイトル" %>
    <%= form.text_field :title %>
  </div>

  <div>
    <%= form.label :content, "内容" %>
    <%= form.text_area :content, rows: 10 %>
  </div>

  <div>
    <%= form.submit "更新する" %>
  </div>
<% end %>

<%= link_to "詳細に戻る", @post %>
<%= link_to "一覧に戻る", posts_path %>
```

## 1.6 認可機能の実装（簡易版）

### ApplicationControllerの更新

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :authenticate_user!
  
  private

  def authorize_post_owner
    @post = Post.find(params[:id])
    redirect_to posts_path, alert: '権限がありません。' unless @post.user == current_user
  end
end
```

### PostsControllerの更新

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post_owner, only: [:edit, :update, :destroy]

  # ... 既存のコード ...
end
```

## 1.7 シードデータの作成

### シードファイルの作成

```ruby
# db/seeds.rb
# ユーザーの作成
user1 = User.create!(
  name: "山田太郎",
  email: "yamada@example.com",
  password: "password123",
  password_confirmation: "password123"
)

user2 = User.create!(
  name: "佐藤花子",
  email: "sato@example.com",
  password: "password123",
  password_confirmation: "password123"
)

# 投稿の作成
5.times do |i|
  Post.create!(
    title: "サンプル投稿 #{i + 1}",
    content: "これはサンプル投稿です。内容は#{i + 1}番目の投稿です。",
    user: [user1, user2].sample
  )
end

puts "シードデータの作成が完了しました。"
```

### シードデータの投入

```bash
rails db:seed
```

## 1.8 確認ポイント

この章の最後に、以下を確認してください：

1. **ユーザー登録ができる**
   - `http://localhost:3000/users/sign_up` にアクセス
   - 新規ユーザーを登録できる

2. **ログイン・ログアウトができる**
   - `http://localhost:3000/users/sign_in` にアクセス
   - ログイン・ログアウトが正常に動作する

3. **投稿のCRUD操作ができる**
   - 新規投稿が作成できる
   - 投稿一覧が表示される
   - 投稿詳細が表示される
   - 投稿の編集ができる（自分の投稿のみ）
   - 投稿の削除ができる（自分の投稿のみ）

4. **認可が正しく動作する**
   - 他のユーザーの投稿は編集・削除できない

## 1.9 トラブルシューティング

### Deviseのルーティングエラー

```bash
# ルーティングを確認
rails routes | grep devise
```

### マイグレーションエラー

```bash
# マイグレーションをリセット（開発環境のみ）
rails db:migrate:reset
rails db:seed
```

### アソシエーションエラー

```ruby
# コンソールで確認
rails console
user = User.first
user.posts
```

## 1.10 次の章へ

この章で基本的なアプリケーションの構築が完了しました。次の章では、このアプリケーションを拡張しながら、高度なRailsアーキテクチャと設計パターンを学びます。

**次の章**: [第3章: Active Recordの高度な活用](chapter03-active-record-advanced.md)

## まとめ

この章では以下を学びました：

- Deviseによる認証機能の実装
- モデル、コントローラー、ビューの基本的な実装
- 基本的なCRUD操作
- 認可機能の実装
- シードデータの作成

次の章では、このアプリケーションを拡張しながら、より高度な設計パターンを学びます。
