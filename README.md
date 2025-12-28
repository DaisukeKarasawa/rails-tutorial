# Railsチュートリアル v2.0 - 中級・上級者向け拡張版（ハンズオン形式）

## 概要

このチュートリアルは、[Railsチュートリアル第7版](https://railstutorial.jp/chapters/beginning?version=7.0)の内容をベースに、中級・上級者向けの内容に拡張したものです。

**実際にアプリケーションを作成しながら、段階的に拡張していくハンズオン形式**で学習を進めます。最初は基本的なRailsアプリケーションを作成し、それを拡張しながら高度なトピックを学んでいきます。

## 対象読者

- Railsの基礎を理解している方
- 実践的なアプリケーション開発の経験がある方
- パフォーマンス、セキュリティ、アーキテクチャに興味がある方
- 本番環境での運用を学びたい方

## 前提知識

- RubyとRailsの基本的な構文と概念
- MVCアーキテクチャの理解
- データベースの基本的な操作
- Gitの基本的な使い方
- HTTPとWebアプリケーションの基礎知識

## 学習の進め方

このチュートリアルでは、**一つのアプリケーションを拡張しながら学習**します。

1. **第0章**で基本的なRailsアプリケーションを作成します
2. **第1章**で基本的な機能（ユーザー、投稿など）を実装します
3. **第2章以降**で、作成したアプリケーションを拡張しながら、中級・上級者向けのトピックを学習します

各章では、実際にコードを書き、コマンドを実行しながら学習を進めます。

## 構成

各章のMarkdownファイルは `tutorial/` ディレクトリに配置されています。

このチュートリアルは、[Rails Guides](https://guides.rubyonrails.org/)と[Rails API Documentation](https://api.rubyonrails.org/)の内容を網羅するように構成されています。

### [第0章: アプリケーションの作成とセットアップ](tutorial/chapter00-setup.md)

- 開発環境のセットアップ
- Railsアプリケーションの作成
- Gitリポジトリの初期化
- 基本的な設定

### [第1章: Railsの内部動作を理解する](tutorial/chapter01-rails-internals.md)

- **Autoloading / Zeitwerk（超重要）**: 定数解決、eager load、開発時リロードの仕組み
- **Rails Initialization Process**: 起動順序、initializer、Railtie/Engine
- **Configuring Rails Applications**: 環境差分、設定のスコープ、カスタム設定
- **The Rails Command Line**: bin/rails、generator、runner、タスク運用
- **Active Support Core Extensions**: Rails流Ruby拡張の前提を整理
- **CurrentAttributes**: 便利だが事故りやすい論点（賛否・注意点を知る）
- **参考**:
  - [Rails Guides - Autoloading and Reloading Constants](https://guides.rubyonrails.org/autoloading_and_reloading_constants.html)
  - [Rails Guides - The Rails Initialization Process](https://guides.rubyonrails.org/initialization.html)
  - [Rails Guides - Configuring Rails Applications](https://guides.rubyonrails.org/configuring.html)
  - [Rails Guides - The Rails Command Line](https://guides.rubyonrails.org/command_line.html)
  - [Rails Guides - Active Support Core Extensions](https://guides.rubyonrails.org/active_support_core_extensions.html)
  - [CurrentAttributesの賛否・注意点を知る（TechRacho）](https://techracho.bpsinc.jp/hachi8833/2024_01_25/43810)

### [第2章: 基本的なアプリケーション構築](tutorial/chapter02-basic-application.md)

- ユーザーモデルの作成
- 投稿（Post）モデルの作成
- 基本的なCRUD操作
- 認証機能の実装（Devise）
- 基本的なビューとコントローラー

### [第3章: Active Recordの高度な活用](tutorial/chapter03-active-record-advanced.md)

- **Active Record Basics**: 基礎を正しい前提に戻す（CRUD操作、属性操作）
- **Active Record Query Interface**: N+1、eager loading、locking、EXPLAINまで含む必修
  - `includes/preload/eager_load`の挙動整理と使い分け
  - **参考**: [includes/preload/eager_load の挙動整理（日本語）](https://tech.stmn.co.jp/entry/2020/11/30/145159)
- **Associations**: 設計の癖（STI/Delegated Types含む）を公式で整理
- **Migrations**: 安全なスキーマ変更・運用の原則（ロールバック戦略含む）
- **Validations**: strict/条件付き/独自バリデーションなど実戦で必要な所
- **Callbacks**: `after_commit`/`after_rollback`、トランザクション境界の罠
- **Multiple Databases**: 読み書き分離、接続切替、シャーディング等
- **Active Record Encryption**: PII保護・移行・共存パターンまで公式で押さえる
- **参考**:
  - [Rails Guides - Active Record Basics](https://guides.rubyonrails.org/active_record_basics.html)
  - [Rails Guides - Active Record Query Interface](https://guides.rubyonrails.org/active_record_querying.html)
  - [Rails Guides - Associations](https://guides.rubyonrails.org/association_basics.html)
  - [Rails Guides - Migrations](https://guides.rubyonrails.org/active_record_migrations.html)
  - [Rails Guides - Validations](https://guides.rubyonrails.org/active_record_validations.html)
  - [Rails Guides - Callbacks](https://guides.rubyonrails.org/active_record_callbacks.html)
  - [Rails Guides - Multiple Databases](https://guides.rubyonrails.org/active_record_multiple_databases.html)
  - [Rails Guides - Active Record Encryption](https://guides.rubyonrails.org/active_record_encryption.html)

### [第4章: Action ControllerとAction Viewの詳細](tutorial/chapter04-action-controller-view.md)

- Action Controllerの詳細（フィルター、Strong Parameters、respond_to、レンダリング）
- **Action View Overview**: テンプレ/partials/layoutsの基礎を公式で再確認
- **Layouts and Rendering**: render/redirect、ネストlayout等の定番
- **Action View Helpers**: sanitize/capture/cache/debugなど"地味に重要"
- **ViewComponent**: コンポーネント指向のビュー設計（採用プロジェクト多い）
- **API-only Applications**: API専用構成の取捨選択
- ビューの最適化（フラグメントキャッシュ）
- **参考**:
  - [Rails Guides - Action Controller Overview](https://guides.rubyonrails.org/action_controller_overview.html)
  - [Rails Guides - Action View Overview](https://guides.rubyonrails.org/action_view_overview.html)
  - [Rails Guides - Layouts and Rendering](https://guides.rubyonrails.org/layouts_and_rendering.html)
  - [Rails Guides - Action View Helpers](https://guides.rubyonrails.org/action_view_helpers.html)
  - [Rails Guides - API-only Applications](https://guides.rubyonrails.org/api_app.html)
  - [ViewComponent（公式）](https://viewcomponent.org/)

### [第5章: ルーティングの高度な使い方](tutorial/chapter05-routing.md)

- **Rails Routing from the Outside In**: 制約、ネスト、concerns等の設計感
- リソースルーティングの詳細（アクションの制限、カスタムアクション）
- 名前空間とスコープ
- 制約（Constraints）の設計感
- Concernsによるルーティングの整理の設計感
- ネストしたリソースの設計（浅いネストの原則）
- APIルーティング
- **参考**: [Rails Guides - Routing from the Outside In](https://guides.rubyonrails.org/routing.html)

### [第6章: 高度なRailsアーキテクチャと設計パターン](tutorial/chapter06-advanced-architecture.md)

- Service Objectパターンの実装
- Form Objectパターンの実装
- Query Objectパターンの実装
- Decoratorパターンの実装
- **Engines / Railties**: モノリス分割、プラグイン化、拡張ポイント理解
- 既存コードのリファクタリング
- **参考**:
  - [Rails Edge Guides - Engines](https://edgeguides.rubyonrails.org/engines.html)

### [第7章: パフォーマンス最適化とスケーリング](tutorial/chapter07-performance-optimization.md)

- N+1問題の特定と解決
- データベースクエリの最適化
- **Caching with Rails**: fragment/SQL cache等、キャッシュ戦略の全体像
- **Active Support Instrumentation**: 計測・可観測性の入口（Notificationsにも繋がる）
- **ActiveSupport::Notifications**: イベント計測の仕様を確定させる
- **Active Job Basics**: 標準APIと（近年）デフォルトキュー基盤（Solid Queue）の説明
- バックグラウンドジョブの実装（Sidekiq + Active Job）
- データベースインデックスの追加
- **クエリ最適化のためのロギング**: スロークエリの検出、クエリの可視化
- **Active Record重い画面の対処Tips**: 必要なカラムのみ取得、集計の最適化、ページネーションの最適化
- **参考**:
  - [Rails Guides - Active Job Basics](https://guides.rubyonrails.org/active_job_basics.html)
  - [Rails Guides - Caching with Rails](https://guides.rubyonrails.org/caching_with_rails.html)
  - [Rails Guides - Active Support Instrumentation](https://guides.rubyonrails.org/active_support_instrumentation.html)
  - [Rails API - ActiveSupport::Notifications](https://api.rubyonrails.org/classes/ActiveSupport/Notifications.html)
  - [Evil Martians - 5 Tips for ActiveRecord Dashboards](https://evilmartians.com/chronicles/5-tips-for-activerecord-dashboards)
  - [Evil Martians - Rails Query Optimizations](https://evilmartians.com/chronicles/rails-query-optimizations)

### [第8章: セキュリティの深掘りとベストプラクティス](tutorial/chapter08-security.md)

- **Securing Rails Applications（公式）**: CSRF/XSS/SQLi/認可・セッション等の基本線
- **Strong Parameters**: permit/requireの実挙動を確認
- 認可機能の実装（Pundit）
- CSRF、XSS対策の強化
- SQLインジェクション対策の詳細
- セッション管理のセキュリティ
- パスワードの安全な管理
- ファイルアップロードのセキュリティ
- ログのセキュリティ
- セキュアなAPI設計
- セキュリティヘッダーの設定
- 脆弱性スキャンの実施
- **参考**:
  - [Rails Guides - Securing Rails Applications（英語）](https://guides.rubyonrails.org/security.html)
  - [Rails Guides - Securing Rails Applications（日本語）](https://railsguides.jp/security.html)
  - [Rails API - ActionController::Parameters](https://api.rubyonrails.org/classes/ActionController/Parameters.html)

### [第9章: 高度なテスト戦略と品質保証](tutorial/chapter09-advanced-testing.md)

- **Testing Rails Applications（公式）**: system test、fixtures、並列実行などRails標準の全体像
- **並列テスト（Parallel Testing）**: 設定、落とし穴と対策、ベストプラクティス
- **テスト改善の実践談**: thoughtbotの原則とベストプラクティス
- **テスト高速化**: Evil Martiansの実例とCI/CDでの最適化
- RSpecのセットアップと基本テスト
- テストダブルの活用
- 統合テストとE2Eテスト
- パフォーマンステスト
- CI/CDパイプラインの構築（高速化含む）
- **参考**:
  - [Rails Guides - Testing Rails Applications](https://guides.rubyonrails.org/testing.html)
  - [thoughtbot - A Journey Towards Better Testing Practices](https://thoughtbot.com/blog/a-journey-towards-better-testing-practices)
  - [Evil Martians - Railing Against Time: Tools and Techniques That Got Us 5x Faster Results](https://evilmartians.com/chronicles/railing-against-time-tools-and-techniques-that-got-us-5x-faster-results)
  - [Parallel Testing in Rails 7: Benefits and Pitfalls](https://dev.to/philsmy/parallel-testing-in-rails-7-benefits-and-pitfalls-15hf)

### [第10章: Asset PipelineとJavaScript](tutorial/chapter10-asset-pipeline-javascript.md)

- Asset Pipelineの基礎（Import Maps、CSS Bundling）
- Turboの活用（Turbo Drive、Turbo Frames、Turbo Streams）
- Stimulusの活用（JavaScriptコントローラー）
- アセットの最適化
- **参考**: [Rails Guides - Asset Pipeline](https://guides.rubyonrails.org/asset_pipeline.html), [Rails Guides - Working with JavaScript in Rails](https://guides.rubyonrails.org/working_with_javascript_in_rails.html)

### [第11章: デバッグ](tutorial/chapter11-debugging.md)

- **Debugging Rails Applications（公式）**: ログ/例外/スタックトレース/デバッガの使い所
- ログの詳細な使い方（ログレベル、出力先、タグ付きログ、構造化ログ、フィルタリング）
- 例外とスタックトレースの理解（例外の種類、スタックトレースの読み方、例外の再発生、カスタム例外）
- デバッガの使い所（debug gem、ブレークポイント、デバッガーコマンド、条件付きブレークポイント）
- エラーページ（better_errors）
- クエリのデバッグ
- パフォーマンスのデバッグ
- デバッグのベストプラクティス
- **参考**: [Rails Guides - Debugging Rails Applications](https://guides.rubyonrails.org/debugging_rails_applications.html)

### [第12章: 本番環境のデプロイと運用](tutorial/chapter12-production-deployment.md)

- **Tuning Performance for Deployment**: Puma/並行性/環境変数の基本
- **Pumaの設定**: スレッドとワーカーの理解、メモリ使用量の最適化
- **Error Reporting in Rails Applications**: Rails標準のエラーレポート基盤を理解
- Docker化
- 本番環境へのデプロイ
- モニタリングとログ管理
- エラートラッキング（Sentry + Rails.error）
- バックアップ戦略
- **参考**:
  - [Rails Guides - Tuning Performance for Deployment](https://guides.rubyonrails.org/tuning_performance_for_deployment.html)
  - [Rails Guides - Error Reporting in Rails Applications](https://guides.rubyonrails.org/error_reporting.html)
  - [Heroku Dev Center - Deploying Rails Applications with the Puma Web Server](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server)
  - [Judo Scale - Puma Default Threads Changed](https://judoscale.com/blog/puma-default-threads-changed)

### [第13章: 実践的な開発手法とワークフロー](tutorial/chapter13-practical-development.md)

- コードレビューの実践
- リファクタリングの実践
- API設計とバージョニング
- マイクロサービスへの移行準備

## 使い方

1. **第0章から順番に進めてください**
2. 各章の指示に従って、実際にコードを書き、コマンドを実行してください
3. 各章の最後にある「確認ポイント」で動作確認を行ってください
4. 問題が発生した場合は、各章の「トラブルシューティング」セクションを参照してください

## 注意事項

- **実際にアプリケーションを作成しながら学習します**
- 各章は前の章で作成したアプリケーションを拡張する形式です
- 環境に応じてコマンドや設定を調整してください
- 各章の作業を完了してから次の章に進んでください

## 参考資料

このチュートリアルは、以下の公式ドキュメントを参考に作成されています：

### 公式ドキュメント

- **[Rails Guides（英語）](https://guides.rubyonrails.org/)**: 中・上級へ進むための一次情報のハブ

  - [Active Record Querying](https://guides.rubyonrails.org/active_record_querying.html)
  - [Active Record Associations](https://guides.rubyonrails.org/association_basics.html)
  - [Action Controller Overview](https://guides.rubyonrails.org/action_controller_overview.html)
  - [Action View Overview](https://guides.rubyonrails.org/action_view_overview.html)
  - [Routing](https://guides.rubyonrails.org/routing.html)
  - [Security](https://guides.rubyonrails.org/security.html)
  - [Caching with Rails](https://guides.rubyonrails.org/caching_with_rails.html)
  - [Testing Rails Applications](https://guides.rubyonrails.org/testing.html)
  - [Asset Pipeline](https://guides.rubyonrails.org/asset_pipeline.html)
  - [Working with JavaScript in Rails](https://guides.rubyonrails.org/working_with_javascript_in_rails.html)
  - [Debugging Rails Applications](https://guides.rubyonrails.org/debugging_rails_applications.html)

- **[Rails API Documentation（英語）](https://api.rubyonrails.org/)**: 実装の挙動を確定させたい時の最終審判

  - [ActiveRecord::QueryMethods](https://api.rubyonrails.org/classes/ActiveRecord/QueryMethods.html)
  - [ActiveRecord::Associations](https://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html)
  - [ActionController::Base](https://api.rubyonrails.org/classes/ActionController/Base.html)
  - [ActionView::Helpers](https://api.rubyonrails.org/classes/ActionView/Helpers.html)

- **[Rails Guides 日本語（翻訳）](https://railsguides.jp/)**: 日本語で追いたい時（内容は版に注意）
  - [Active Record クエリインターフェイス](https://railsguides.jp/active_record_querying.html)
  - [Active Record の関連付け](https://railsguides.jp/association_basics.html)
  - [Action Controller の概要](https://railsguides.jp/action_controller_overview.html)
  - [Action View の概要](https://railsguides.jp/action_view_overview.html)
  - [Rails のルーティング](https://railsguides.jp/routing.html)

### その他の参考資料

- [Railsチュートリアル第7版](https://railstutorial.jp/chapters/beginning?version=7.0)
- [Turbo Handbook](https://turbo.hotwired.dev/handbook/introduction)
- [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction)
