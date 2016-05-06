# Upload user profile image to s3 with refile

Reference:
* [Refile](https://github.com/refile/refile)

---

```bash
rails g migration AddAvatarIdToUsers avatar_id:string
```

```ruby
# Gemfile

gem "refile", require: "refile/rails"
gem "refile-mini_magick"
gem "refile-s3"
```

```ruby
# config/initializers/refile.rb

require "refile/s3"
require "refile/simple_form"

aws = {
  access_key_id: ENV["AMAZON_ACCESS_KEY_ID"],
  secret_access_key: ENV["AMAZON_SECRET_ACCESS_KEY"],
  region: ENV["AMAZON_BUCKET_REGION"],
  bucket: ENV["AMAZON_BUCKET_NAME"]
}
Refile.cache = Refile::S3.new(prefix: "cache", **aws)
Refile.store = Refile::S3.new(prefix: "store", **aws)
```

```ruby
# .env

AMAZON_ACCESS_KEY_ID=
AMAZON_SECRET_ACCESS_KEY=
AMAZON_BUCKET_REGION=
AMAZON_BUCKET_NAME=
```

```ruby
# .env.example

AMAZON_ACCESS_KEY_ID=
AMAZON_SECRET_ACCESS_KEY=
AMAZON_BUCKET_REGION=
AMAZON_BUCKET_NAME=
```

```coffee
# app/assets/javascripts/application.coffee

#= require refile
#= require_tree ./files
```

```coffee
# app/assets/javascripts/files/file-preview.coffee

(($) ->

  class FilePreview
    constructor: (element) ->
      @dataKey = "file-preview"
      @defaults = width: 200
      @options = @_options(element)

      @_bind()

    _bind: ->
      $("body").on "change", "[data-" + @dataKey + "]", (e) =>
        @_show(e.target)

    _show: (input) ->
      if input.files and input.files[0]
        reader = new FileReader

        reader.onload = (e) =>
          @_image(input)
            .attr("src", e.target.result)
            .attr("width", @options.width)

          $(@options.remove_selector).trigger("remove-#{@options.selector}:reset")

        reader.readAsDataURL input.files[0]

    _image: (input) =>
      $("#" + @options.selector)

    _options: (input) ->
      $.extend @defaults, $(input).data(@dataKey)

  $.fn.filePreview = ->
    return unless @length
    new FilePreview(this)

) jQuery
```

```coffee
# app/assets/javascripts/files/file-remove.coffee

(($) ->

  class FileRemove
    constructor: (element) ->
      @dataKey = "file-remove"
      @options = element.data(@dataKey)
      @element = element
      @_bind()

    _bind: ->
      $("body").on "click", "[data-" + @dataKey + "]", (e) =>
        e.preventDefault()
        @_remove()

      $("body").on "remove-#{@options.selector}:reset", "[data-" + @dataKey + "]", (e) =>
        @element.removeClass("hidden")
        @_reset()

    _remove: =>
      $("#" + @options.selector).attr("src", @options.default)
      $("#" + @options.hidden).prop("checked", true)
      $("#" + @options.file_input).val("")
      @element.addClass("hidden")

    _reset: =>
      $("#" + @options.hidden).val(false)

  $.fn.fileRemove = ->
    return unless @length
    new FileRemove(this)

) jQuery
```

```coffee
# app/assets/javascripts/users/edit-profile.coffee

$(document).on "ready page:load", ->
  $("[data-file-remove]").fileRemove()
  $("[data-file-preview]").filePreview()
```

```ruby
# app/models/user.rb

class User < ActiveRecord::Base
  # ...

  attachment :avatar

  # ...
end
```

```ruby
# app/sanitizers/user/parameter_sanitizer.rb

  def account_update
    default_params.permit(USER_PARAMS, :current_password, :avatar, :remove_avatar)
  end
```

```slim
/ app/views/users/registrations/edit.html.slim

.edit-profile-photo
  .wrap
    = attachment_image_tag(resource, :avatar, :fill, 200, 200, format: "jpg",
        fallback: "defaultavatar.png", id: "avatar-image")
  .actions
    .change
      = f.input :avatar, as: :attachment, direct: true, presigned: true
          input_html: { data: { "file-preview" => { selector: "avatar-image",
            remove_selector: "[data-file-remove]" } } }

    .remove
      = link_to "Remove avatar", "#",
          data: { "file-remove" => { selector: "avatar-image", hidden: "user_remove_avatar",
            file_input: "user_avatar", default: image_path("defaultavatar.png") } },
          class: "button small alert #{resource.avatar.nil? ? 'hidden' : ''}"
      = f.check_box :remove_avatar, class: "hidden"
```

Unfortunately `Refile::FileDouble` does not work with capybara.
Before running tests save any lightweight image as `spec/fixtures/avatar.png`.
For instance [this one](http://i7.5cm.ru/i/TWFB.png).

```ruby
# spec/support/features/shared_contexts/upload_avatar.rb

shared_context "upload avatar" do
  let(:file) { File.open(Rails.root.join("spec/fixtures/avatar.png")) }

  def visit_profile_settings
    visit edit_user_registration_path(current_user)
  end

  def upload_avatar
    visit_profile_settings
    fill_form(:user, avatar: file)
    click_on "Update"
  end

  background do
    stub_request(:any, /amazonaws.com/)
  end
end
```

```ruby
# spec/features/user/account/upload_avatar_spec.rb

require "rails_helper"

feature "Upload Avatar" do
  include_context "current user signed in"
  include_context "upload avatar"

  scenario "User uploads avatar" do
    upload_avatar
    visit_profile_settings

    expect(page).to have_css("img[alt='Avatar']")
  end
end
```

```ruby
# spec/features/user/account/remove_avatar_spec.rb

require "rails_helper"

feature "Remove Avatar", js: true do
  include_context "current user signed in"
  include_context "upload avatar"

  background do
    upload_avatar
    visit_profile_settings
  end

  scenario "User uploads avatar" do
    click_on "Remove avatar"
    click_on "Update"

    visit_profile_settings

    expect(page).to have_css("img[alt='Defaultavatar']")
  end
end
```

Allow to connect to CodeClimate from tests so that build would not break up on SemaphoreCI.

```ruby
# spec/rails_helper.rb

require "webmock/rspec"

WebMock.disable_net_connect!(allow_localhost: true, allow: %w(codeclimate.com))
```

On Amazon S3 go to bucket "Properties" -> "Permissions" -> "Edit CORS Configuration"

Set:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>{HOST or *}</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <AllowedMethod>POST</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>Authorization</AllowedHeader>
        <AllowedHeader>Content-Type</AllowedHeader>
        <AllowedHeader>Origin</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```

You can restrict the allowed origin to only your host (e.g. "http://www.example.com"),
but since your bucket is only writable with authentication anyway, this shouldn't be necessary.
