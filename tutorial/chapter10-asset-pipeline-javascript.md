# 第9章: Asset PipelineとJavaScript

## はじめに

この章では、[Rails Guides - Asset Pipeline](https://guides.rubyonrails.org/asset_pipeline.html)と[Rails Guides - Working with JavaScript in Rails](https://guides.rubyonrails.org/working_with_javascript_in_rails.html)を参考に、アセットパイプラインとJavaScriptの活用方法を学びます。

## 9.1 Asset Pipelineの基礎

### 参考資料
- [Rails Guides - Asset Pipeline（英語）](https://guides.rubyonrails.org/asset_pipeline.html)
- [Rails Guides - Asset Pipeline（日本語）](https://railsguides.jp/asset_pipeline.html)

### Asset Pipelineの概要

Rails 7では、Asset Pipelineの代わりに以下の方法が推奨されています：
- **Import Maps**: JavaScriptの依存関係管理
- **CSS Bundling**: CSSのバンドリング
- **JavaScript Bundling**: JavaScriptのバンドリング（オプション）

### Import Mapsの設定

```ruby
# config/importmap.rb
pin "application", preload: true
pin "@hotwired/turbo-rails", to: "turbo.min.js", preload: true
pin "@hotwired/stimulus", to: "stimulus.min.js", preload: true
```

```erb
<!-- app/views/layouts/application.html.erb -->
<%= javascript_importmap_tags %>
```

### CSSの管理

```ruby
# Gemfile
gem 'cssbundling-rails'
```

```bash
bundle install
rails css:install:bootstrap
```

## 9.2 Turboの活用

### 参考資料
- [Turbo Handbook](https://turbo.hotwired.dev/handbook/introduction)

### Turbo Drive

```erb
<!-- app/views/posts/index.html.erb -->
<%= link_to "投稿詳細", post_path(post), data: { turbo_frame: "post_content" } %>
```

### Turbo Frames

```erb
<!-- app/views/posts/_post.html.erb -->
<%= turbo_frame_tag "post_#{post.id}" do %>
  <h2><%= post.title %></h2>
  <%= link_to "編集", edit_post_path(post), data: { turbo_frame: "_top" } %>
<% end %>
```

### Turbo Streams

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)

    respond_to do |format|
      if @post.save
        format.turbo_stream
        format.html { redirect_to @post }
      else
        format.turbo_stream { render :new }
        format.html { render :new }
      end
    end
  end
end
```

```erb
<!-- app/views/posts/create.turbo_stream.erb -->
<%= turbo_stream.append "posts", @post %>
<%= turbo_stream.update "post_form", partial: "posts/form" %>
```

## 9.3 Stimulusの活用

### 参考資料
- [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction)

### Stimulusコントローラーの作成

```javascript
// app/javascript/controllers/post_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["title", "content", "form"]

  connect() {
    console.log("Post controller connected")
  }

  submit(event) {
    event.preventDefault()
    // フォーム送信の処理
  }

  validate() {
    if (this.titleTarget.value.length < 5) {
      alert("タイトルは5文字以上必要です")
      return false
    }
    return true
  }
}
```

```erb
<!-- app/views/posts/new.html.erb -->
<div data-controller="post">
  <%= form_with model: @post, data: { post_target: "form", action: "submit->post#submit" } do |form| %>
    <%= form.text_field :title, data: { post_target: "title", action: "blur->post#validate" } %>
    <%= form.text_area :content, data: { post_target: "content" } %>
    <%= form.submit "投稿する" %>
  <% end %>
</div>
```

## 9.3 アセットの最適化

### プリコンパイル

```bash
# 本番環境用のアセットをプリコンパイル
RAILS_ENV=production rails assets:precompile
```

### CDNの設定

```ruby
# config/environments/production.rb
config.asset_host = 'https://cdn.example.com'
```

## 9.4 確認ポイント

この章の最後に、以下を確認してください：

1. **Turboが動作している**
   - ページ遷移が高速化されている

2. **Stimulusコントローラーが動作している**
   - JavaScriptが正しく実行されている

3. **アセットが最適化されている**
   - 本番環境でアセットが正しく読み込まれている

## 9.5 参考資料

- [Rails Guides - Asset Pipeline（英語）](https://guides.rubyonrails.org/asset_pipeline.html)
- [Rails Guides - Working with JavaScript in Rails（英語）](https://guides.rubyonrails.org/working_with_javascript_in_rails.html)
- [Turbo Handbook](https://turbo.hotwired.dev/handbook/introduction)
- [Stimulus Handbook](https://stimulus.hotwired.dev/handbook/introduction)

## 9.6 次の章へ

この章で、Asset PipelineとJavaScriptの活用方法を学びました。次の章では、デバッグの方法を学びます。

**次の章**: [第10章: デバッグ](chapter10-debugging.md)

## まとめ

この章では以下を学びました：

- **Asset Pipeline**: Import Maps、CSS Bundling
- **Turbo**: Turbo Drive、Turbo Frames、Turbo Streams
- **Stimulus**: JavaScriptコントローラーの作成
- **アセットの最適化**: プリコンパイル、CDN
