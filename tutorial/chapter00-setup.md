# 第0章: アプリケーションの作成とセットアップ

## はじめに

この章では、Railsアプリケーションを作成し、基本的なセットアップを行います。[Railsチュートリアル第7版](https://railstutorial.jp/chapters/beginning?version=7.0)の第1章を参考にしながら、中級・上級者向けの拡張版の基盤となるアプリケーションを作成します。

## 0.1 前提知識

このチュートリアルを進めるために、以下の知識が必要です：

- RubyとRailsの基本的な構文
- コマンドラインの基本的な操作
- Gitの基本的な使い方
- データベースの基本的な概念

## 0.2 開発環境のセットアップ

### Rubyのインストール確認

```bash
ruby -v
# => ruby 3.1.0 以上が推奨
```

Rubyがインストールされていない場合は、[rbenv](https://github.com/rbenv/rbenv)または[rvm](https://rvm.io/)を使用してインストールしてください。

### Railsのインストール

```bash
gem install rails -v 7.0.0
rails -v
# => Rails 7.0.0
```

### データベースの準備

PostgreSQLを使用します。インストールされていない場合は、以下のコマンドでインストールしてください。

**macOS (Homebrew):**
```bash
brew install postgresql@14
brew services start postgresql@14
```

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib
sudo systemctl start postgresql
```

## 0.3 アプリケーションの作成

### プロジェクトディレクトリの作成

```bash
cd ~/workspace  # または任意のディレクトリ
rails new sample_app --database=postgresql
cd sample_app
```

### Gemfileの確認と調整

`Gemfile`を開いて、以下のgemが含まれていることを確認します：

```ruby
# Gemfile
source "https://rubygems.org"
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby "3.1.0"

gem "rails", "~> 7.0.0"
gem "pg", "~> 1.1"
gem "puma", "~> 5.0"
gem "importmap-rails"
gem "turbo-rails"
gem "stimulus-rails"
gem "jbuilder"
gem "bootsnap", require: false

group :development, :test do
  gem "debug", platforms: %i[ mri mingw x64_mingw ]
  gem "rspec-rails"
  gem "factory_bot_rails"
  gem "faker"
end

group :development do
  gem "web-console"
end
```

### 依存関係のインストール

```bash
bundle install
```

## 0.4 データベースのセットアップ

### データベースの作成

```bash
rails db:create
```

### データベースの確認

```bash
rails db:migrate:status
# => マイグレーションファイルがまだないことを確認
```

## 0.5 Gitリポジトリの初期化

### .gitignoreの確認

`.gitignore`ファイルが自動生成されていることを確認します。必要に応じて追加の設定を行います：

```bash
# .gitignore に追加（既に含まれている場合もあります）
echo "
# 環境変数
.env
.env.local

# ログ
log/*
tmp/*
" >> .gitignore
```

### Gitリポジトリの初期化

```bash
git init
git add .
git commit -m "Initial commit: Rails 7.0 application setup"
```

### GitHubリポジトリの作成（オプション）

GitHubでリポジトリを作成し、リモートリポジトリを追加します：

```bash
git remote add origin https://github.com/your-username/sample_app.git
git branch -M main
git push -u origin main
```

## 0.6 アプリケーションの起動確認

### サーバーの起動

```bash
rails server
# または
rails s
```

ブラウザで `http://localhost:3000` にアクセスして、Railsのデフォルトページが表示されることを確認します。

### 動作確認

- [ ] サーバーが正常に起動する
- [ ] ブラウザで `http://localhost:3000` にアクセスできる
- [ ] Railsのデフォルトページが表示される

## 0.7 基本的な設定

### アプリケーション名の確認

`config/application.rb`を確認します：

```ruby
# config/application.rb
module SampleApp
  class Application < Rails::Application
    config.load_defaults 7.0
    # その他の設定
  end
end
```

### タイムゾーンの設定

```ruby
# config/application.rb
config.time_zone = 'Tokyo'
config.active_record.default_timezone = :local
```

### ロケールの設定

```ruby
# config/application.rb
config.i18n.default_locale = :ja
config.i18n.load_path += Dir[Rails.root.join('config', 'locales', '**', '*.{rb,yml}').to_s]
```

### ログレベルの設定

```ruby
# config/environments/development.rb
config.log_level = :debug
```

## 0.8 開発環境の便利な設定

### エラーページのカスタマイズ（オプション）

```ruby
# Gemfile
group :development do
  gem 'better_errors'
  gem 'binding_of_caller'
end
```

```bash
bundle install
```

### データベースの確認ツール（オプション）

```ruby
# Gemfile
group :development do
  gem 'annotate'
end
```

```bash
bundle install
rails generate annotate:install
```

## 0.9 確認ポイント

この章の最後に、以下を確認してください：

1. **アプリケーションが正常に起動する**
   ```bash
   rails server
   # http://localhost:3000 にアクセスして確認
   ```

2. **データベースが作成されている**
   ```bash
   rails db:version
   ```

3. **Gitリポジトリが初期化されている**
   ```bash
   git status
   ```

4. **依存関係がインストールされている**
   ```bash
   bundle check
   ```

## 0.10 トラブルシューティング

### PostgreSQLの接続エラー

```bash
# PostgreSQLが起動しているか確認
pg_isready

# 起動していない場合
brew services start postgresql@14  # macOS
sudo systemctl start postgresql    # Linux
```

### ポート3000が使用中

```bash
# 別のポートで起動
rails server -p 3001
```

### 権限エラー

```bash
# ファイルの権限を確認
chmod +x bin/rails
```

## 0.11 次の章へ

この章で基本的なRailsアプリケーションのセットアップが完了しました。次の章では、ユーザー認証機能と基本的なCRUD操作を実装します。

**次の章**: [第1章: Railsの内部動作を理解する](chapter01-rails-internals.md)

## まとめ

この章では以下を学びました：

- Railsアプリケーションの作成
- PostgreSQLデータベースのセットアップ
- Gitリポジトリの初期化
- 基本的な設定の確認

次の章では、実際にアプリケーションに機能を追加していきます。
