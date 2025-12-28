# 第5章: ルーティングの高度な使い方

## はじめに

この章では、[Rails Guides - Routing from the Outside In](https://guides.rubyonrails.org/routing.html)を参考に、Railsのルーティングの高度な機能と設計感を学びます。制約、ネスト、concernsなどの設計パターンを理解し、保守性の高いルーティングを設計します。

## 5.1 Rails Routing from the Outside In（設計感）

### 参考資料
- [Rails Guides - Routing from the Outside In（英語）](https://guides.rubyonrails.org/routing.html)
- [Rails Guides - Routing from the Outside In（日本語）](https://railsguides.jp/routing.html)
- [Rails API - ActionDispatch::Routing](https://api.rubyonrails.org/classes/ActionDispatch/Routing.html)

### ルーティング設計の原則

1. **RESTfulな設計**: リソースを中心に設計
2. **浅いネスト**: ネストは1レベルまで
3. **Concernsの活用**: 共通のルーティングパターンを再利用
4. **制約の適切な使用**: パスとリクエストの制約でセキュリティと柔軟性を確保

### ルーティングの基礎

### 基本的なルーティング

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # リソースフルルーティング
  resources :posts

  # 単数リソース
  resource :profile

  # ネストしたリソース
  resources :posts do
    resources :comments
  end
end
```

### ルーティングの確認

```bash
# すべてのルーティングを確認
rails routes

# 特定のコントローラーのルーティングを確認
rails routes -c posts

# 特定のパスのルーティングを確認
rails routes -g posts
```

## 5.2 リソースルーティングの詳細

### アクションの制限

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # 特定のアクションのみ
  resources :posts, only: [:index, :show, :create]

  # 特定のアクションを除外
  resources :posts, except: [:destroy]

  # すべてのアクション（デフォルト）
  resources :posts
end
```

### カスタムアクションの追加

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts do
    # メンバールート（単一リソースに対するアクション）
    member do
      get :publish
      patch :archive
    end

    # コレクションルート（コレクションに対するアクション）
    collection do
      get :search
      get :popular
    end
  end
end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def publish
    @post = Post.find(params[:id])
    @post.update(status: 'published')
    redirect_to @post
  end

  def archive
    @post = Post.find(params[:id])
    @post.update(status: 'archived')
    redirect_to posts_path
  end

  def search
    @posts = Post.search(params[:q])
  end

  def popular
    @posts = Post.popular
  end
end
```

### パスとURLヘルパーのカスタマイズ

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resources :posts, path: 'articles' do
    # /articles/:id になる
  end

  resources :posts, param: :slug do
    # /posts/:slug になる（params[:slug]で取得）
  end
end
```

## 5.3 名前空間とスコープ

### 名前空間

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :admin do
    resources :posts
    resources :users
  end
end
```

```ruby
# app/controllers/admin/posts_controller.rb
module Admin
  class PostsController < ApplicationController
    # /admin/posts にアクセス
  end
end
```

### スコープ

```ruby
# config/routes.rb
Rails.application.routes.draw do
  scope :admin do
    resources :posts
    # パスは /admin/posts だが、コントローラーは PostsController
  end

  scope module: 'admin' do
    resources :posts
    # パスは /posts だが、コントローラーは Admin::PostsController
  end

  scope '/admin', module: 'admin', as: 'admin' do
    resources :posts
    # パスは /admin/posts、コントローラーは Admin::PostsController、ヘルパーは admin_posts_path
  end
end
```

## 5.4 制約（Constraints）の設計感

### パス制約

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # IDが数値のみ
  get 'posts/:id', to: 'posts#show', constraints: { id: /\d+/ }

  # カスタム制約クラス
  class SlugConstraint
    def matches?(request)
      Post.exists?(slug: request.params[:slug])
    end
  end

  get 'posts/:slug', to: 'posts#show', constraints: SlugConstraint.new
end
```

### リクエスト制約

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # サブドメイン制約
  constraints subdomain: 'api' do
    namespace :api do
      resources :posts
    end
  end

  # ホスト制約
  constraints host: 'admin.example.com' do
    namespace :admin do
      resources :posts
    end
  end
end
```

## 5.5 ルーティングの整理（Concerns）の設計感

```ruby
# config/routes.rb
Rails.application.routes.draw do
  concern :commentable do
    resources :comments, only: [:create, :destroy]
  end

  concern :likeable do
    resources :likes, only: [:create, :destroy]
  end

  resources :posts, concerns: [:commentable, :likeable]
  resources :articles, concerns: [:commentable, :likeable]
end
```

## 5.6 ネストしたリソースの設計

### 浅いネストの原則

```ruby
# 良い例: 浅いネスト（1レベルまで）
resources :posts do
  resources :comments, only: [:index, :create]
end

# 悪い例: 深いネスト（避ける）
resources :posts do
  resources :comments do
    resources :likes  # 深すぎる
  end
end

# 改善例: 浅いネストに変更
resources :posts do
  resources :comments, only: [:index, :create]
end

resources :comments do
  resources :likes, only: [:create, :destroy]
end
```

### ネストの代替案

```ruby
# ネストを使わずに、パラメータで関連を表現
resources :posts
resources :comments, param: :post_id  # post_idで関連を表現
```

## 5.7 ルートの直接定義

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # GETリクエスト
  get 'about', to: 'pages#about'
  get 'contact', to: 'pages#contact'

  # POSTリクエスト
  post 'posts/:id/publish', to: 'posts#publish'

  # PUT/PATCHリクエスト
  patch 'posts/:id/archive', to: 'posts#archive'

  # DELETEリクエスト
  delete 'posts/:id', to: 'posts#destroy'

  # 複数のHTTPメソッド
  match 'posts/:id', to: 'posts#show', via: [:get, :post]

  # すべてのHTTPメソッド
  match 'posts/:id', to: 'posts#show', via: :all
end
```

## 5.7 ルーティングのテスト

```ruby
# spec/routing/posts_routing_spec.rb
require 'rails_helper'

RSpec.describe "Posts routing" do
  it "routes to #index" do
    expect(get: "/posts").to route_to("posts#index")
  end

  it "routes to #show" do
    expect(get: "/posts/1").to route_to("posts#show", id: "1")
  end

  it "routes to #create" do
    expect(post: "/posts").to route_to("posts#create")
  end

  it "routes to #publish" do
    expect(get: "/posts/1/publish").to route_to("posts#publish", id: "1")
  end
end
```

## 5.8 実践的な例：APIルーティング

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :posts, only: [:index, :show, :create, :update, :destroy] do
        resources :comments, only: [:index, :create]
        member do
          post :like
          delete :unlike
        end
      end
    end

    namespace :v2 do
      resources :posts do
        resources :comments
      end
    end
  end
end
```

## 5.9 確認ポイント

この章の最後に、以下を確認してください：

1. **ルーティングが正しく設定されている**
   ```bash
   rails routes
   ```

2. **カスタムアクションが動作している**
   - メンバールート、コレクションルートが動作する

3. **名前空間が正しく機能している**
   - コントローラーが正しい場所にある

## 5.10 参考資料

- [Rails Guides - Routing（英語）](https://guides.rubyonrails.org/routing.html)
- [Rails Guides - Routing（日本語）](https://railsguides.jp/routing.html)
- [Rails API - ActionDispatch::Routing](https://api.rubyonrails.org/classes/ActionDispatch/Routing.html)

## 5.11 次の章へ

この章で、ルーティングの高度な使い方と設計感を学びました。次の章では、高度なRailsアーキテクチャと設計パターンを学びます。

**次の章**: [第6章: 高度なRailsアーキテクチャと設計パターン](chapter06-advanced-architecture.md)

## まとめ

この章では以下を学びました：

- **Rails Routing from the Outside In**: 設計感と原則
- **リソースルーティング**: アクションの制限、カスタムアクション
- **名前空間とスコープ**: ルーティングの整理
- **制約**: パス制約、リクエスト制約の設計感
- **Concerns**: ルーティングの再利用と設計感
- **ネストしたリソース**: 浅いネストの原則
- **APIルーティング**: バージョニング
