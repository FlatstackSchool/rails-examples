# Authentication via Social Networks

Omniauth schema:
![alt text](omniauth-schema.png "Omniauth schema")

```ruby
# Gemfile
...
gem "omniauth-facebook"
gem "omniauth-google-oauth2"
...
```

```ruby
# app/controllers/omniauth_callbacks_controller.rb
class OmniauthCallbacksController < Devise::OmniauthCallbacksController
  include OmniauthHelper

  Identity::PROVIDERS.each do |provider|
    define_method(provider.to_s) do
      show_verification_notice and return unless auth_verified?

      current_user ? connect_identity : process_sign_in
    end
  end

  private

  def show_verification_notice
    redirect_to root_path, flash: { error: t("omniauth.verification.failure", kind: provider_name(auth.provider)) }
  end

  def auth_verified?
    AuthVerificationPolicy.new(auth).verified?
  end

  def auth
    request.env["omniauth.auth"]
  end

  def connect_identity
    ConnectIdentity.new(current_user, auth).call
    redirect_to edit_user_registration_path
  end

  def process_sign_in
    user = FetchOauthUser.new(auth).call
    sign_in_and_redirect user, event: :authentication
  end
end
```

```ruby
# app/helpers/omniauth_helper.rb
module OmniauthHelper
  def provider_name(provider)
    t "active_record.attributes.identity.provider_name.#{provider}"
  end
end
```

```ruby
# app/controllers/identities_controller.rb
class IdentitiesController < ApplicationController
  before_action :authenticate_user!

  expose(:identities) { current_user.identities }
  expose(:identity)

  def destroy
    if identity.destroy
      flash[:notice] = t "flash.actions.destroy.notice", resource_name: resource_name
    else
      flash[:alert] = t "flash.actions.destroy.alert", resource_name: resource_name
    end
    redirect_to edit_user_registration_url
  end

  private

  def resource_name
    Identity.model_name.human
  end
end
```

```ruby
# app/interactors/connect_identity.rb
class ConnectIdentity
  attr_reader :user, :auth
  private :user, :auth

  def initialize(user, auth)
    @user = user
    @auth = auth
  end

  def call
    update_or_create_identity
    confirm_user
  end

  private

  def update_or_create_identity
    identity.present? ? update_identity : create_identity
  end

  def identity
    @identity ||= Identity.from_omniauth(auth)
  end

  def update_identity
    identity.update_attribute(:user, user)
  end

  def create_identity
    user.identities.create!(provider: auth.provider, uid: auth.uid)
  end

  def confirm_user
    user.confirm if user.email == auth.info.email
  end
end
```

```ruby
# app/interactors/create_user_from_auth.rb
class CreateUserFromAuth
  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def call
    user = User.new(user_params)
    user.skip_confirmation!
    user.save!
    user
  end

  private

  def user_params
    password = Devise.friendly_token.first(8)
    {
      email: auth.info.email,
      full_name: auth.info.name,
      password: password,
      password_confirmation: password
    }
  end
end
```

```ruby
# app/interactors/fetch_oauth_user.rb
class FetchOauthUser
  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def call
    user_found_by_uid || user_found_by_email || new_user
  end

  private

  def user_found_by_uid
    Identity.from_omniauth(auth).try(:user)
  end

  def user_found_by_email
    FindUserByEmail.new(auth).call
  end

  def new_user
    CreateUserFromAuth.new(auth).call
  end
end
```

```ruby
# app/interactors/find_user_by_email.rb
class FindUserByEmail
  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def call
    return unless user

    create_identity
    user.confirm unless user.confirmed?
    user
  end

  private

  def user
    @user ||= User.find_by(email: auth.info.email)
  end

  def create_identity
    user.identities.where(provider: auth.provider, uid: auth.uid).first_or_create!
  end
end
```

```ruby
# app/models/identity.rb
class Identity < ActiveRecord::Base
  PROVIDERS = OmniAuth.strategies.map { |s| s.to_s.demodulize.underscore }.drop(1)

  belongs_to :user

  validates :user, :provider, :uid, presence: true
  validates :uid, uniqueness: { scope: :provider }

  def self.from_omniauth(auth)
    find_by(provider: auth.provider, uid: auth.uid)
  end
end
```

```ruby
# app/models/user.rb
class User < ActiveRecord::Base
  devise ..., :omniauthable, omniauth_providers: Identity::PROVIDERS

  has_many :identities, dependent: :destroy
  ...
end
```

```ruby
# app/policies/auth_verification_policy.rb
class AuthVerificationPolicy
  class OauthError < StandardError
  end

  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def verified?
    send(auth.provider)
  rescue NoMethodError
    fail OauthError, I18n.t("omniauth.verification.not_implemented", kind: auth.provider)
  end

  private

  def facebook
    auth.info.verified? || auth.extra.raw_info.verified?
  end

  def google_oauth2
    auth.extra.raw_info.email_verified?
  end
end
```

```ruby
# app/views/identities/_list.html.slim
- if identities.any?
  b Successfully authorized via:
  ul.js-identities
    - identities.each do |identity|
      li = link_to "#{provider_name(identity.provider)} (#{identity.uid.truncate(9)}). Unauthorize?",
                   identity,
                   data: { confirm: "Are you sure you want to remove this identity?" },
                   method: :delete,
                   class: "js-unauthorize"
b Add service to sign in with:
ul
  - Identity::PROVIDERS.each do |provider|
    li = link_to provider_name(provider), user_omniauth_authorize_path(provider)
```

```ruby
# app/views/users/registrations/edit.html.slim
...
= render "identities/list", identities: current_user.identities
...
```

```ruby
# config/initializers/devise.rb
...
config.omniauth :google_oauth2, ENV["GOOGLE_CLIENT_ID"], ENV["GOOGLE_CLIENT_SECRET"]
config.omniauth :facebook, ENV["FACEBOOK_APP_ID"], ENV["FACEBOOK_APP_SECRET"], info_fields: "email, name, verified"
...
```

```ruby
# config/locales/models/identity.en.yml
en:
  active_record:
    attributes:
      identity:
        provider_name:
          google_oauth2: Google
          facebook: Facebook
```

```ruby
# config/locales/oauth.en.yml
en:
  omniauth:
    verification:
      failure: Please confirm your %{kind} account before continuing.
      not_implemented: Verification checking is not implemented for %{kind}.
```

```ruby
# config/routes.rb
...
devise_for :users, controllers: { omniauth_callbacks: "omniauth_callbacks" }
resources :identities, only: :destroy
...
```

```ruby
# db/migrate/20160531141911_create_identities.rb
class CreateIdentities < ActiveRecord::Migration
  def change
    create_table :identities do |t|
      t.belongs_to :user, index: true
      t.string :provider, index: true, null: false, default: ""
      t.string :uid, index: true, null: false, default: ""
    end
  end
end
```

```ruby
# spec/factories/social_profiles.rb
FactoryGirl.define do
  factory :identity do
    user
    provider "facebook"
    uid "123545"
  end
end
```

```ruby
# spec/factories/users.rb
...
trait :from_auth_hashie do
  email "joe@bloggs.com"
end
```

```ruby
# spec/features/user/connect_social_account_spec.rb
require "rails_helper"

feature "Connect social account" do
  let!(:user) { create(:user, :from_auth_hashie) }

  context "oauth confirmed" do
    include_context :stub_omniauth

    scenario "User connects social account" do
      click_connect_fb
      expect(page).to have_connected_account("Facebook")
    end
  end

  context "oauth not confirmed" do
    include_context :stub_not_verified_omniauth

    scenario "User views alert message" do
      click_connect_fb
      expect(page).to have_text("Please confirm your Facebook account before continuing.")
    end
  end

  def click_connect_fb
    login_as(user, scope: :user)
    visit edit_user_registration_path(user)
    click_link "Facebook"
  end
end
```

```ruby
# spec/features/visitor/sign_in_with_social_account_spec.rb
require "rails_helper"

feature "Sign in with social account" do
  context "when oauth confirmed" do
    include_context :stub_omniauth

    context "when user found by uid" do
      let!(:identity) { create(:identity, user: user) }
      let(:user) { create(:user, :from_auth_hashie) }

      it_behaves_like "success sign in"
    end

    context "when user found by email" do
      let!(:user) { create(:user, :from_auth_hashie) }

      it_behaves_like "success sign in"
    end

    context "when user not found" do
      let(:user) { User.last }

      it_behaves_like "success sign in"
    end
  end

  context "when oauth not confirmed" do
    include_context :stub_not_verified_omniauth

    scenario "Visitor sees alert message" do
      visit new_user_session_path
      click_link "Sign in with Facebook"

      expect(page).to have_text("Please confirm your Facebook account before continuing.")
    end
  end
end
```

```ruby
# spec/interactors/connect_identity_spec.rb
require "rails_helper"

describe ConnectIdentity do
  include_context :auth_hashie

  let(:service) { described_class.new(user, auth_hashie) }

  subject(:connect_social_account) { service.call }

  context "when identity exists" do
    let(:user) { create(:user) }
    let(:user_2) { create(:user) }

    let!(:identity) do
      create(:identity, uid: auth_hashie.uid, provider: auth_hashie.provider, user: user_2)
    end

    it "updates user id" do
      expect { connect_social_account }.to change { identity.reload.user }.from(user_2).to(user)
    end
  end

  context "when identity not exists" do
    let(:user) { create(:user) }

    it "creates related identity" do
      expect { connect_social_account }.to change { user.identities.count }.by(1)
    end
  end

  context "when user email matches with oauth email" do
    let(:user) { create(:user, email: auth_hashie.info.email, confirmed_at: nil) }

    it "confirms user" do
      expect(user.confirmed?).to be_falsey
      connect_social_account
      expect(user.confirmed?).to be_truthy
    end
  end

  context "when user email not matches with oauth email" do
    let(:user) { create(:user, email: "not@matched.email", confirmed_at: nil) }

    it "not confirms user" do
      expect(user.confirmed?).to be_falsey
      connect_social_account
      expect(user.confirmed?).to be_falsey
    end
  end
end
```

```ruby
# spec/interactors/create_user_from_auth_spec.rb
require "rails_helper"

describe CreateUserFromAuth do
  include_context :auth_hashie

  let(:user) { User.last }
  let(:service) { described_class.new(auth_hashie) }
  let(:sent_emails) { ActionMailer::Base.deliveries.count }

  subject { service.call }

  it "creates new confirmed user from auth hash" do
    expect { subject }.to change { User.count }.by(1)
    expect(sent_emails).to eq(0)
    expect(user.email).to eq(auth_hashie.info.email)
    expect(user.full_name).to eq(auth_hashie.info.name)
    expect(user.confirmed?).to be_truthy
  end
end
```

```ruby
# spec/interactors/fetch_oauth_user_spec.rb
require "rails_helper"

describe FetchOauthUser do
  include_context :auth_hashie

  let(:service) { described_class.new(auth_hashie) }

  subject(:fetched_user) { service.call }

  context "when identity exists" do
    let!(:identity) { create(:identity, uid: auth_hashie.uid, provider: auth_hashie.provider) }

    it { is_expected.to eq(identity.user) }
  end

  context "when identity not exists" do
    context "when user exists" do
      let(:user) { build(:user) }

      before do
        allow(FindUserByEmail).to receive_message_chain(:new, :call).and_return(user)
      end

      it "fetches user by email" do
        expect(FindUserByEmail).to receive_message_chain(:new, :call)
        expect(fetched_user).to eq(user)
      end
    end

    context "when user not exists" do
      let(:user) { build(:user) }

      before do
        allow(CreateUserFromAuth).to receive_message_chain(:new, :call).and_return(user)
      end

      it "creates new one" do
        expect(CreateUserFromAuth).to receive_message_chain(:new, :call)
        expect(fetched_user).to eq(user)
      end
    end
  end
end
```

```ruby
# spec/interactors/find_user_by_email_spec.rb
require "rails_helper"

describe FindUserByEmail do
  include_context :auth_hashie

  let(:service) { described_class.new(auth_hashie) }

  subject(:find_user_by_email) { service.call }

  context "when user not exists" do
    it { is_expected.to be_nil }
  end

  context "when user exists" do
    let!(:user) { create(:user, :from_auth_hashie, confirmed_at: nil) }

    it "creates new identity" do
      expect { find_user_by_email }.to change { user.identities.count }.by(1)
      expect(subject).to eq(user)
    end

    it "confirms user" do
      expect(user.confirmed?).to be_falsey
      find_user_by_email
      expect(user.reload.confirmed?).to be_truthy
    end
  end
end
```

```ruby
# spec/policies/auth_verification_policy_spec.rb
require "rails_helper"

describe AuthVerificationPolicy do
  let(:auth) { double(:omniauth, provider: provider) }

  describe ".verified?" do
    subject { described_class.new(auth).verified? }

    context "when provider is Facebook" do
      let(:provider) { "facebook" }

      before do
        allow(auth).to receive_message_chain(:info, :verified?).and_return(true)
        allow(auth).to receive_message_chain(:extra, :raw_info, :verified?).and_return(true)
      end

      it "returns corresponding value" do
        expect(subject).to eq(true)
      end
    end

    context "when provider is Google" do
      let(:provider) { "google_oauth2" }

      before do
        allow(auth).to receive_message_chain(:extra, :raw_info, :email_verified?).and_return(true)
      end

      it "returns corresponding value" do
        expect(subject).to eq(true)
      end
    end

    context "when provider checking is not defined" do
      let(:provider) { "another" }

      it "raises Exception" do
        expect { subject }
          .to raise_error(AuthVerificationPolicy::OauthError, I18n.t("omniauth.verification.not_implemented", kind: provider))
      end
    end
  end
end
```

```ruby
# spec/rails_helper.rb
...
OmniAuth.config.test_mode = true
...
```

```ruby
# spec/support/matchers/have_connected_account.rb
RSpec::Matchers.define :have_connected_account do |identity|
  match do
    within ".js-identities" do
      have_text(identity)
    end
  end
end
```

```ruby
# spec/support/shared_examples/omniauth_stub.rb
require "rails_helper"

shared_context :auth_hashie do
  let(:auth_hashie) do
    OmniAuth::AuthHash.new(
      provider: "facebook",
      uid: "123545",
      info: {
        email: "joe@bloggs.com",
        name: "Joe Bloggs",
        verified: true
      },
      extra: {
        raw_info: {
          email: "joe@bloggs.com",
          name: "Joe Bloggs",
          verified: true,
          email_verified: true
        }
      }
    )
  end
end

shared_context :stub_omniauth do
  background do
    OmniAuth.config.mock_auth[:facebook] = OmniAuth::AuthHash.new(
      provider: "facebook",
      uid: "123545",
      info: {
        email: "joe@bloggs.com",
        name: "Joe Bloggs",
        verified: true
      },
      extra: {
        raw_info: {
          email: "joe@bloggs.com",
          name: "Joe Bloggs",
          verified: true,
          email_verified: true
        }
      }
    )
  end
end

shared_context :stub_not_verified_omniauth do
  background do
    OmniAuth.config.mock_auth[:facebook] = OmniAuth::AuthHash.new(
      provider: "facebook",
      uid: "123545",
      info: {
        email: "joe@bloggs.com",
        name: "Joe Bloggs",
        verified: false
      },
      extra: {
        raw_info: {
          email: "joe@bloggs.com",
          name: "Joe Bloggs",
          verified: false,
          email_verified: false
        }
      }
    )
  end
end
```

```ruby
# spec/support/shared_examples/success_sign_in.rb
shared_examples_for "success sign in" do
  scenario "User signs in" do
    visit new_user_session_path
    click_link "Sign in with Facebook"

    expect(page).to have_text(user.full_name)
    expect(current_path).to eq(root_path)
  end
end
```
---

## To add new provider you have to do:
- Add new provider related omniauth gem into `Gemfile`
- Add new provider into `config/initializers/devise.rb` with specific params and `ENV` vars, e.g.:
```ruby
...
config.omniauth :new_awesome_provider, ENV["NEW_PROVIDER_ID"], ENV["NEW_PROVIDER_SECRET"], {}
...
```
- Add new provider verification rule into `app/policies/auth_verification_policy.rb`, e.g.:
```ruby
...
def new_example_provider
  auth.extra.example_attribute.verified?
end
...
```
- Extend `config/locales/models/social_profile.en.yml` with new provider name
