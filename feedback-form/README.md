# Feedback form

```ruby
# app/controllers/feedbacks_controller.rb
class FeedbacksController < ApplicationController
  expose_decorated(:feedback) do
    Feedback.new(feedback_attributes)
  end

  def new
  end

  def create
    DeliveryNotifications.call(feedback_attributes) if feedback.valid?
    respond_with(feedback, location: root_path)
  end

  private

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
# app/interactors/delivery_notifications.rb
class DeliveryNotifications
  ROOM = ENV["HIPCHAT_ROOM"]

  include Interactor

  def call
    if context
      send_email
      send_hipchat
    else
      context.fail!(message: "Delivery Failed!")
    end
  end

  private

  def client
    @_client ||= HipchatClient.connect
  end

  def send_email
    ApplicationMailer.feedback(context).deliver_now!
  end

  def send_hipchat
    return false unless client || context.name

    client[ROOM].send("", message)
  end

  def message
    "Feedback from #{context.name}: <br><br>
    #{context.message}      <br><br>
    ____________________________<br>
    email: #{context.email}     <br>
    phone: #{context.phone}"
  end
end
```

```ruby
# app/services/hipchat_client.rb
class HipchatClient
  TOKEN = ENV["HIPCHAT_API_TOKEN"]
  class << self
    def connect
      client || nil
    end

    private

    def client
      HipChat::Client.new(TOKEN, api_version: "v2", server_url: "https://fs.hipchat.com")
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
    user
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
# spec/interactors/delivery_notifications_spec.rb

require "rails_helper"

describe DeliveryNotifications do
  let(:feedback_attributes) { FactoryGirl.attributes_for(:feedback).merge(user: create(:user)) }
  let(:mailer_double) { double }
  let(:client_double) { double("HipChat::Client") }
  let(:room_double) { double() }

  subject(:call) { described_class.call(feedback_attributes) }

  before do
    allow(ApplicationMailer).to receive(:feedback).and_return(mailer_double)
    allow(mailer_double).to receive(:deliver_now!).and_return("YES SIR!")
    allow(HipchatClient).to receive(:connect).and_return(:client_double)
    allow(client_double).to receive(:[]).with(described_class::ROOM).and_return(room_double)
    allow(room_double).to receive(:send).and_return("ding dong!")
  end

  it "sends email and HipChat notifications" do
    expect(ApplicationMailer).to receive(:feedback).and_return(mailer_double)
    expect(mailer_double).to receive(:deliver_now!).and_return("YES SIR!")

    expect(HipchatClient).to receive(:connect).and_return(client_double)
    expect(client_double).to receive(:[]).with(described_class::ROOM).and_return(room_double)
    expect(room_double).to receive(:send).and_return("ding dong!")

    call
  end
end
```

```ruby
# spec/services/hipchat_client_spec.rb
require "rails_helper"

describe HipchatClient do
  let(:call) { described_class.connect }

  before do
    stub_const("HipchatClient::TOKEN", "123456")
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
