# 第9章: 高度なテスト戦略と品質保証

## はじめに

この章では、[Rails Guides - Testing Rails Applications](https://guides.rubyonrails.org/testing.html)を参考に、既存のアプリケーションに対して、高度なテスト戦略を実装します。Rails標準のテスト、RSpecのセットアップ、テストダブルの活用、統合テスト、並列テスト、CI/CDパイプラインの構築などを実装しながら、実践的なテスト手法を学びます。

## 9.1 Testing Rails Applications（Rails標準の全体像）

### 参考資料
- [Rails Guides - Testing Rails Applications（英語）](https://guides.rubyonrails.org/testing.html)
- [Rails Guides - Testing Rails Applications（日本語）](https://railsguides.jp/testing.html)

### Rails標準のテスト構造

```ruby
# test/test_helper.rb
ENV['RAILS_ENV'] ||= 'test'
require_relative '../config/environment'
require 'rails/test_help'

class ActiveSupport::TestCase
  # 並列テストの設定
  parallelize(workers: :number_of_processors)
  
  # フィクスチャの設定
  fixtures :all
end
```

### モデルテスト

```ruby
# test/models/post_test.rb
require 'test_helper'

class PostTest < ActiveSupport::TestCase
  test "should not save post without title" do
    post = Post.new
    assert_not post.save, "Saved the post without a title"
  end

  test "should save post with valid attributes" do
    post = Post.new(title: "Test", content: "Content", user: users(:one))
    assert post.save
  end
end
```

### コントローラーテスト

```ruby
# test/controllers/posts_controller_test.rb
require 'test_helper'

class PostsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @post = posts(:one)
    @user = users(:one)
  end

  test "should get index" do
    get posts_url
    assert_response :success
  end

  test "should create post" do
    assert_difference('Post.count') do
      post posts_url, params: { post: { title: @post.title, content: @post.content } }
    end

    assert_redirected_to post_url(Post.last)
  end
end
```

### System Test（システムテスト）

```ruby
# test/system/posts_test.rb
require 'application_system_test_case'

class PostsTest < ApplicationSystemTestCase
  setup do
    @post = posts(:one)
    @user = users(:one)
  end

  test "visiting the index" do
    visit posts_url
    assert_selector "h1", text: "Posts"
  end

  test "creating a Post" do
    visit posts_url
    click_on "New Post"

    fill_in "Title", with: @post.title
    fill_in "Content", with: @post.content
    click_on "Create Post"

    assert_text "Post was successfully created"
    click_on "Back"
  end
end
```

### Fixtures（フィクスチャ）

```yaml
# test/fixtures/users.yml
one:
  name: MyString
  email: user1@example.com
  password_digest: <%= BCrypt::Password.create('password') %>

two:
  name: MyString
  email: user2@example.com
  password_digest: <%= BCrypt::Password.create('password') %>
```

```yaml
# test/fixtures/posts.yml
one:
  title: MyString
  content: MyText
  user: one

two:
  title: MyString
  content: MyText
  user: two
```

### 並列テストの設定

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  # 並列テストを有効化
  parallelize(workers: :number_of_processors)
  
  # データベースの設定
  parallelize_setup do |worker|
    # 各ワーカー用のデータベースを設定
    ActiveRecord::Base.establish_connection
  end
  
  parallelize_teardown do |worker|
    # クリーンアップ
  end
end
```

### テストの実行

```bash
# すべてのテストを実行
rails test

# 特定のテストファイルを実行
rails test test/models/post_test.rb

# 特定のテストメソッドを実行
rails test test/models/post_test.rb:10

# 並列テストを実行
rails test --parallel

# システムテストを実行
rails test:system
```

## 9.2 並列テスト（Parallel Testing）の詳細と落とし穴

### 参考資料
- [Parallel Testing in Rails 7: Benefits and Pitfalls](https://dev.to/philsmy/parallel-testing-in-rails-7-benefits-and-pitfalls-15hf)

### 並列テストの利点

1. **実行時間の短縮**: 複数のCPUコアを活用
2. **CI/CDの高速化**: パイプラインの実行時間を短縮
3. **開発効率の向上**: フィードバックループの短縮

### 並列テストの設定

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  # ワーカー数を自動検出
  parallelize(workers: :number_of_processors)
  
  # または固定数を指定
  parallelize(workers: 4)
end
```

### データベースの設定

```yaml
# config/database.yml
test:
  <<: *default
  database: sample_app_test

# 並列テスト用のデータベース（自動生成）
# sample_app_test-0
# sample_app_test-1
# sample_app_test-2
# ...
```

### 落とし穴と対策

#### 1. グローバル状態の共有

```ruby
# 問題のあるコード
class PostTest < ActiveSupport::TestCase
  @@shared_state = []  # グローバル状態（並列実行で問題）
  
  test "test 1" do
    @@shared_state << :test1
  end
end

# 解決策: インスタンス変数を使用
class PostTest < ActiveSupport::TestCase
  def setup
    @local_state = []  # 各テストで独立
  end
end
```

#### 2. ファイルシステムの競合

```ruby
# 問題のあるコード
test "creates file" do
  File.write("tmp/test.txt", "content")  # 並列実行で競合
end

# 解決策: ユニークなファイル名を使用
test "creates file" do
  filename = "tmp/test-#{Process.pid}-#{rand(1000)}.txt"
  File.write(filename, "content")
end
```

#### 3. 外部サービスのモック

```ruby
# 問題のあるコード
test "calls external API" do
  # 外部APIを直接呼び出し（並列実行で問題）
  response = ExternalService.call
end

# 解決策: モックを使用
test "calls external API" do
  stub_request(:get, "https://api.example.com/data")
    .to_return(status: 200, body: '{"data": "test"}')
  
  response = ExternalService.call
end
```

#### 4. データベースの競合

```ruby
# 問題のあるコード
test "creates unique record" do
  Post.create!(title: "Test", slug: "test")  # ユニーク制約で競合
end

# 解決策: ユニークな値を生成
test "creates unique record" do
  Post.create!(title: "Test", slug: "test-#{SecureRandom.hex(4)}")
end
```

### 並列テストのベストプラクティス

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  
  # 各ワーカーで独立した設定
  parallelize_setup do |worker|
    # ログファイルの分離
    Rails.logger.info "Worker #{worker} started"
  end
  
  # クリーンアップ
  parallelize_teardown do |worker|
    # 一時ファイルの削除など
  end
end
```

## 9.3 テスト改善の実践談（thoughtbot）

### 参考資料
- [thoughtbot - A Journey Towards Better Testing Practices](https://thoughtbot.com/blog/a-journey-towards-better-testing-practices)

### テストの原則

1. **テストは独立している**: 他のテストに依存しない
2. **テストは高速**: 実行時間を最小化
3. **テストは明確**: 何をテストしているか明確
4. **テストは保守しやすい**: 変更に強い

### テストの構造

```ruby
# 良い例: 明確な構造
RSpec.describe PostCreationService do
  describe '#call' do
    context 'with valid parameters' do
      it 'creates a post' do
        # テスト内容
      end
    end
    
    context 'with invalid parameters' do
      it 'returns errors' do
        # テスト内容
      end
    end
  end
end
```

### テストデータの管理

```ruby
# 良い例: FactoryBotを使用
let(:user) { create(:user) }
let(:post) { create(:post, user: user) }

# 悪い例: テスト内で直接作成
it 'creates a post' do
  user = User.create!(name: "Test", email: "test@example.com")
  post = Post.create!(title: "Test", content: "Content", user: user)
end
```

### テストの重複を避ける

```ruby
# 良い例: shared_examplesを使用
shared_examples 'a post creator' do
  it 'creates a post' do
    expect { subject.call }.to change(Post, :count).by(1)
  end
end

RSpec.describe PostCreationService do
  it_behaves_like 'a post creator'
end
```

## 9.4 テストを速くする実例（Evil Martians）

### 参考資料
- [Evil Martians - Railing Against Time: Tools and Techniques That Got Us 5x Faster Results](https://evilmartians.com/chronicles/railing-against-time-tools-and-techniques-that-got-us-5x-faster-results)

### テストの最適化戦略

#### 1. データベースの最適化

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  # トランザクションロールバックを使用
  self.use_transactional_tests = true
  
  # フィクスチャの読み込みを最適化
  fixtures :all
end
```

#### 2. 不要なテストのスキップ

```ruby
# 特定の環境でのみ実行
test "slow test", if: ENV['RUN_SLOW_TESTS'] do
  # 時間のかかるテスト
end
```

#### 3. テストの並列化

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
end
```

#### 4. キャッシュの活用

```ruby
# config/environments/test.rb
Rails.application.configure do
  # アセットのコンパイルをスキップ
  config.assets.compile = false
  
  # キャッシュストアを設定
  config.cache_store = :memory_store
end
```

#### 5. テストの分類

```ruby
# spec/spec_helper.rb
RSpec.configure do |config|
  config.define_derived_metadata(file_path: %r{/spec/models/}) do |metadata|
    metadata[:type] = :model
  end
  
  config.define_derived_metadata(file_path: %r{/spec/controllers/}) do |metadata|
    metadata[:type] = :controller
  end
end
```

### CI/CDでの最適化

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test_group: [1, 2, 3, 4]  # テストを分割
    
    steps:
    - name: Run tests
      run: |
        bundle exec rspec --tag "group:${{ matrix.test_group }}"
```

## 9.5 RSpecのセットアップ

### RSpecのインストール

```ruby
# Gemfile
group :development, :test do
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'shoulda-matchers'
  gem 'database_cleaner-active_record'
end
```

```bash
bundle install
rails generate rspec:install
```

### 設定ファイルの更新

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include FactoryBot::Syntax::Methods
  config.include Shoulda::Matchers::ActiveRecord, type: :model
  config.include Shoulda::Matchers::ActiveModel, type: :model
  config.include Devise::Test::ControllerHelpers, type: :controller

  config.before(:suite) do
    DatabaseCleaner.strategy = :transaction
    DatabaseCleaner.clean_with(:truncation)
  end

  config.around(:each) do |example|
    DatabaseCleaner.cleaning do
      example.run
    end
  end
end
```

## 9.6 FactoryBotのセットアップ

### ファクトリーの作成

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    email { Faker::Internet.unique.email }
    password { 'password123' }
    password_confirmation { 'password123' }

    trait :admin do
      role { 'admin' }
    end
  end
end
```

```ruby
# spec/factories/posts.rb
FactoryBot.define do
  factory :post do
    title { Faker::Lorem.sentence }
    content { Faker::Lorem.paragraph }
    user
  end
end
```

## 9.7 モデルテストの作成

### Postモデルのテスト

```ruby
# spec/models/post_spec.rb
require 'rails_helper'

RSpec.describe Post, type: :model do
  describe 'validations' do
    it { should validate_presence_of(:title) }
    it { should validate_length_of(:title).is_at_most(255) }
    it { should validate_presence_of(:content) }
    it { should validate_length_of(:content).is_at_most(10000) }
  end

  describe 'associations' do
    it { should belong_to(:user) }
  end

  describe '#truncated_content' do
    let(:post) { create(:post, content: 'a' * 200) }

    it 'truncates content to 100 characters' do
      expect(post.decorate.truncated_content.length).to be <= 103  # 100 + '...'
    end
  end
end
```

### Userモデルのテスト

```ruby
# spec/models/user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  describe 'validations' do
    it { should validate_presence_of(:name) }
    it { should validate_length_of(:name).is_at_most(50) }
  end

  describe 'associations' do
    it { should have_many(:posts).dependent(:destroy) }
  end
end
```

## 9.8 コントローラーテストの作成

### PostsControllerのテスト

```ruby
# spec/controllers/posts_controller_spec.rb
require 'rails_helper'

RSpec.describe PostsController, type: :controller do
  let(:user) { create(:user) }
  let(:post) { create(:post, user: user) }

  before { sign_in user }

  describe 'GET #index' do
    it 'returns a successful response' do
      get :index
      expect(response).to be_successful
    end

    it 'assigns @posts' do
      get :index
      expect(assigns(:posts)).to include(post)
    end
  end

  describe 'POST #create' do
    context 'with valid parameters' do
      let(:valid_params) { { post: { title: 'Test', content: 'Content' } } }

      it 'creates a new post' do
        expect {
          post :create, params: valid_params
        }.to change(Post, :count).by(1)
      end

      it 'redirects to the post' do
        post :create, params: valid_params
        expect(response).to redirect_to(Post.last)
      end
    end

    context 'with invalid parameters' do
      let(:invalid_params) { { post: { title: '', content: '' } } }

      it 'does not create a new post' do
        expect {
          post :create, params: invalid_params
        }.not_to change(Post, :count)
      end
    end
  end
end
```

## 9.9 テストダブルの活用

### Service Objectのテスト

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
    end

    context 'with invalid parameters' do
      let(:post_params) { { title: '', content: '' } }

      it 'returns failure result' do
        result = service.call
        expect(result.success?).to be false
        expect(result.errors).not_to be_empty
      end
    end
  end
end
```

### Mockの使用

```ruby
# spec/services/post_creation_service_spec.rb
describe '#call' do
  it 'sends notification job' do
    expect(PostCreatedNotificationJob).to receive(:perform_later)
    service.call
  end
end
```

## 9.10 統合テストの作成

### Capybaraのセットアップ

```ruby
# Gemfile
group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
end
```

```ruby
# spec/support/capybara.rb
require 'capybara/rails'
require 'capybara/rspec'

Capybara.default_driver = :rack_test
Capybara.javascript_driver = :selenium_chrome_headless
```

### フィーチャースペック

```ruby
# spec/features/posts_spec.rb
require 'rails_helper'

RSpec.feature 'Posts', type: :feature do
  let(:user) { create(:user) }

  before { sign_in user }

  scenario 'user can create a post' do
    visit new_post_path

    fill_in 'タイトル', with: 'Test Post'
    fill_in '内容', with: 'Test content'
    click_button '投稿する'

    expect(page).to have_content('投稿が作成されました')
    expect(Post.count).to eq(1)
  end
end
```

## 9.11 CI/CDパイプラインの構築（高速化含む）

### GitHub Actionsの設定

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3

    - name: Set up Ruby
      uses: ruby/setup-ruby@v1
      with:
        ruby-version: 3.1.0
        bundler-cache: true

    - name: Setup database
      env:
        RAILS_ENV: test
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
      run: |
        bundle exec rails db:create
        bundle exec rails db:schema:load

    - name: Run tests
      env:
        RAILS_ENV: test
        DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
        REDIS_URL: redis://localhost:6379/0
      run: |
        bundle exec rspec

    - name: Run Brakeman
      run: |
        bundle exec brakeman --no-pager

    - name: Run bundle audit
      run: |
        bundle exec bundle-audit check --update
```

## 9.12 確認ポイント

この章の最後に、以下を確認してください：

1. **テストが通る**
   ```bash
   rails test
   bundle exec rspec
   ```

2. **並列テストが動作している**
   ```bash
   rails test --parallel
   ```

3. **テストカバレッジが十分**
   - モデル、コントローラー、サービスがテストされている

4. **CI/CDパイプラインが動作している**
   - GitHub Actionsが正常に動作している
   - テストの実行時間が最適化されている

## 9.13 参考資料

- [Rails Guides - Testing Rails Applications（英語）](https://guides.rubyonrails.org/testing.html)
- [thoughtbot - A Journey Towards Better Testing Practices](https://thoughtbot.com/blog/a-journey-towards-better-testing-practices)
- [Evil Martians - Railing Against Time: Tools and Techniques That Got Us 5x Faster Results](https://evilmartians.com/chronicles/railing-against-time-tools-and-techniques-that-got-us-5x-faster-results)
- [Parallel Testing in Rails 7: Benefits and Pitfalls](https://dev.to/philsmy/parallel-testing-in-rails-7-benefits-and-pitfalls-15hf)

## 9.14 次の章へ

この章で、高度なテスト戦略を実装しました。次の章では、Asset PipelineとJavaScriptを学びます。

**次の章**: [第10章: Asset PipelineとJavaScript](chapter10-asset-pipeline-javascript.md)

## まとめ

この章では以下を学びました：

- **Testing Rails Applications**: Rails標準のテスト（system test、fixtures、並列実行）
- **並列テスト**: 設定、落とし穴と対策、ベストプラクティス
- **テスト改善の実践談**: thoughtbotの原則とベストプラクティス
- **テスト高速化**: Evil Martiansの実例とCI/CDでの最適化
- **RSpecのセットアップ**: テスト環境の構築
- **FactoryBot**: テストデータの作成
- **テストダブル**: Mock、Stubの活用
- **統合テスト**: CapybaraによるE2Eテスト
- **CI/CD**: GitHub Actionsの構築と高速化
