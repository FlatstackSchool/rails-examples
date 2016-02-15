# Feedback form

```ruby
# app/controllers/feedbacks_controller.rb
class FeedbacksController < ApplicationController
  expose(:feedback) { Feedback.new(feedback_attributes) }
  expose(:decorated_feedback) { FeedbackDecorator.new(feedback) }

  def new
  end

  def create
    perform_delivery if feedback.valid?
    respond_with(feedback, location: root_path)
  end

  private

  def perform_delivery
    ApplicationMailer.feedback(feedback).deliver_now!
    HipchatInteractor::Organizer.call(feedback_attributes)
  end

  def feedback_attributes
    params
      .fetch(:feedback, {})
      .permit(:email, :name, :message, :phone, :user)
      .merge(user: current_user)
  end
end
```

```ruby
# app/models/feedback.rb
class Feedback
  include ActiveModel::Model

  attr_accessor :email, :name, :message, :phone, :user

  validates :email, :name, :message, presence: true
  validates :email, format: Devise.email_regexp
end
```

```ruby
# app/decorators/feedback_decorator.rb
class FeedbackDecorator < ApplicationDecorator
  delegate :user, :phone, :message

  def author
    user ? user.full_name : ""
  end

  def email
    user ? user.email : ""
  end
end
```

```ruby
# app/interactors/hipchat_interactor/organizer.rb
module HipchatInteractor
  class Organizer
    include Interactor::Organizer

    organize HipchatInteractor::Connector, HipchatInteractor::Messenger
  end
end
```

```ruby
# app/interactors/hipchat_interactor/connector.rb
module HipchatInteractor
  class Connector
    TOKEN = ENV["HIPCHAT_API_TOKEN"]

    include Interactor

    def call
      if client
        context.client = client
      else
        context.fail!(message: "Couldn't establish connection to HipChat servers.")
      end
    end

    private

    def client
      @_client ||= HipChat::Client
        .new(TOKEN, api_version: "v2", server_url: "https://fs.hipchat.com")
    end
  end
end
```

```ruby
# app/interactors/hipchat_interactor/messenger.rb
module HipchatInteractor
  class Messenger
    ROOM = ENV["HIPCHAT_ROOM"]

    include Interactor

    def call
      if context.client && context.name
        context.client[ROOM].send(name, message)
      else
        context.fail!(message: "Couldn't send a message")
      end
    end

    private

    def name
      context.name.to_s
    end

    def message
      "Feedback from #{name}: <br><br>
      #{context.message}      <br><br>
      ____________________________<br>
      email: #{context.email}     <br>
      phone: #{context.phone}"
    end
  end
end
```

```ruby
# .env.example
# We send all feedback email to this address
FEEDBACK_EMAIL=test@example.com
HIPCHAT_ROOM=room_name
HIPCHAT_API_TOKEN=1234567890
```

```ruby
# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  def feedback(feedback)
    @feedback = feedback
    mail(subject: "Feedback", from: feedback.email, to: ENV.fetch("FEEDBACK_EMAIL"))
  end
end
```

```slim
# app/views/application/_navigation_main.html.slim

ul.left
  - if user_signed_in?
    = active_link_to "Home", root_path, active: :exclusive, wrap_tag: :li
  = active_link_to "Feedback", new_feedback_path, active: :exclusive, wrap_tag: :li
```

```slim
# app/views/application_mailer/feedback.html.slim
p
  | Hello, here is feedback from #{@feedback.name} (#{@feedback.email} #{@feedback.phone})

blockquote
  = @feedback.message
```

```erb
# app/views/application_mailer/feedback.text.erb
Hello, here is feedback from <%= @feedback.name %> (<%= @feedback.email %> <%= @feedback.phone %>)

<%= @feedback.message %>
```

```slim
# app/views/feedbacks/new.html.slim
ruby:
  title("Contact Us")
  description("Contact the team")
  keywords("Contact, Email, Touch, Getting in touch")
  meta robots: "noindex"

.row
  .columns
    h2 Feedback

.row
  .medium-6.columns
    = simple_form_for(decorated_feedback, url: feedback_path) do |f|

      legend
        ' Have a question?
        | You may find the answer in the #{link_to("FAQ", "#")}.

      .form-inputs
        = f.input :name, input_html: { value: decorated_feedback.author }
        = f.input :email, input_html: { value: decorated_feedback.email }
        = f.input :phone
        = f.input :message, as: :text

      .form-actions
        = f.button :submit, "Submit"
```

```yml
# config/locales/flash.en.yml
en:
  flash:
    actions:
      create:
        notice: '%{resource_name} was successfully created.'
        alert: '%{resource_name} could not be created.'
      update:
        notice: '%{resource_name} was successfully updated.'
        alert: '%{resource_name} could not be updated.'
      destroy:
        notice: '%{resource_name} was successfully destroyed.'
        alert: '%{resource_name} could not be destroyed.'

    feedbacks:
        create:
          notice: "Email was successfully sent."
```

```ruby
# config/routes.rb
Rails.application.routes.draw do
  resource :feedback, only: %i(new create)
end
```

```ruby
# spec/factories/feedbacks.rb
FactoryGirl.define do
  factory :feedback do
    email
    name  { Faker::Name.name }
    message { Faker::Lorem.paragraph }
    phone { Faker::PhoneNumber.phone_number }
  end
end
```

```ruby
# spec/features/visitor/create_feedback_spec.rb
require "rails_helper"

feature "Create Feedback." do
  let(:feedback_attributes) { attributes_for(:feedback) }

  scenario "Visitor creates feedback" do
    visit new_feedback_path

    fill_form :feedback, feedback_attributes
    click_button "Submit"

    open_email(ENV.fetch("FEADBACK_EMAIL"))

    expect(current_email).to have_subject("Feedback")
    expect(current_email).to be_delivered_from(feedback_attributes[:email])

    expect(current_email).to have_body_text(feedback_attributes[:name])
    expect(current_email).to have_body_text(feedback_attributes[:phone])
    expect(current_email).to have_body_text(feedback_attributes[:email])
    expect(current_email).to have_body_text(feedback_attributes[:message])

    expect(page).to have_content("Email was successfully sent.")
  end
end
```

```ruby
# spec/interactors/hipchat_interactor/organizer_spec.rb

require "rails_helper"

describe HipchatInteractor::Organizer do
  let(:params) do
    {
      email: "makcimka1994@gmail.com",
      name: "Max L",
      message: "1337",
      phone: "+79991560391"
    }
  end

  let(:call) { described_class.call(params) }

  before do
    allow_any_instance_of(HipchatInteractor::Connector).to receive(:call)
    allow_any_instance_of(HipchatInteractor::Messenger).to receive(:call)
  end

  it "calls HipchatInteractor::Connector and HipchatInteractor::Messenger" do
    expect_any_instance_of(HipchatInteractor::Connector).to receive(:call)
    expect_any_instance_of(HipchatInteractor::Messenger).to receive(:call)

    call
  end
end
```

```ruby
# spec/interactors/hipchat_interactor/connector_spec.rb

require "rails_helper"

describe HipchatInteractor::Connector do
  let(:call) { described_class.call }

  before do
    stub_const("HipchatInteractor::Connector::TOKEN", "123456")
    allow(HipChat::Client)
      .to receive(:new)
      .with("123456", api_version: "v2", server_url: "https://fs.hipchat.com")
  end

  it "authenticates to Hipchat" do
    expect(HipChat::Client)
      .to receive(:new)
      .with("123456", api_version: "v2", server_url: "https://fs.hipchat.com")

    call
  end
end
```

```ruby
# spec/interactors/hipchat_interactor/messenger_spec.rb

require "rails_helper"

describe HipchatInteractor::Messenger do
  xit "pending" do
  end
end
```

```ruby
# spec/decorators/feedback_decorator_spec.rb

require "rails_helper"

describe FeedbackDecorator do
  let(:user) { User.new(full_name: "Name", email: "user@example.com") }
  let(:feedback) { build(:feedback, user: user) }

  subject { described_class.new(feedback) }

  its(:author) { is_expected.to eq user.full_name }
  its(:email) { is_expected.to eq user.email}

  context "when there is no user" do
    let(:user) { nil }

    its(:author) { is_expected.to eq "" }
    its(:email) { is_expected.to eq ""}
  end
end
```

```ruby
# Gemfile.rb

gem 'hipchat'

group :development, :test do
  gem 'rspec-its'
end
```
