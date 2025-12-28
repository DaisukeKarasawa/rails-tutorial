# 第3章: Action ControllerとAction Viewの詳細

## はじめに

この章では、[Rails Guides - Action Controller Overview](https://guides.rubyonrails.org/action_controller_overview.html)と[Rails Guides - Action View Overview](https://guides.rubyonrails.org/action_view_overview.html)を参考に、Action ControllerとAction Viewの高度な機能を学びます。

## 3.1 Action Controllerの詳細

### 参考資料
- [Rails Guides - Action Controller Overview（英語）](https://guides.rubyonrails.org/action_controller_overview.html)
- [Rails Guides - Action Controller Overview（日本語）](https://railsguides.jp/action_controller_overview.html)
- [Rails API - ActionController::Base](https://api.rubyonrails.org/classes/ActionController/Base.html)

### フィルター（before_action、after_action、around_action）

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  before_action :authenticate_user!, except: [:index, :show]
  before_action :set_post, only: [:show, :edit, :update, :destroy]
  before_action :authorize_post, only: [:edit, :update, :destroy]
  after_action :log_action, only: [:create, :update, :destroy]
  around_action :set_time_zone, only: [:index]

  private

  def set_post
    @post = Post.find(params[:id])
  end

  def authorize_post
    redirect_to posts_path, alert: '権限がありません' unless @post.user == current_user
  end

  def log_action
    Rails.logger.info("Post #{action_name} by user #{current_user.id}")
  end

  def set_time_zone
    Time.use_zone(current_user.time_zone) { yield }
  end
end
```

### Strong Parametersの詳細

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    # ...
  end

  private

  def post_params
    params.require(:post).permit(:title, :content, :status, tag_ids: [])
  end

  # ネストしたパラメータ
  def post_with_comments_params
    params.require(:post).permit(
      :title, :content,
      comments_attributes: [:id, :content, :_destroy]
    )
  end
end
```

### respond_toとフォーマット

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.all

    respond_to do |format|
      format.html
      format.json { render json: @posts }
      format.xml { render xml: @posts }
      format.csv { send_data @posts.to_csv, filename: "posts-#{Date.today}.csv" }
    end
  end

  def show
    @post = Post.find(params[:id])

    respond_to do |format|
      format.html
      format.json { render json: @post }
      format.pdf { render pdf: @post }
    end
  end
end
```

### レンダリングの詳細

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)

    if @post.save
      # デフォルトのレンダリング（create.html.erb）
      redirect_to @post, notice: '投稿が作成されました'

      # または明示的に指定
      # render :show, status: :created, location: @post
    else
      # ステータスコードを指定
      render :new, status: :unprocessable_entity

      # 別のアクションのビューをレンダリング
      # render :edit

      # テンプレートを明示的に指定
      # render template: 'posts/new'
    end
  end

  def update
    @post = Post.find(params[:id])

    if @post.update(post_params)
      # JSONレスポンス
      render json: @post, status: :ok
    else
      render json: { errors: @post.errors }, status: :unprocessable_entity
    end
  end
end
```

### リダイレクトの詳細

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)

    if @post.save
      # 基本的なリダイレクト
      redirect_to @post

      # パスを指定
      redirect_to post_path(@post)

      # URLを指定
      redirect_to "https://example.com/posts/#{@post.id}"

      # ステータスコードを指定
      redirect_to @post, status: :see_other

      # オプションを指定
      redirect_to @post, notice: '投稿が作成されました', alert: '成功しました'

      # 戻る
      redirect_back(fallback_location: root_path)
    end
  end
end
```

### セッションとクッキーの管理

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :set_user_preferences

  private

  def set_user_preferences
    session[:locale] ||= 'ja'
    session[:theme] ||= 'light'
  end

  def remember_user(user)
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  def forget_user
    cookies.delete(:user_id)
    cookies.delete(:remember_token)
  end
end
```

### ストリーミングレスポンス

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def stream
    response.headers['Content-Type'] = 'text/event-stream'

    self.response_body = Enumerator.new do |yielder|
      100.times do |i|
        yielder << "data: #{i}\n\n"
        sleep 1
      end
    end
  end
end
```

## 3.2 Action View Overview（テンプレ/partials/layoutsの基礎）

### 参考資料
- [Rails Guides - Action View Overview（英語）](https://guides.rubyonrails.org/action_view_overview.html)
- [Rails Guides - Action View Overview（日本語）](https://railsguides.jp/action_view_overview.html)
- [Rails Guides - Layouts and Rendering（英語）](https://guides.rubyonrails.org/layouts_and_rendering.html)
- [Rails Guides - Layouts and Rendering（日本語）](https://railsguides.jp/layouts_and_rendering.html)
- [Rails Guides - Action View Helpers（英語）](https://guides.rubyonrails.org/action_view_helpers.html)
- [Rails Guides - Action View Helpers（日本語）](https://railsguides.jp/action_view_helpers.html)
- [Rails API - ActionView::Helpers](https://api.rubyonrails.org/classes/ActionView/Helpers.html)

### テンプレートの基礎

Railsは、コントローラーのアクション名に対応するテンプレートを自動的に探します。

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def index
    @posts = Post.all
    # app/views/posts/index.html.erb を自動的にレンダリング
  end
end
```

### パーシャル（部分テンプレート）の基礎

### ヘルパーメソッドの作成

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def post_status_badge(post)
    status_class = case post.status
                   when 'published'
                     'badge-success'
                   when 'draft'
                     'badge-secondary'
                   when 'archived'
                     'badge-warning'
                   else
                     'badge-default'
                   end

    content_tag(:span, post.status, class: "badge #{status_class}")
  end

  def post_author_link(post)
    link_to post.user.name, user_path(post.user), class: 'author-link'
  end

  def post_meta(post)
    "#{post_author_link(post)} - #{time_ago_in_words(post.created_at)}前"
  end
end
```

### パーシャル（部分テンプレート）の活用

```erb
<!-- app/views/posts/_post.html.erb -->
<article class="post" id="post_<%= post.id %>">
  <h2><%= link_to post.title, post %></h2>
  <div class="content">
    <%= truncate(post.content, length: 200) %>
  </div>
  <div class="meta">
    <%= render 'posts/meta', post: post %>
  </div>
</article>
```

```erb
<!-- app/views/posts/_meta.html.erb -->
<div class="post-meta">
  <span class="author"><%= post.user.name %></span>
  <span class="date"><%= post.created_at.strftime('%Y年%m月%d日') %></span>
  <span class="comments-count"><%= post.comments.count %>件のコメント</span>
</div>
```

```erb
<!-- app/views/posts/index.html.erb -->
<h1>投稿一覧</h1>

<div class="posts">
  <%= render @posts %>
  <!-- または明示的に指定 -->
  <!-- <%= render partial: 'posts/post', collection: @posts %> -->
</div>
```

## 3.3 Layouts and Rendering（render/redirect、ネストlayout等の定番）

### レイアウトの基礎

```erb
<!-- app/views/layouts/application.html.erb -->
<!DOCTYPE html>
<html>
  <head>
    <title><%= content_for?(:title) ? yield(:title) : 'Sample App' %></title>
    <%= csrf_meta_tags %>
    <%= csp_meta_tag %>
    <%= stylesheet_link_tag "application", "data-turbo-track": "reload" %>
    <%= javascript_importmap_tags %>
  </head>

  <body>
    <%= render 'shared/header' %>
    <%= render 'shared/flash_messages' %>

    <main>
      <%= yield %>
    </main>

    <%= render 'shared/footer' %>
    <%= yield :javascript %>
  </body>
</html>
```

### レイアウトの選択

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  layout 'admin', only: [:new, :edit]  # 特定のアクションのみ
  layout false  # レイアウトなし
end
```

### ネストしたレイアウト

```erb
<!-- app/views/layouts/admin.html.erb -->
<%= render template: 'layouts/application' do %>
  <div class="admin-sidebar">
    <%= render 'admin/sidebar' %>
  </div>
  <div class="admin-content">
    <%= yield %>
  </div>
<% end %>
```

### renderとredirectの詳細

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    
    if @post.save
      # リダイレクト
      redirect_to @post, notice: '投稿が作成されました。'
      # または
      redirect_to posts_path, status: :see_other
    else
      # レンダリング（ステータスコードを指定）
      render :new, status: :unprocessable_entity
    end
  end
  
  def show
    @post = Post.find(params[:id])
    # デフォルトで app/views/posts/show.html.erb をレンダリング
    
    # 明示的にレンダリング
    render :show
    # または別のテンプレート
    render 'posts/detail'
    # または別のコントローラーのテンプレート
    render 'shared/error', status: :not_found
  end
  
  def update
    @post = Post.find(params[:id])
    
    if @post.update(post_params)
      # JSONレスポンス
      render json: @post, status: :ok
    else
      render json: { errors: @post.errors }, status: :unprocessable_entity
    end
  end
end
```

### 条件付きレンダリング

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    
    respond_to do |format|
      format.html
      format.json { render json: @post }
      format.xml { render xml: @post }
    end
  end
end
```

### content_forの活用

```erb
<!-- app/views/posts/show.html.erb -->
<% content_for :title do %>
  投稿詳細 - <%= @post.title %>
<% end %>

<% content_for :javascript do %>
  <script>
    // ページ固有のJavaScript
  </script>
<% end %>

<article>
  <h1><%= @post.title %></h1>
  <div><%= @post.content %></div>
</article>
```

## 3.4 Action View Helpers（sanitize/capture/cache/debugなど"地味に重要"）

### sanitizeヘルパー

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def sanitize_content(content)
    # HTMLタグをサニタイズ
    sanitize(content, tags: %w[p br strong em ul ol li a], attributes: %w[href title])
  end
  
  def sanitize_strict(content)
    # すべてのHTMLタグを削除
    sanitize(content, tags: [], attributes: [])
  end
end
```

### captureヘルパー

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def post_card(post)
    capture do
      concat content_tag(:h2, post.title)
      concat content_tag(:p, post.content)
      concat render('posts/meta', post: post)
    end
  end
end
```

### cacheヘルパー

```erb
<!-- app/views/posts/index.html.erb -->
<% cache @posts do %>
  <% @posts.each do |post| %>
    <% cache post do %>
      <%= render post %>
    <% end %>
  <% end %>
<% end %>

<!-- 条件付きキャッシュ -->
<% cache_if user_signed_in?, [post, 'authenticated'] do %>
  <%= render post %>
<% end %>

<!-- キャッシュの無効化 -->
<% cache_unless post.published?, post do %>
  <%= render post %>
<% end %>
```

### debugヘルパー

```erb
<!-- app/views/posts/show.html.erb -->
<%= debug @post %>
<%= debug params %>
<%= debug @post.errors if @post.errors.any? %>
```

### その他の重要なヘルパー

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  # link_to
  def post_link(post)
    link_to post.title, post_path(post), class: 'post-link', data: { id: post.id }
  end
  
  # image_tag
  def post_image(post)
    image_tag post.image_url, alt: post.title, class: 'post-image' if post.image_url
  end
  
  # time_ago_in_words
  def post_time(post)
    "#{time_ago_in_words(post.created_at)}前"
  end
  
  # number_to_currency
  def format_price(price)
    number_to_currency(price, unit: '¥', precision: 0)
  end
  
  # truncate
  def post_excerpt(post, length: 100)
    truncate(post.content, length: length, omission: '...')
  end
  
  # simple_format
  def format_text(text)
    simple_format(text)
  end
  
  # pluralize
  def comments_count(post)
    pluralize(post.comments.count, 'comment')
  end
end
```

### フォームヘルパーの詳細

```erb
<!-- app/views/posts/new.html.erb -->
<%= form_with model: @post, local: true, html: { class: 'post-form' } do |form| %>
  <div class="field">
    <%= form.label :title, "タイトル" %>
    <%= form.text_field :title, class: 'form-control', placeholder: 'タイトルを入力' %>
    <%= form.error_messages_for :title %>
  </div>

  <div class="field">
    <%= form.label :content, "内容" %>
    <%= form.text_area :content, rows: 10, class: 'form-control' %>
  </div>

  <div class="field">
    <%= form.label :status, "ステータス" %>
    <%= form.select :status, options_for_select([
      ['下書き', 'draft'],
      ['公開', 'published'],
      ['アーカイブ', 'archived']
    ], @post.status), {}, { class: 'form-control' } %>
  </div>

  <div class="field">
    <%= form.label :tag_ids, "タグ" %>
    <%= form.collection_check_boxes :tag_ids, Tag.all, :id, :name do |b| %>
      <div class="checkbox">
        <%= b.check_box %>
        <%= b.label %>
      </div>
    <% end %>
  </div>

  <div class="actions">
    <%= form.submit "投稿する", class: 'btn btn-primary' %>
    <%= form.submit "下書き保存", name: 'draft', class: 'btn btn-secondary' %>
  </div>
<% end %>
```

### カスタムヘルパーの作成

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def page_title
    if content_for?(:title)
      "#{content_for(:title)} - Sample App"
    else
      "Sample App"
    end
  end

  def flash_messages
    flash.map do |type, message|
      content_tag(:div, message, class: "alert alert-#{type}")
    end.join.html_safe
  end

  def markdown(text)
    Redcarpet::Markdown.new(Redcarpet::Render::HTML).render(text).html_safe
  end
end
```

## 3.5 ViewComponent（コンポーネント指向のビュー設計）

### 参考資料
- [ViewComponent（公式）](https://viewcomponent.org/)

### ViewComponentとは

ViewComponentは、コンポーネント指向のビュー設計を可能にするgemです。パーシャルよりも強力で、テストしやすいコンポーネントを作成できます。

### ViewComponentのセットアップ

```ruby
# Gemfile
gem 'view_component'
```

```bash
bundle install
```

### コンポーネントの作成

```bash
rails generate component PostCard post
```

```ruby
# app/components/post_card_component.rb
class PostCardComponent < ViewComponent::Base
  def initialize(post:, show_author: true)
    @post = post
    @show_author = show_author
  end

  private

  attr_reader :post, :show_author
end
```

```erb
<!-- app/components/post_card_component.html.erb -->
<article class="post-card">
  <h2><%= link_to post.title, post_path(post) %></h2>
  <div class="content">
    <%= truncate(post.content, length: 200) %>
  </div>
  <% if show_author %>
    <div class="author">
      <%= post.user.name %>
    </div>
  <% end %>
</article>
```

### コンポーネントの使用

```erb
<!-- app/views/posts/index.html.erb -->
<% @posts.each do |post| %>
  <%= render PostCardComponent.new(post: post, show_author: true) %>
<% end %>

<!-- または短縮形 -->
<%= render(PostCardComponent.with_collection(@posts, show_author: true)) %>
```

### コンポーネントのテスト

```ruby
# spec/components/post_card_component_spec.rb
require 'rails_helper'

RSpec.describe PostCardComponent, type: :component do
  it 'renders post title' do
    post = create(:post, title: 'Test Post')
    component = PostCardComponent.new(post: post)
    
    render_inline(component)
    
    expect(page).to have_text('Test Post')
  end
  
  it 'hides author when show_author is false' do
    post = create(:post)
    component = PostCardComponent.new(post: post, show_author: false)
    
    render_inline(component)
    
    expect(page).not_to have_css('.author')
  end
end
```

### スロットの使用

```ruby
# app/components/post_card_component.rb
class PostCardComponent < ViewComponent::Base
  def initialize(post:)
    @post = post
  end

  private

  attr_reader :post
end
```

```erb
<!-- app/components/post_card_component.html.erb -->
<article class="post-card">
  <h2><%= post.title %></h2>
  <div class="content">
    <%= content %>
  </div>
  <div class="footer">
    <%= footer %>
  </div>
</article>
```

```erb
<!-- 使用例 -->
<%= render PostCardComponent.new(post: post) do |component| %>
  <% component.with_content do %>
    <%= post.content %>
  <% end %>
  <% component.with_footer do %>
    <%= post.user.name %>
  <% end %>
<% end %>
```

## 3.6 API-only Applications（API専用構成の取捨選択）

### 参考資料
- [Rails Guides - API-only Applications（英語）](https://guides.rubyonrails.org/api_app.html)
- [Rails Guides - API-only Applications（日本語）](https://railsguides.jp/api_app.html)

### API-onlyアプリケーションの作成

```bash
rails new api_app --api
```

### API-onlyアプリケーションの特徴

```ruby
# config/application.rb
module ApiApp
  class Application < Rails::Application
    config.load_defaults 7.0
    config.api_only = true  # API専用モード
  end
end
```

### API-onlyモードの影響

1. **ActionController::API**: ActionController::Baseの代わりに使用
2. **ビューレイヤーなし**: ビュー、レイアウト、ヘルパーが不要
3. **ミドルウェアの削減**: セッション、クッキー関連のミドルウェアが削除
4. **CSRF保護なし**: APIでは不要（代わりに認証トークンを使用）

### APIコントローラーの作成

```ruby
# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < Api::BaseController
      before_action :authenticate_api_user!
      
      def index
        @posts = Post.all
        render json: @posts
      end
      
      def show
        @post = Post.find(params[:id])
        render json: @post
      end
      
      def create
        @post = current_user.posts.build(post_params)
        
        if @post.save
          render json: @post, status: :created
        else
          render json: { errors: @post.errors }, status: :unprocessable_entity
        end
      end
      
      private
      
      def post_params
        params.require(:post).permit(:title, :content)
      end
    end
  end
end
```

### API専用の設定

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

### 通常のRailsアプリケーションからAPIを分離する場合

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # Webアプリケーション用
  resources :posts
  
  # API用
  namespace :api do
    namespace :v1 do
      resources :posts
    end
  end
end
```

## 3.7 ビューの最適化

### フラグメントキャッシュ

```erb
<!-- app/views/posts/index.html.erb -->
<% cache @posts do %>
  <% @posts.each do |post| %>
    <% cache post do %>
      <%= render post %>
    <% end %>
  <% end %>
<% end %>
```

### 条件付きキャッシュ

```erb
<% cache_if user_signed_in?, [post, 'authenticated'] do %>
  <%= render post %>
<% end %>
```

### コレクションキャッシュ

```erb
<%= render partial: 'posts/post', collection: @posts, cached: true %>
```

## 3.8 確認ポイント

この章の最後に、以下を確認してください：

1. **フィルターが正しく動作している**
   - before_action、after_actionが実行されている

2. **Strong Parametersが機能している**
   - 不正なパラメータが拒否されている

3. **ビューヘルパーが動作している**
   - カスタムヘルパーが正しく表示されている

4. **パーシャルが正しく使用されている**
   - コードの重複が減っている

## 3.9 参考資料

- [Rails Guides - Action Controller Overview（英語）](https://guides.rubyonrails.org/action_controller_overview.html)
- [Rails Guides - Action View Overview（英語）](https://guides.rubyonrails.org/action_view_overview.html)
- [Rails Guides - Layouts and Rendering（英語）](https://guides.rubyonrails.org/layouts_and_rendering.html)
- [Rails Guides - Action View Helpers（英語）](https://guides.rubyonrails.org/action_view_helpers.html)
- [Rails Guides - API-only Applications（英語）](https://guides.rubyonrails.org/api_app.html)
- [ViewComponent（公式）](https://viewcomponent.org/)
- [Rails API - ActionController::Base](https://api.rubyonrails.org/classes/ActionController/Base.html)
- [Rails API - ActionView::Helpers](https://api.rubyonrails.org/classes/ActionView/Helpers.html)

## 3.10 次の章へ

この章で、Action ControllerとAction Viewの詳細を学びました。次の章では、ルーティングの高度な使い方を学びます。

**次の章**: [第5章: ルーティングの高度な使い方](chapter05-routing.md)

## まとめ

この章では以下を学びました：

- **Action Controller**: フィルター、Strong Parameters、respond_to、レンダリング、リダイレクト
- **Action View Overview**: テンプレート、パーシャル、レイアウトの基礎
- **Layouts and Rendering**: render/redirect、ネストlayout等の定番
- **Action View Helpers**: sanitize/capture/cache/debugなど"地味に重要"
- **ViewComponent**: コンポーネント指向のビュー設計
- **API-only Applications**: API専用構成の取捨選択
- **ビューの最適化**: フラグメントキャッシュ、コレクションキャッシュ
