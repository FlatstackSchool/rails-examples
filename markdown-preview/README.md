# Preview Markdown as HTML

```ruby
# Gemfile
gem "github-markdown"
gem "html-pipeline"
gem "rinku"
gem "sanitize"
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :markdowns, only: %i(create)
end
```

```ruby
# app/controllers/markdowns_controller.rb
class MarkdownsController < ApplicationController
  def create
    render html: Markdown.new(params[:markdown]).to_html
  end
end
```

```ruby
# spec/controllers/markdowns_controller_spec.rb
require "rails_helper"

describe MarkdownsController do
  describe "POST create" do
    it "renders the markdown as HTML" do
      post :create, markdown: "**Hello**"
      expect(response.body).to eql("<p><strong>Hello</strong></p>")
    end
  end
end
```

```ruby
# app/models/markdown.rb
class Markdown
  FILTERS = [
    HTML::Pipeline::MarkdownFilter, HTML::Pipeline::SanitizationFilter,
    HTML::Pipeline::AutolinkFilter
  ]

  CONTEXT = {
    gfm: true
  }

  attr_reader :source, :pipeline
  private :source, :pipeline

  def initialize(source, filters: FILTERS, context: CONTEXT)
    @source = source
    @pipeline = HTML::Pipeline.new(filters, context)
  end

  def to_html
    pipeline.to_html(source).html_safe
  end
end
```

```slim
# app/views/feedbacks/new.html.slim
= simple_form_for(feedback.object) do |f|
  = f.input :good, as: :markdown, url: markdowns_path, input_html: { rows: 5 }
  = f.button :submit, "Send Feedback"
```

```coffee
# app/assets/javascripts/markdown.coffee
class MarkdownPreview
  constructor: (el) ->
    @$el = $(el)
    @options = @$el.data("markdown-preview")

    @_bindEvents()

  _bindEvents: ->
    @$el.on "toggled", @_preview

  _preview: =>
    $.post @options.url, { markdown: $("##{@options.source}").val() }, (html) =>
      @$el.html(html)

for el in $("[data-markdown-preview]")
  new MarkdownPreview(el)
```

```ruby
# app/inputs/markdown_input.rb
class MarkdownInput < SimpleForm::Inputs::Base
  DEFAULT_LOADING = "Loading..."

  def input(_)
    template.content_tag(:div) do
      template.concat tab_titles
      template.concat tab_contents
    end
  end

  def tab_titles
    template.content_tag(:ul, class: "tabs", "data-tab" => true) do
      template.concat tab_title(:write)
      template.concat tab_title(:preview)
    end
  end

  def tab_contents
    template.content_tag(:div, class: "tabs-content") do
      template.concat tab_write
      template.concat tab_preview
    end
  end

  def tab_write
    tab_content(:write) do
      @builder.text_area(attribute_name, input_html_options)
    end
  end

  def tab_preview
    tab_content(:preview, data: data) do
      input_options[:loading] || DEFAULT_LOADING
    end
  end

  def tab_content(type, options = {}, &block)
    template.content_tag(:div, { class: "content #{active_class(type)}", id: tab_id(type) }.merge(options)) do
      template.capture(&block)
    end
  end

  def tab_title(type)
    template.content_tag(:li, class: "tab-title #{active_class(type)}") do
      template.link_to(type.to_s.titleize, "##{tab_id(type)}", class: "tab-link")
    end
  end

  def active_class(type)
    type == :write ? "active" : ""
  end

  def tab_id(type)
    "#{attribute_name}-#{type}"
  end

  def data
    { "markdown-preview" => { url: input_options[:url], source: input_class } }
  end
end
```
