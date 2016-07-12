# Notification

In case we would like to to send notifications using multiple channels like Email
and HipChat right after some business action (send feedback for example).

The entry point is `SendFeedback` organizer which will organize interactors for:
create feedback, send notification via Email, and send notification via HipChat.

All notification will be pushed in the background using `ActiveJob`.

```ruby
# app/interactors/send_feedback.rb
class SendFeedback
  include Interactor::Organizer

  organize CreateFeedback, NotifyUserViaEmail, NotifyUserViaHipchat
end

# app/interactors/notify_user_via_email.rb
class NotifyUserViaEmail
  include Interactor

  delegate :feedback, to: :context
  delegate :email, to: feedback.employee

  def call
    ApplicationMailer.feedback_notification(email, message.content_for(:email)).deliver_later
  end

  private

  def message
    FeedbackNotification.new(feedback)
  end
end

# app/interactors/notify_user_via_hipchat.rb
class NotifyUserViaHipchat
  include Interactor

  class Job < ActiveJob::Base
    include Rollbar::ActiveJob

    def perform(email, message)
      HipchatApi.new.private_message(email, message)
    end
  end

  delegate :feedback, to: :context
  delegate :email, to: feedback.employee

  def call
    Job.perform_later(email, message.content_for(:hipchat))
  end

  private

  def message
    FeedbackNotification.new(feedback)
  end
end
```

Content for each notification is generated using `FeedbackNotification` class.

Gem `render_anywhere` used in this example, but could be replaced with native Rails 5
feature `ActionController::Renderer` later.

```ruby
# app/controllers/application_rendering_controller.rb
class ApplicationRenderingController < RenderAnywhere::RenderingController
  def default_url_options
    { host: ENV.fetch("HOST") }
  end
end

# app/models/fedback_notification.rb
class FeedbackNotification
  include RenderAnywhere

  attr_reader :feedback, :rendering_controller
  private :feedback, :rendering_controller

  def initialize(feedback)
    @feedback = feedback
    @rendering_controller ||= ApplicationRenderingController.new
  end

  def content_for(type)
    render(
      template: template_for(type),
      layout: false,
      locals: locals
    )
  end

  private

  def template_for(type)
    "feedback_notification/#{type}",
  end

  def locals
    {
      feedback: feedback.decorate
    }
  end
end

# app/views/feedback_notification/email.html.slim
p
  | You have new feedback from #{feedback.full_name_with_email}.

  p
    blockquote = feedback.text

# app/views/feedback_notification/hipchat.html.slim
p
  | You have new feedback from #{feedback.full_name_with_email}.

br

  p
    em = feedback.text
```

Here are examples of the wrapper for HipChat API and `ApplicationMailer`

```ruby
# app/models/hipchat_api.rb
class HipchatApi
  attr_reader :client
  private :client

  def initialize(client: build_client)
    @client = client
  end

  def private_message(email, message, format = "html")
    client.user(email).send(message, format)
  rescue HipChat::UnknownUser => e
    Rails.logger.error("HipChat: #{e.message}")
  end

  private

  def build_client
    HipChat::Client.new(ENV.fetch("HIPCHAT_TOKEN"), api_version: "v2")
  end
end

# app/mailers/application_mailer.rb
class ApplicationMailer < ActionMailer::Base
  default from: ENV.fetch("MAILER_SENDER")

  def feedback_notification(email, body)
    mail(
      body: body,
      content_type: "text/html",
      to: email,
      subject: "You have new Feedback"
    )
  end
end
```

```ruby
# Gemfile
gem "hipchat"
gem "interactor"
gem "render_anywhere"
gem "sidekiq"
```
