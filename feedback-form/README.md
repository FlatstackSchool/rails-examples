# Feedback form

```ruby
# app/controllers/feedbacks_controller.rb
class FeedbacksController < ApplicationController
  expose(:feedback) { Feedback.new(feedback_attributes) }

  def new
  end

  def create
    SubmitFeedback.call(feedback: feedback) if feedback.valid?

    respond_with(feedback, location: root_path)
  end

  private

  def feedback_attributes
    params.fetch(:feedback, default_attributes).permit(:email, :name, :message, :phone)
  end

  def default_attributes
    return {} unless current_user

    {
      email: current_user.email,
      name: current_user.full_name
    }
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

```slim
# app/views/feedbacks/new.html.slim
- title("Contact Us")
- description("Contact the team")
- keywords("Contact, Email, Touch, Getting in touch")
- meta robots: "noindex"

.row
  .columns
    h2 Feedback

.row
  .medium-6.columns
    = simple_form_for(feedback, url: feedback_path) do |f|

      legend
        ' Have a question?
        | You may find the answer in the #{link_to("FAQ", "#")}.

      .form-inputs
        = f.input :name
        = f.input :email
        = f.input :phone
        = f.input :message, as: :text

      .form-actions
        = f.button :submit, "Submit"
```

```ruby
# .env.example
# We send all feedback email to this address
FEEDBACK_EMAIL=test@example.com
HIPCHAT_ROOM_ID=hip_chat_room_id
HIPCHAT_AUTH_TOKEN=valid_room_token
```

```ruby
# Gemfile
...
gem "hipchat"
...
```

```ruby
# app/interactors/submit_feedback.rb
class SubmitFeedback
  include Interactor::Organizer

  organize SendFeedbackEmail, SendHipChatNotification
end
```

```ruby
# app/interactors/send_feedback_email.rb
class SendFeedbackEmail
  include Interactor

  delegate :feedback, to: :context

  def call
    ApplicationMailer.feedback(feedback).deliver_now!
  end
end
```

```ruby
# app/interactors/send_hip_chat_notification.rb
class SendHipChatNotification
  include Interactor

  delegate :feedback, to: :context

  def call
    HipChatClient.new.send_feedback(feedback)
  end
end
```

```ruby
# app/models/hip_chat_client.rb
class HipChatClient
  attr_reader :room, :auth_token
  private :room, :auth_token

  def initialize
    @room = ENV["HIPCHAT_ROOM_ID"]
    @auth_token = ENV["HIPCHAT_AUTH_TOKEN"]
  end

  def send_feedback(feedback)
    feedback_room.send(feedback.name, feedback.message)
  end

  private

  def feedback_room
    connection[room]
  end

  def connection
    HipChat::Client.new(auth_token, api_version: "v2")
  end
end
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

```ruby
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
        notice: "Feedback was successfully sent."
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

  before do
    allow_any_instance_of(HipChatClient)
      .to receive(:send_feedback)
      .and_return(:true)
  end

  scenario "Visitor creates feedback" do
    visit new_feedback_path

    fill_form :feedback, feedback_attributes
    click_button "Submit"

    open_email(ENV.fetch("FEEDBACK_EMAIL"))

    expect(current_email).to have_subject("Feedback")
    expect(current_email).to be_delivered_from(feedback_attributes[:email])

    expect(current_email).to have_body_text(feedback_attributes[:name])
    expect(current_email).to have_body_text(feedback_attributes[:phone])
    expect(current_email).to have_body_text(feedback_attributes[:email])
    expect(current_email).to have_body_text(feedback_attributes[:message])

    expect(page).to have_content("Feedback was successfully sent.")
  end
end
```
