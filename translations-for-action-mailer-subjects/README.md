# Translations for Action Mailer E-Mail Subjects
If you don't pass a subject to the mail method, Action Mailer will try to find it in your translations.
The performed lookup will use the pattern ``<mailer_scope>.<action_name>.subject`` to construct the key.


```ruby
# mailers/application_mailer.rb

class ApplicationMailer < ActionMailer::Base
  def send_weekly_report_notifier(manager, projects)
    @manager = manager
    @projects = projects

    mail(to: manager.email)
  end
end

```

```yml
# config/locales/mailers.en.yml

en:
  application_mailer:
    send_weekly_report_notifier:
      subject: He-hey, it is time to report!

```

```slim
# views/application_mailer/send_weekly_report_notifier.html.slim

h1 Hi, #{@manager.full_name}! There is some job for you!

p Please, fill the weekly reports for your projects!

p Project list:
ul
  - @projects.each do |project|
    li = link_to(project)

```


If you want to send parameters to interpolation use the default_i18n_subject method on the mailer.

```ruby
# mailers/application_mailer.rb

class ApplicationMailer < ActionMailer::Base
  def send_weekly_report_notifier(manager, projects)
    @manager = manager
    @projects = projects

    mail(to: manager.email, subject: default_i18n_subject(manager: manager.full_name))
  end
end

```

```yml
# config/locales/mailers.en.yml

en:
  application_mailer:
    send_weekly_report_notifier:
      subject: He-hey %{manager}, it is time to report!

```
