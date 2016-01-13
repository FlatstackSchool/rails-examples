# Repository pattern

```ruby
# app/repositories/users_repository.rb
class UsersRepository
  def find(id, attributes: nil)
    users.only(attributes).where(id: id).first
  end

  def find_by_email(email, attributes: nil)
    users.only(attributes).where(email: email).first
  end

  def all(attributes: nil)
    users.only(attributes)
  end

  def update(user, attrs = {})
    user.update(attrs)
  end

  private

  def users
    @users ||= User.all
  end
end
```

```ruby
# spec/factories/users.rb
FactoryGirl.define do
  factory :user do
    email
    full_name { Faker::Name.name }
    password "123456"
    password_confirmation { password }
    confirmed_at 1.hour.ago
  end

  trait :not_confirmed do
    confirmed_at nil

    after(:create) do |user|
      UsersRepository.new.update(user, confirmation_sent_at: 3.days.ago)
    end
  end
end
```

```ruby
# spec/features/sign_up.rb
feature "Sign Up" do
  let(:registered_user) { UsersRepository.new.find_by_email(user_attributes[:email]) }
end
```
