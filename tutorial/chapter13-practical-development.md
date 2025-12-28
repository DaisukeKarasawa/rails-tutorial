# 第7章: 実践的な開発手法とワークフロー

## はじめに

この章では、実践的な開発手法とワークフローを学びます。コードレビュー、リファクタリング、API設計、マイクロサービスへの移行準備など、実際の開発現場で役立つ手法を扱います。

## 7.1 コードレビューの実践

### レビューチェックリスト

#### セキュリティ
- [ ] SQLインジェクションの脆弱性がないか
- [ ] XSS対策が適切か
- [ ] 認証・認可が適切に実装されているか

#### パフォーマンス
- [ ] N+1問題がないか
- [ ] 適切なインデックスが設定されているか
- [ ] キャッシュが適切に使用されているか

#### コード品質
- [ ] DRY原則に従っているか
- [ ] 単一責任の原則に従っているか
- [ ] 適切な命名がされているか

### 自動レビューツール

```yaml
# .github/workflows/code_review.yml
name: Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run RuboCop
      run: |
        bundle exec rubocop
    
    - name: Run Brakeman
      run: |
        bundle exec brakeman --no-pager
```

## 7.2 リファクタリングの実践

### リファクタリングの原則

1. **テストを書く**（既存の動作を保証）
2. **小さな変更を加える**
3. **テストを実行して確認**
4. **次の変更に進む**

### 長いメソッドの分割

```ruby
# リファクタリング前
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    
    if @post.title.present? && @post.content.present?
      if @post.save
        PostCreatedNotificationJob.perform_later(@post.id)
        redirect_to @post, notice: '投稿が作成されました。'
      else
        render :new, status: :unprocessable_entity
      end
    else
      @post.errors.add(:base, 'タイトルと内容は必須です')
      render :new, status: :unprocessable_entity
    end
  end
end

# リファクタリング後
class PostsController < ApplicationController
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
end
```

## 7.3 API設計とバージョニング

### APIバージョニング

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :posts
    end
    
    namespace :v2 do
      resources :posts do
        resources :comments
      end
    end
  end
end
```

### APIシリアライザー

```ruby
# app/serializers/api/v1/post_serializer.rb
class Api::V1::PostSerializer < ActiveModel::Serializer
  attributes :id, :title, :content, :created_at, :updated_at
  
  belongs_to :user, serializer: Api::V1::UserSerializer
end
```

## 7.4 マイクロサービスへの移行準備

### サービス分離の検討

現在のアプリケーションを、将来的にマイクロサービスに分割することを想定します。

#### 候補となるサービス
- **User Service**: ユーザー管理
- **Post Service**: 投稿管理
- **Notification Service**: 通知管理

### イベント駆動アーキテクチャの準備

```ruby
# app/services/event_publisher.rb
class EventPublisher
  def self.publish(event_type, payload)
    # 将来的にメッセージキューを使用
    Rails.logger.info("Event: #{event_type}, Payload: #{payload}")
  end
end
```

```ruby
# app/services/post_creation_service.rb
class PostCreationService
  def create_post
    post = user.posts.create!(post_params)
    EventPublisher.publish('post.created', { post_id: post.id, user_id: user.id })
    post
  end
end
```

## 7.5 開発ワークフロー

### Git Flow

```bash
# 機能ブランチの作成
git checkout -b feature/add-comments develop

# 開発とコミット
git add .
git commit -m "feat: add comments feature"

# developブランチにマージ
git checkout develop
git merge feature/add-comments
```

### コミットメッセージの規約

```
<type>(<scope>): <subject>

<body>

<footer>
```

例：
```
feat(posts): add comment functionality

- Implement comment model
- Add comment controller
- Update post views

Closes #123
```

## 7.6 確認ポイント

この章の最後に、以下を確認してください：

1. **コードレビューが実施されている**
   - プルリクエストでレビューが行われている

2. **リファクタリングが完了している**
   - コードが整理されている
   - テストが通る

3. **APIが設計されている**
   - バージョニングが適切
   - シリアライザーが実装されている

## 7.7 まとめ

この章では以下を学びました：

- **コードレビュー**: チェックリスト、自動レビュー
- **リファクタリング**: 実践的な手法
- **API設計**: バージョニング、シリアライザー
- **マイクロサービス**: 移行準備
- **開発ワークフロー**: Git Flow、コミット規約

## おめでとうございます！

このチュートリアルを完了しました。以下のスキルを身につけました：

- 基本的なRailsアプリケーションの構築
- 高度な設計パターンの実装
- パフォーマンス最適化
- セキュリティ対策
- テスト戦略
- 本番環境へのデプロイ
- 実践的な開発手法

これで、実践的なRailsアプリケーション開発のスキルが身につきました！
