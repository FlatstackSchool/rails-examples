# Feedback form

```ruby
# app/controllers/feedbacks_controller.rb
class FeedbacksController < ApplicationController
  expose(:feedback, attributes: feedback_attributes)

  def new
  end

  def create
    ApplicationMailer.feedback(feedback).deliver_now! if feedback.save
    respond_with(feedback, location: root_path)
  end

  private

  def feedback_attributes
    params.fetch(:feedback, {}).permit(:email, :name, :message, :phone)
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

  def save
    valid?
  end

  def attributes=(attributes)
    attributes.each do |key, value|
      public_send "#{key}=", value
    end
  end
end
```

```ruby
# app/views/feedbacks/new.html.erb
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

# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  def feedback(feedback)
    @feedback = feedback
    mail(subject: "Feedback", from: feedback.email, to: ENV.fetch("FEEDBACK_EMAIL"))
  end
end
```

```ruby
# app/views/application_mailer/feedback.html.erb
p
  | Hello, here is feedback from #{@feedback.name} (#{@feedback.email} #{@feedback.phone})

blockquote
  = @feedback.message
```

```ruby
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
