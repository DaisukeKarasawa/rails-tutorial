# 第8章: セキュリティの深掘りとベストプラクティス

## はじめに

この章では、[Rails Guides - Securing Rails Applications](https://guides.rubyonrails.org/security.html)を参考に、既存のアプリケーションのセキュリティを強化します。CSRF/XSS/SQLインジェクション対策、認可・セッション管理、Strong Parametersの詳細などを実装しながら、実践的なセキュリティ対策を学びます。

## 8.0 Securing Rails Applications（公式ガイドの基本線）

### 参考資料
- [Rails Guides - Securing Rails Applications（英語）](https://guides.rubyonrails.org/security.html)
- [Rails Guides - Securing Rails Applications（日本語）](https://railsguides.jp/security.html)

### セキュリティの基本原則

1. **最小権限の原則**: 必要最小限の権限のみを付与
2. **防御的プログラミング**: すべての入力を検証
3. **セキュリティをデフォルトで有効化**: 明示的に無効化しない限り有効
4. **多層防御**: 複数の防御層を実装

## 8.1 CSRF（Cross-Site Request Forgery）対策

### CSRFとは

CSRFは、ユーザーが意図しないリクエストを実行させられる攻撃です。

### RailsのCSRF対策

RailsはデフォルトでCSRF対策が有効です。

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  # または
  protect_from_forgery with: :null_session
end
```

### CSRFトークンの確認

```erb
<!-- app/views/layouts/application.html.erb -->
<%= csrf_meta_tags %>
```

### APIでのCSRF対策

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  # APIではCSRFトークンは不要（代わりに認証トークンを使用）
  # ActionController::APIはCSRF保護がデフォルトで無効
end
```

### カスタムCSRF対策

```ruby
# config/application.rb
config.action_controller.default_protect_from_forgery = true

# 特定のアクションのみCSRF保護をスキップ（非推奨）
skip_before_action :verify_authenticity_token, only: [:webhook]
```

## 8.2 XSS（Cross-Site Scripting）対策

### XSSとは

XSSは、悪意のあるスクリプトをWebページに注入する攻撃です。

### Railsの自動エスケープ

RailsはデフォルトでHTMLを自動的にエスケープします。

```erb
<!-- 自動的にエスケープされる -->
<%= @user.name %>
<!-- => &lt;script&gt;alert('XSS')&lt;/script&gt; -->

<!-- エスケープをスキップ（危険） -->
<%= raw @user.bio %>
<%= @user.bio.html_safe %>
```

### Content Security Policy（CSP）

```ruby
# config/initializers/content_security_policy.rb
Rails.application.config.content_security_policy do |policy|
  policy.default_src :self, :https
  policy.font_src    :self, :https, :data
  policy.img_src     :self, :https, :data
  policy.object_src  :none
  policy.script_src  :self, :https
  policy.style_src   :self, :https
  policy.base_uri    :self
  policy.form_action :self
  policy.frame_ancestors :none
  
  # インラインスクリプトを許可する場合（非推奨）
  # policy.script_src :self, :unsafe_inline
end

# CSP違反のレポート
Rails.application.config.content_security_policy_report_only = false
Rails.application.config.content_security_policy_nonce_generator = ->(request) { SecureRandom.base64(16) }
```

### sanitizeの使用

```ruby
# Gemfile
gem 'rails-html-sanitizer'
```

```ruby
# app/helpers/posts_helper.rb
module PostsHelper
  def sanitize_content(content)
    # 許可するタグと属性を指定
    sanitize(content, 
      tags: %w[p br strong em ul ol li a],
      attributes: %w[href title]
    )
  end
  
  def sanitize_strict(content)
    # より厳格なサニタイズ
    sanitize(content, tags: [], attributes: [])
  end
end
```

## 8.3 SQLインジェクション対策

### SQLインジェクションとは

SQLインジェクションは、悪意のあるSQLコードを注入する攻撃です。

### パラメータ化クエリの使用

```ruby
# 危険な例（SQLインジェクションの脆弱性）
User.where("email = '#{params[:email]}'")
# params[:email] = "admin@example.com' OR '1'='1"
# => SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'

# 安全な例（パラメータ化クエリ）
User.where("email = ?", params[:email])
User.where(email: params[:email])
```

### 動的クエリの安全な構築

```ruby
# 危険な例
Post.order("#{params[:sort]} #{params[:direction]}")

# 安全な例
allowed_sort = %w[title created_at updated_at].include?(params[:sort]) ? params[:sort] : 'created_at'
allowed_direction = %w[asc desc].include?(params[:direction]) ? params[:direction] : 'desc'
Post.order("#{allowed_sort} #{allowed_direction}")

# より安全な例
Post.order(Arel.sql("#{sanitize_sql_for_order("#{allowed_sort} #{allowed_direction}")}"))
```

### find_by_sqlの注意点

```ruby
# 危険な例
Post.find_by_sql("SELECT * FROM posts WHERE title = '#{params[:title]}'")

# 安全な例
Post.find_by_sql(["SELECT * FROM posts WHERE title = ?", params[:title]])
```

## 8.4 Strong Parametersの詳細

### 参考資料
- [Rails API - ActionController::Parameters](https://api.rubyonrails.org/classes/ActionController/Parameters.html)

### requireとpermitの実挙動

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    # require: 指定したキーが必須、なければActionController::ParameterMissing
    params.require(:post)
    # => { title: "...", content: "..." }
    # もし:postキーがなければ例外が発生

    # permit: 許可するパラメータのみを返す
    params.require(:post).permit(:title, :content)
    # => { title: "...", content: "..." }
    # 許可されていないパラメータは無視される
  end
end
```

### ネストしたパラメータ

```ruby
# app/controllers/posts_controller.rb
def post_params
  params.require(:post).permit(
    :title, 
    :content,
    comments_attributes: [:id, :content, :_destroy],
    tags_attributes: [:id, :name]
  )
end
```

### 配列パラメータ

```ruby
# app/controllers/posts_controller.rb
def post_params
  params.require(:post).permit(
    :title,
    :content,
    tag_ids: []  # 配列として受け取る
  )
end
```

### ハッシュパラメータ

```ruby
# app/controllers/posts_controller.rb
def post_params
  params.require(:post).permit(
    :title,
    metadata: {}  # ハッシュとして受け取る
  )
end
```

### 条件付きpermit

```ruby
# app/controllers/posts_controller.rb
def post_params
  base_params = [:title, :content]
  base_params << :admin_only_field if current_user&.admin?
  
  params.require(:post).permit(*base_params)
end
```

### Strong Parametersのテスト

```ruby
# spec/controllers/posts_controller_spec.rb
RSpec.describe PostsController do
  describe 'POST #create' do
    it 'filters out unpermitted parameters' do
      post :create, params: {
        post: {
          title: 'Test',
          content: 'Content',
          admin_only: true,  # 許可されていないパラメータ
          malicious: '<script>alert("XSS")</script>'
        }
      }
      
      expect(Post.last.title).to eq('Test')
      expect(Post.last).not_to respond_to(:admin_only)
    end
  end
end
```

## 8.5 認可機能の実装（Pundit）

### Punditのセットアップ

```ruby
# Gemfile
gem 'pundit'
```

```bash
bundle install
rails generate pundit:install
```

### PostPolicyの作成

```ruby
# app/policies/post_policy.rb
class PostPolicy < ApplicationPolicy
  def show?
    true
  end

  def create?
    user.present?
  end

  def update?
    user.present? && (record.user == user || user.admin?)
  end

  def destroy?
    user.present? && (record.user == user || user.admin?)
  end

  class Scope < Scope
    def resolve
      if user&.admin?
        scope.all
      else
        scope.where(published: true)
      end
    end
  end
end
```

### PostsControllerの更新

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  include Pundit::Authorization

  after_action :verify_authorized, except: :index
  after_action :verify_policy_scoped, only: :index

  def index
    @posts = policy_scope(Post).includes(:user).order(created_at: :desc).page(params[:page])
  end

  def show
    authorize @post
  end

  def create
    @post = current_user.posts.build(post_params)
    authorize @post
    
    if @post.save
      redirect_to @post, notice: '投稿が作成されました。'
    else
      render :new, status: :unprocessable_entity
    end
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
end
```

## 8.6 セッション管理のセキュリティ

### セッションストアの設定

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: '_sample_app_session',
  secure: Rails.env.production?,  # HTTPSのみ
  httponly: true,                 # JavaScriptからアクセス不可
  same_site: :lax,                # CSRF対策
  expire_after: 2.weeks
```

### セッションタイムアウト

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :check_session_timeout

  private

  def check_session_timeout
    if session[:last_activity_time] && session[:last_activity_time] < 30.minutes.ago
      reset_session
      redirect_to login_path, alert: 'セッションがタイムアウトしました'
    else
      session[:last_activity_time] = Time.current
    end
  end
end
```

### セッションの固定攻撃対策

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :regenerate_session_id, if: :user_signed_in?

  private

  def regenerate_session_id
    # ログイン時にセッションIDを再生成
    old_session = session.to_hash
    reset_session
    old_session.each { |k, v| session[k] = v }
  end
end
```

## 8.7 セキュアなAPI設計

### APIコントローラーの作成

```ruby
# app/controllers/api/v1/posts_controller.rb
module Api
  module V1
    class PostsController < Api::BaseController
      before_action :authenticate_api_user!
      
      def index
        @posts = policy_scope(Post).includes(:user).order(created_at: :desc)
        render json: @posts, each_serializer: Api::V1::PostSerializer
      end

      def show
        @post = Post.find(params[:id])
        authorize @post
        render json: @post, serializer: Api::V1::PostSerializer
      end

      def create
        @post = current_user.posts.build(post_params)
        authorize @post
        
        if @post.save
          render json: @post, serializer: Api::V1::PostSerializer, status: :created
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

### JWT認証の実装

```ruby
# Gemfile
gem 'jwt'
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def generate_jwt
    JWT.encode(
      { user_id: id, exp: 24.hours.from_now.to_i },
      Rails.application.credentials.secret_key_base
    )
  end

  def self.from_jwt(token)
    decoded = JWT.decode(token, Rails.application.credentials.secret_key_base)[0]
    find(decoded['user_id'])
  rescue JWT::DecodeError, ActiveRecord::RecordNotFound
    nil
  end
end
```

```ruby
# app/controllers/api/base_controller.rb
module Api
  class BaseController < ActionController::API
    include Pundit::Authorization
    
    before_action :authenticate_api_user!

    private

    def authenticate_api_user!
      token = request.headers['Authorization']&.split(' ')&.last
      @current_user = User.from_jwt(token)

      render json: { error: 'Unauthorized' }, status: :unauthorized unless @current_user
    end

    def current_user
      @current_user
    end
  end
end
```

### ルーティングの追加

```ruby
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :posts
    end
  end
  
  # ... 既存のルーティング ...
end
```

## 8.8 セキュリティヘッダーの設定

```ruby
# config/initializers/security_headers.rb
Rails.application.config.action_dispatch.default_headers.merge!(
  'X-Frame-Options' => 'DENY',
  'X-Content-Type-Options' => 'nosniff',
  'X-XSS-Protection' => '1; mode=block',
  'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains',
  'Referrer-Policy' => 'strict-origin-when-cross-origin'
)
```

## 8.9 パスワードの安全な管理

### 強力なパスワード要件

```ruby
# app/models/user.rb
class User < ApplicationRecord
  PASSWORD_REQUIREMENTS = /\A
    (?=.*\d)           # 数字を含む
    (?=.*[a-z])        # 小文字を含む
    (?=.*[A-Z])        # 大文字を含む
    (?=.*[[:^alnum:]]) # 特殊文字を含む
    .{8,}              # 8文字以上
  \z/x

  validates :password, format: { with: PASSWORD_REQUIREMENTS }, if: :password_required?
end
```

### パスワードのハッシュ化

Deviseはデフォルトでbcryptを使用します。

```ruby
# config/initializers/devise.rb
config.stretches = Rails.env.test? ? 1 : 12
config.pepper = ENV['DEVISE_PEPPER'] if ENV['DEVISE_PEPPER']
```

### パスワードリセットのセキュリティ

```ruby
# app/models/user.rb
class User < ApplicationRecord
  def generate_password_reset_token!
    self.password_reset_token = SecureRandom.urlsafe_base64
    self.password_reset_sent_at = Time.current
    save!
  end

  def password_reset_token_valid?
    password_reset_sent_at.present? && 
    password_reset_sent_at > 1.hour.ago &&
    password_reset_sent_at < 24.hours.ago
  end
end
```

## 8.10 ファイルアップロードのセキュリティ

### ファイルタイプの検証

```ruby
# app/models/document.rb
class Document < ApplicationRecord
  has_one_attached :file

  validate :acceptable_file_type
  validate :acceptable_file_size

  private

  def acceptable_file_type
    return unless file.attached?

    allowed_types = %w[application/pdf image/jpeg image/png]
    unless file.content_type.in?(allowed_types)
      errors.add(:file, '許可されていないファイルタイプです')
    end
  end

  def acceptable_file_size
    return unless file.attached?

    max_size = 5.megabytes
    if file.byte_size > max_size
      errors.add(:file, "ファイルサイズが大きすぎます（最大#{max_size}）")
    end
  end
end
```

### ファイル名の検証

```ruby
# app/models/document.rb
class Document < ApplicationRecord
  validate :safe_file_name

  private

  def safe_file_name
    return unless file.attached?

    # 危険な文字をチェック
    dangerous_chars = /[\/\\\?\*\|<>:]/
    if file.filename.to_s.match?(dangerous_chars)
      errors.add(:file, 'ファイル名に無効な文字が含まれています')
    end
  end
end
```

## 8.11 ログのセキュリティ

### 機密情報のログ除外

```ruby
# config/initializers/filter_parameter_logging.rb
Rails.application.config.filter_parameters += [
  :password,
  :password_confirmation,
  :credit_card,
  :ssn,
  :api_key,
  :secret_token
]
```

### ログのローテーション

```ruby
# config/environments/production.rb
config.logger = ActiveSupport::Logger.new(
  Rails.root.join('log', 'production.log'),
  5,  # 保持するログファイル数
  100.megabytes  # ファイルサイズの上限
)
```

## 8.12 脆弱性スキャンの実施

### Brakemanのセットアップ

```ruby
# Gemfile
group :development, :test do
  gem 'brakeman', require: false
end
```

```bash
bundle install
brakeman
```

### bundler-auditのセットアップ

```ruby
# Gemfile
group :development, :test do
  gem 'bundler-audit', require: false
end
```

```bash
bundle install
bundle audit check --update
```

## 8.13 確認ポイント

この章の最後に、以下を確認してください：

1. **CSRF対策が有効**
   - フォームにCSRFトークンが含まれている
   - APIでは認証トークンを使用している

2. **XSS対策が実装されている**
   - HTMLが自動的にエスケープされている
   - CSPが設定されている

3. **SQLインジェクション対策が実装されている**
   - パラメータ化クエリを使用している
   - 動的クエリが安全に構築されている

4. **Strong Parametersが正しく使用されている**
   - requireとpermitが適切に使用されている
   - 許可されていないパラメータが除外されている

5. **認可が正しく動作している**
   - 他のユーザーの投稿は編集・削除できない
   - 管理者はすべての投稿を編集・削除できる

6. **セッション管理が安全**
   - セッションタイムアウトが実装されている
   - セッションIDが適切に管理されている

7. **APIがセキュアに動作している**
   - 認証トークンなしではアクセスできない
   - 認可が正しく機能している

8. **脆弱性スキャンが通る**
   - Brakemanで重大な問題がない
   - bundler-auditで脆弱性がない

## 8.14 参考資料

- [Rails Guides - Securing Rails Applications（英語）](https://guides.rubyonrails.org/security.html)
- [Rails Guides - Securing Rails Applications（日本語）](https://railsguides.jp/security.html)
- [Rails API - ActionController::Parameters](https://api.rubyonrails.org/classes/ActionController/Parameters.html)

## 8.15 次の章へ

この章で、アプリケーションのセキュリティを強化しました。次の章では、高度なテスト戦略を学びます。

**次の章**: [第9章: 高度なテスト戦略と品質保証](chapter09-advanced-testing.md)

## まとめ

この章では以下を学びました：

- **CSRF対策**: protect_from_forgery、CSRFトークン
- **XSS対策**: 自動エスケープ、CSP、sanitize
- **SQLインジェクション対策**: パラメータ化クエリ、動的クエリの安全な構築
- **Strong Parameters**: require/permitの実挙動、ネストしたパラメータ
- **認可機能**: Punditによる実装
- **セッション管理**: セキュアな設定、タイムアウト、セッション固定攻撃対策
- **パスワード管理**: 強力な要件、ハッシュ化
- **ファイルアップロード**: タイプ検証、サイズ制限、ファイル名検証
- **ログのセキュリティ**: 機密情報の除外、ログローテーション
- **セキュアなAPI**: JWT認証、認可
- **セキュリティヘッダー**: 各種ヘッダーの設定
- **脆弱性スキャン**: Brakeman、bundler-audit
