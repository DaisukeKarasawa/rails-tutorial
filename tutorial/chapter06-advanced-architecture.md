# 第2章: 高度なRailsアーキテクチャと設計パターン

## はじめに

この章では、第1章で作成した基本的なアプリケーションを拡張しながら、より保守性が高く、テストしやすいコードを書くための設計パターンを学びます。

現在のアプリケーションには、UserモデルとPostモデルがあり、基本的なCRUD操作が実装されています。この章では、これらのコードをリファクタリングしながら、様々な設計パターンを実装します。

## 2.1 現在のコードの問題点を確認

### PostsControllerの確認

現在の`app/controllers/posts_controller.rb`を確認してください。コントローラーにビジネスロジックが含まれている可能性があります。

### リファクタリングの目標

- コントローラーをシンプルにする
- ビジネスロジックをモデルやサービスから分離する
- テストしやすいコードにする
- 再利用可能なコードにする

## 2.2 Service Objectパターンの実装

### 目的

ビジネスロジックをコントローラーやモデルから分離し、再利用可能でテストしやすいコードを書くためのパターンです。

### Post作成サービスの実装

現在の`PostsController#create`メソッドを確認し、ビジネスロジックをService Objectに移動します。

```ruby
# app/services/post_creation_service.rb
class PostCreationService
  def initialize(user, post_params)
    @user = user
    @post_params = post_params
  end

  def call
    return failure('Invalid parameters') unless valid_params?

    ActiveRecord::Base.transaction do
      post = create_post
      notify_followers(post) if should_notify?
      success(post)
    end
  rescue StandardError => e
    failure(e.message)
  end

  private

  attr_reader :user, :post_params

  def valid_params?
    post_params[:title].present? && post_params[:content].present?
  end

  def create_post
    user.posts.create!(post_params)
  end

  def should_notify?
    # 将来的にフォロワー通知機能を追加する場合の準備
    false
  end

  def notify_followers(post)
    # 将来的な実装
  end

  def success(post)
    OpenStruct.new(success?: true, post: post, errors: [])
  end

  def failure(message)
    OpenStruct.new(success?: false, post: nil, errors: [message])
  end
end
```

### PostsControllerの更新

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
    service = PostCreationService.new(current_user, post_params)
    result = service.call

    if result.success?
      redirect_to result.post, notice: '投稿が作成されました。'
    else
      @post = current_user.posts.build(post_params)
      flash.now[:alert] = result.errors.join(', ')
      render :new, status: :unprocessable_entity
    end
  end

  # ... 既存のコード ...
end
```

### テストの作成

```ruby
# spec/services/post_creation_service_spec.rb
require 'rails_helper'

RSpec.describe PostCreationService do
  let(:user) { create(:user) }
  let(:post_params) { { title: 'Test Post', content: 'Test content' } }
  let(:service) { described_class.new(user, post_params) }

  describe '#call' do
    context 'with valid parameters' do
      it 'creates a post' do
        expect { service.call }.to change(Post, :count).by(1)
      end

      it 'returns success result' do
        result = service.call
        expect(result.success?).to be true
        expect(result.post).to be_a(Post)
      end

      it 'associates post with user' do
        result = service.call
        expect(result.post.user).to eq(user)
      end
    end

    context 'with invalid parameters' do
      let(:post_params) { { title: '', content: '' } }

      it 'does not create a post' do
        expect { service.call }.not_to change(Post, :count)
      end

      it 'returns failure result' do
        result = service.call
        expect(result.success?).to be false
        expect(result.errors).not_to be_empty
      end
    end
  end
end
```

## 2.3 Query Objectパターンの実装

### 目的

複雑なデータベースクエリをモデルから分離し、再利用可能でテストしやすい形にするパターンです。

### PostsQueryの実装

現在の`PostsController#index`のクエリをQuery Objectに移動します。

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
      .then { |r| apply_sorting(r) }
      .then { |r| apply_pagination(r) }
  end

  def with_user(user)
    @user = user
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

  def per(per_page)
    @per_page = per_page
    self
  end

  private

  attr_reader :user, :search_query, :order_field, :order_direction, :page, :per_page

  def apply_includes(relation)
    relation.includes(:user)
  end

  def apply_filters(relation)
    relation = relation.where(user: user) if user
    relation = apply_search(relation) if search_query
    relation
  end

  def apply_search(relation)
    relation.where(
      'title ILIKE ? OR content ILIKE ?',
      "%#{search_query}%",
      "%#{search_query}%"
    )
  end

  def apply_sorting(relation)
    field = order_field || :created_at
    direction = order_direction || :desc
    relation.order(field => direction)
  end

  def apply_pagination(relation)
    return relation unless page

    relation.page(page).per(per_page || 25)
  end
end
```

### PostsControllerの更新

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = PostsQuery.new
                       .search(params[:q])
                       .order_by(:created_at, :desc)
                       .page(params[:page])
                       .per(25)
                       .call
  end

  # ... 既存のコード ...
end
```

### テストの作成

```ruby
# spec/queries/posts_query_spec.rb
require 'rails_helper'

RSpec.describe PostsQuery do
  let!(:user1) { create(:user) }
  let!(:user2) { create(:user) }
  let!(:post1) { create(:post, user: user1, title: 'First Post') }
  let!(:post2) { create(:post, user: user2, title: 'Second Post') }

  describe '#call' do
    it 'returns all posts by default' do
      result = described_class.new.call
      expect(result).to include(post1, post2)
    end

    it 'filters by user' do
      result = described_class.new.with_user(user1).call
      expect(result).to include(post1)
      expect(result).not_to include(post2)
    end

    it 'searches by title' do
      result = described_class.new.search('First').call
      expect(result).to include(post1)
      expect(result).not_to include(post2)
    end

    it 'orders by created_at desc by default' do
      result = described_class.new.call
      expect(result.first).to eq(post2)  # より新しい投稿が先頭
    end
  end
end
```

## 2.4 Form Objectパターンの実装

### 目的

複雑なバリデーションロジックや、複数のモデルにまたがるフォームを扱うためのパターンです。

### PostFormの実装

将来的にタグ機能を追加することを想定して、Form Objectを作成します。

```ruby
# app/forms/post_form.rb
class PostForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :title, :string
  attribute :content, :text
  attribute :tag_names, :string

  validates :title, presence: true, length: { maximum: 255 }
  validates :content, presence: true, length: { maximum: 10000 }
  validate :title_uniqueness_for_user

  def initialize(user, post = nil, attributes = {})
    @user = user
    @post = post
    super(attributes)
  end

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      if @post
        update_post
      else
        create_post
      end
      true
    end
  rescue StandardError => e
    errors.add(:base, e.message)
    false
  end

  private

  attr_reader :user, :post

  def title_uniqueness_for_user
    return unless title.present?

    existing_post = user.posts.where(title: title)
    existing_post = existing_post.where.not(id: post.id) if post

    if existing_post.exists?
      errors.add(:title, 'は既に使用されています')
    end
  end

  def create_post
    @post = user.posts.create!(
      title: title,
      content: content
    )
  end

  def update_post
    @post.update!(
      title: title,
      content: content
    )
  end
end
```

### PostsControllerの更新

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def new
    @form = PostForm.new(current_user)
  end

  def create
    @form = PostForm.new(current_user, nil, post_params)

    if @form.save
      redirect_to @form.post, notice: '投稿が作成されました。'
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    @form = PostForm.new(current_user, @post)
  end

  def update
    @form = PostForm.new(current_user, @post, post_params)

    if @form.save
      redirect_to @form.post, notice: '投稿が更新されました。'
    else
      render :edit, status: :unprocessable_entity
    end
  end

  # ... 既存のコード ...
end
```

### ビューの更新

```erb
<!-- app/views/posts/new.html.erb -->
<h1>新規投稿</h1>

<%= form_with model: @form, url: posts_path, local: true do |form| %>
  <% if @form.errors.any? %>
    <div>
      <h2><%= pluralize(@form.errors.count, "件のエラー") %>があります:</h2>
      <ul>
        <% @form.errors.full_messages.each do |message| %>
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
```

## 2.5 Decoratorパターンの実装

### 目的

ビューロジックをモデルから分離し、表示に関する責任を明確にするパターンです。

### Draperのセットアップ

```ruby
# Gemfile
gem 'draper'
```

```bash
bundle install
rails generate draper:install
```

### PostDecoratorの実装

```ruby
# app/decorators/post_decorator.rb
class PostDecorator < Draper::Decorator
  delegate_all

  def formatted_created_at
    created_at.strftime('%Y年%m月%d日 %H:%M')
  end

  def truncated_content(length: 100)
    if content.length > length
      "#{content[0, length]}..."
    else
      content
    end
  end

  def author_name
    user.name
  end

  def time_ago
    time_ago_in_words(created_at) + '前'
  end
end
```

### コントローラーでの使用

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = PostsQuery.new.call.map(&:decorate)
  end

  def show
    @post = @post.decorate
  end
end
```

### ビューでの使用

```erb
<!-- app/views/posts/index.html.erb -->
<% @posts.each do |post| %>
  <article>
    <h2><%= link_to post.title, post %></h2>
    <p><%= post.truncated_content %></p>
    <p>
      投稿者: <%= post.author_name %>
      投稿日時: <%= post.formatted_created_at %>
      (<%= post.time_ago %>)
    </p>
  </article>
<% end %>
```

## 2.6 既存コードのリファクタリング

### リファクタリングの手順

1. **テストを書く**（既存の動作を保証）
2. **小さな変更を加える**
3. **テストを実行して確認**
4. **次の変更に進む**

### PostsControllerの最終的な形

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post_owner, only: [:edit, :update, :destroy]

  def index
    @posts = PostsQuery.new
                       .search(params[:q])
                       .order_by(:created_at, :desc)
                       .page(params[:page])
                       .per(25)
                       .call
                       .map(&:decorate)
  end

  def show
    @post = @post.decorate
  end

  def new
    @form = PostForm.new(current_user)
  end

  def create
    @form = PostForm.new(current_user, nil, post_params)

    if @form.save
      redirect_to @form.post, notice: '投稿が作成されました。'
    else
      render :new, status: :unprocessable_entity
    end
  end

  def edit
    @form = PostForm.new(current_user, @post)
  end

  def update
    @form = PostForm.new(current_user, @post, post_params)

    if @form.save
      redirect_to @form.post, notice: '投稿が更新されました。'
    else
      render :edit, status: :unprocessable_entity
    end
  end

  def destroy
    @post.destroy
    redirect_to posts_path, notice: '投稿が削除されました。'
  end

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def authorize_post_owner
    redirect_to posts_path, alert: '権限がありません。' unless @post.user == current_user
  end

  def post_params
    params.require(:post_form).permit(:title, :content, :tag_names)
  end
end
```

## 2.7 Engines / Railties（モノリス分割、プラグイン化、拡張ポイント理解）

### 参考資料
- [Rails Edge Guides - Engines（英語）](https://edgeguides.rubyonrails.org/engines.html)

### Enginesとは

Enginesは、Railsアプリケーションの機能を再利用可能な形にしたものです。モノリスを分割したり、プラグインとして機能を提供する際に使用します。

### Engineの作成

```bash
rails plugin new blog_engine --mountable
```

### Engineの構造

```
blog_engine/
  app/
    controllers/
      blog_engine/
        posts_controller.rb
    models/
      blog_engine/
        post.rb
    views/
      blog_engine/
        posts/
          index.html.erb
  config/
    routes.rb
  lib/
    blog_engine/
      engine.rb
  blog_engine.gemspec
```

### Engineの実装

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine

    config.generators do |g|
      g.test_framework :rspec
      g.fixture_replacement :factory_bot
    end
  end
end
```

```ruby
# config/routes.rb
BlogEngine::Engine.routes.draw do
  resources :posts
end
```

### メインアプリケーションでの使用

```ruby
# Gemfile
gem 'blog_engine', path: '../blog_engine'
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  mount BlogEngine::Engine, at: '/blog'
end
```

### モノリス分割の例

```ruby
# 大きなアプリケーションを分割
# メインアプリケーション
Rails.application.routes.draw do
  mount BlogEngine::Engine, at: '/blog'
  mount AdminEngine::Engine, at: '/admin'
  mount ApiEngine::Engine, at: '/api'
end
```

### Railtieとは

Railtieは、Railsアプリケーションに機能を追加するための仕組みです。Engineの基盤でもあります。

### Railtieの実装

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

    # ミドルウェアを追加
    config.middleware.use MyGem::Middleware

    # Active Recordの拡張
    ActiveSupport.on_load(:active_record) do
      include MyGem::ActiveRecordExtension
    end
  end
end
```

### 拡張ポイントの理解

```ruby
# Active Recordの拡張
ActiveSupport.on_load(:active_record) do
  # Active Recordが読み込まれた後に実行
  include MyGem::ActiveRecordExtension
end

# Action Controllerの拡張
ActiveSupport.on_load(:action_controller) do
  # Action Controllerが読み込まれた後に実行
  include MyGem::ControllerExtension
end

# Action Viewの拡張
ActiveSupport.on_load(:action_view) do
  # Action Viewが読み込まれた後に実行
  include MyGem::ViewExtension
end
```

### Engineの設定

```ruby
# lib/blog_engine/engine.rb
module BlogEngine
  class Engine < ::Rails::Engine
    isolate_namespace BlogEngine

    # 設定の追加
    config.blog_engine = ActiveSupport::OrderedOptions.new
    config.blog_engine.posts_per_page = 25

    # 初期化
    initializer "blog_engine.configure" do |app|
      BlogEngine.configure do |config|
        config.posts_per_page = app.config.blog_engine.posts_per_page
      end
    end
  end
end
```

### Engineのテスト

```ruby
# spec/dummy/config/routes.rb
Rails.application.routes.draw do
  mount BlogEngine::Engine, at: '/blog'
end

# spec/integration/posts_spec.rb
RSpec.describe "BlogEngine::Posts", type: :request do
  it "displays posts" do
    get "/blog/posts"
    expect(response).to be_successful
  end
end
```

### Engineのベストプラクティス

1. **名前空間の分離**: `isolate_namespace`を使用
2. **設定の外部化**: 設定を外部から変更可能にする
3. **依存関係の明確化**: gemspecで依存関係を明示
4. **テストの分離**: dummyアプリケーションでテスト

## 2.8 確認ポイント

この章の最後に、以下を確認してください：

1. **既存の機能が正常に動作する**
   - 投稿の作成、編集、削除ができる
   - 投稿一覧が表示される
   - 投稿詳細が表示される

2. **新しいパターンが適用されている**
   - Service Objectが使用されている
   - Query Objectが使用されている
   - Form Objectが使用されている
   - Decoratorが使用されている

3. **Engineが理解できている**
   - Engineの構造が理解できている
   - Railtieの拡張ポイントが理解できている

4. **テストが通る**
   ```bash
   bundle exec rspec
   ```

## 2.9 トラブルシューティング

### Draperのエラー

```bash
# Draperを再インストール
rails generate draper:install
```

### テストのエラー

```bash
# テストデータベースをリセット
rails db:test:prepare
```

## 2.10 参考資料

- [Rails Edge Guides - Engines（英語）](https://edgeguides.rubyonrails.org/engines.html)

## 2.11 次の章へ

この章で、既存のアプリケーションをリファクタリングしながら、高度な設計パターンを実装しました。次の章では、パフォーマンス最適化に取り組みます。

**次の章**: [第7章: パフォーマンス最適化とスケーリング](chapter07-performance-optimization.md)

## まとめ

この章では以下を学びました：

- **Service Object**: ビジネスロジックの分離
- **Query Object**: 複雑なクエリの分離
- **Form Object**: 複雑なフォーム処理
- **Decorator**: ビューロジックの分離
- **Engines / Railties**: モノリス分割、プラグイン化、拡張ポイント理解
- **リファクタリング**: 既存コードの改善

これらのパターンを適用することで、保守性が高く、テストしやすいコードになりました。
