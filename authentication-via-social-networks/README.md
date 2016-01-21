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
  SocialProfile::PROVIDERS.each do |provider|
    define_method(provider.to_s) do
      begin
        current_user ? connect_social_profile : handle_sign_in
      rescue AuthVerificationPolicy::OauthError => e
        redirect_to root_path, flash: { notice: e.to_s }
      end
    end
  end

  private

  def connect_social_profile
    OauthConnectOrganizer.new(auth, current_user).call
    redirect_to edit_user_registration_path
  end

  def auth
    request.env["omniauth.auth"]
  end

  def handle_sign_in
    user = auth_organizer.new(auth).user
    sign_in_and_redirect user, event: :authentication
  end

  def auth_organizer
    auth_verified? ? VerifiedAuthOrganizer : UnverifiedAuthOrganizer
  end

  def auth_verified?
    AuthVerificationPolicy.new(auth).verified?
  end

  def after_sign_in_path_for(resource)
    if resource.confirmed?
      edit_user_registration_path
    else
      session[:auth_verified?] = auth_verified?
      resource.reset_password(new_password, new_password)
      finish_signup_path(resource)
    end
  end

  def new_password
    @new_password ||= Devise.friendly_token.first(8)
  end
end
```

```ruby
# app/controllers/social_profiles_controller.rb
class SocialProfilesController < ApplicationController
  before_action :authenticate_user!

  expose(:social_profiles) { current_user.social_profiles }
  expose(:social_profile)

  def destroy
    if social_profile.destroy
      flash[:notice] = t "flash.actions.destroy.notice", resource_name: resource_name
    else
      flash[:alert] = t "flash.actions.destroy.alert", resource_name: resource_name
    end
    redirect_to edit_user_registration_url
  end

  private

  def resource_name
    SocialProfile.model_name.human
  end
end
```

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController
  before_action :authenticate_user!, only: :home

  expose(:user, attributes: :user_params)

  def home
  end

  def finish_signup
    return false unless request.patch?

    user.update_attributes(user_params) ? sign_in_user : render_errors
  end

  private

  def user_params
    params.require(:user).permit(%i(full_name email password))
  end

  def sign_in_user
    confirm_user
    sign_in(user, bypass: true)
    redirect_to root_path, notice: "Welcome!"
  end

  def render_errors
    render :finish_signup
  end

  def confirm_user
    if session[:auth_verified?]
      session[:auth_verified?] = nil
      user.update_attribute(:confirmed_at, Time.zone.now)
    else
      user.send_confirmation_instructions
    end
  end
end
```

```ruby
# app/helpers/omniauth_helper.rb
module OmniauthHelper
  def provider_name(provider)
    t "active_record.attributes.social_profile.provider_name.#{provider}"
  end
end
```

```ruby
# app/interactors/connect_social_profile.rb
class ConnectSocialProfile
  attr_reader :user, :auth
  private :user, :auth

  def initialize(user, auth)
    @user = user
    @auth = auth
  end

  def call
    social_profile ? update_social_profile : create_social_profile
  end

  private

  def social_profile
    @social_profile ||= SocialProfile.from_omniauth(auth)
  end

  def update_social_profile
    social_profile.update_attribute(:user, user)
  end

  def create_social_profile
    user.social_profiles.create!(provider: auth.provider, uid: auth.uid)
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
    user = User.new(
      email: auth.info.email,
      full_name: auth.info.name,
      password: new_password,
      password_confirmation: new_password
    )
    user.skip_confirmation_notification! && user.save!
    user
  end

  private

  def new_password
    @new_password ||= Devise.friendly_token.first(8)
  end
end
```

```ruby
# app/interactors/find_user_by_email_service.rb
class FindUserByEmailService
  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def call
    return unless user

    create_social_profile
    user
  end

  private

  def user
    @user ||= User.find_by(email: auth.info.email)
  end

  def create_social_profile
    user.social_profiles.create!(provider: auth.provider, uid: auth.uid)
  end
end
```

```ruby
# app/interactors/oauth_connect_organizer.rb
class OauthConnectOrganizer
  attr_reader :auth, :user
  private :auth, :user

  def initialize(auth, user)
    @auth = auth
    @user = user
  end

  def call
    unless user.confirmed?
      auth_verified? ? process_user_confirmation : fail_oauth
    end

    connect_social_profile
  end

  private

  def auth_verified?
    AuthVerificationPolicy.new(auth).verified?
  end

  def process_user_confirmation
    user.confirm
    user.send_reset_password_instructions
  end

  def fail_oauth
    fail AuthVerificationPolicy::OauthError,
      "Please confirm your account before connecting your #{auth.provider} account."
  end

  def connect_social_profile
    ConnectSocialProfile.new(user, auth).call
  end
end
```

```ruby
# app/interactors/unverified_auth_organizer.rb
class UnverifiedAuthOrganizer
  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def user
    check_user_by_email!

    found_user.send_confirmation_instructions unless found_user.confirmed?
    found_user
  end

  private

  def check_user_by_email!
    user_by_email = User.find_by(email: auth.info.email)
    fail AuthVerificationPolicy::OauthError, "Please, connect your account from profile page." if user_by_email
  end

  def found_user
    user_found_by_uid || new_user
  end

  def user_found_by_uid
    @user_found_by_uid ||= SocialProfile.from_omniauth(auth).try(:user)
  end

  def new_user
    @new_user ||= CreateUserFromAuth.new(auth).call
  end
end
```

```ruby
# app/interactors/verified_auth_organizer.rb
class VerifiedAuthOrganizer
  attr_reader :auth
  private :auth

  def initialize(auth)
    @auth = auth
  end

  def user
    user_found_by_uid || user_found_by_email || new_user
  end

  private

  def user_found_by_uid
    SocialProfile.from_omniauth(auth).try(:user)
  end

  def user_found_by_email
    FindUserByEmailService.new(auth).call
  end

  def new_user
    CreateUserFromAuth.new(auth).call
  end
end
```

```ruby
# app/models/social_profile.rb
class SocialProfile < ActiveRecord::Base
  PROVIDERS = OmniAuth.strategies.map { |s| s.to_s.demodulize.underscore }.last(2)

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
  devise :database_authenticatable, :registerable, :confirmable,
    :recoverable, :rememberable, :trackable, :validatable,
    :omniauthable, omniauth_providers: SocialProfile::PROVIDERS

  has_many :social_profiles, dependent: :destroy
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
    request_verification_for
  rescue NoMethodError
    fail_with_error
  end

  private

  def request_verification_for
    send(auth.provider)
  end

  def fail_with_error
    fail ArgumentError, I18n.t("omniauth.verification.not_implemented", kind: auth.provider)
  end

  def facebook
    auth.info.verified? || auth.extra.raw_info.verified?
  end

  def google_oauth2
    auth.extra.raw_info.email_verified?
  end
end
```

```ruby
# app/views/social_profiles/_list.html.slim
- if social_profiles.any?
  b Successfully authorized via:
  ul.js-social-profiles
    - social_profiles.each do |social_profile|
      li = link_to "#{provider_name(social_profile.provider)} (#{social_profile.uid.truncate(9)}). Unauthorize?",
                   social_profile,
                   data: { confirm: "Are you sure you want to remove this social profile?" },
                   method: :delete,
                   class: "js-unauthorize"

b Add service to sign in with:
ul
  - SocialProfile::PROVIDERS.each do |provider|
    li = link_to provider_name(provider), user_omniauth_authorize_path(provider)
```

```ruby
# app/views/users/finish_signup.html.slim
.row
  h3 Finish Signup

  .medium-6.columns
    = simple_form_for user,
      as: 'user',
      url: finish_signup_path(user),
      html: { method: :patch } do |f|

      .form-inputs
        = f.input :email, required: true, autofocus: true,  placeholder: "Your email here"

        = f.input :password, required: true, hint: '', placeholder: "Your password here"

      .form-actions
        = f.button :submit, "Finish Signup"
```

```ruby
# app/views/users/registrations/edit.html.slim
...
= render "social_profiles/list", social_profiles: current_user.social_profiles
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
# config/locales/models/social_profile.en.yml
en:
  active_record:
    attributes:
      social_profile:
        provider_name:
          google_oauth2: Google
          facebook: Facebook
```

```ruby
# config/locales/oauth.en.yml
en:
  omniauth:
    verification:
      failure: Your %{kind} account can't be used to sign in. Please verify it via profile page.
      not_implemented: Verification checking is not implemented for %{kind}.
```

```ruby
# config/routes.rb
...
devise_for :users, controllers: { omniauth_callbacks: "omniauth_callbacks" }
resources :social_profiles, only: :destroy
match "/users/:id/finish_signup" => "users#finish_signup", via: %i(get patch), as: :finish_signup
...
```

```ruby
# db/migrate/20151208195800_create_social_profiles.rb
class CreateSocialProfiles < ActiveRecord::Migration
  def change
    create_table :social_profiles do |t|
      t.references :user, index: true
      t.string :provider, index: true, null: false, default: ""
      t.string :uid, index: true, null: false, default: ""
    end
  end
end
```

```ruby
# spec/factories/social_profiles.rb
FactoryGirl.define do
  factory :social_profile do
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
  context "oauth confirmed" do
    include_context :stub_omniauth

    before { click_connect_fb }

    context "user confirmed" do
      let!(:user) { create(:user, :confirmed, :from_auth_hashie) }

      scenario "User connects social account" do
        expect(page).to have_connected_account("Facebook")
      end
    end

    context "user not confirmed" do
      let!(:user) { create(:user, :from_auth_hashie) }

      scenario "User have to confirm own account" do
        expect(page).to have_connected_account("Facebook")

        open_email(user.email)

        expect(current_email).to have_subject("Confirmation instructions")
        expect(current_email).to have_body_text(user.full_name)
      end
    end
  end

  context "oauth not confirmed" do
    include_context :stub_not_verified_omniauth

    before { click_connect_fb }

    context "user confirmed" do
      let!(:user) { create(:user, :confirmed, :from_auth_hashie) }

      scenario "User connects social account" do
        expect(page).to have_connected_account("Facebook")
      end
    end

    context "user not confirmed" do
      let!(:user) { create(:user, :from_auth_hashie) }

      scenario "User sees alert" do
        expect(page).to have_text("Please confirm your account before connecting your facebook account.")
        expect(current_path).to eq(root_path)
      end
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
# spec/features/visitor/sign_in_with_social_spec.rb
require "rails_helper"

feature "Sign in with social account" do
  context "when oauth confirmed" do
    include_context :stub_omniauth

    context "when user found by uid" do
      let!(:social_profile) { create(:social_profile, user: user) }

      before { click_sign_in_with_fb }

      context "when user confirmed" do
        let!(:user) { create(:user, :confirmed, :from_auth_hashie) }

        it_behaves_like "success sign in"
      end

      context "when user not confirmed" do
        let!(:user) { create(:user, :from_auth_hashie) }

        it_behaves_like "finishing sign up" do
          let(:name) { user.full_name }
          let(:email) { "mailer@mail.com" }
          let(:password) { "123456qwe" }
        end
      end
    end

    context "when user found by email" do
      context "when user confirmed" do
        let!(:user) { create(:user, :confirmed, :from_auth_hashie) }

        before { click_sign_in_with_fb }

        it_behaves_like "success sign in"
      end

      context "when user not confirmed" do
        let!(:user) { create(:user, :from_auth_hashie) }

        it_behaves_like "finishing sign up" do
          let(:name) { user.full_name }
          let(:email) { "mailer@mail.com" }
          let(:password) { "123456qwe" }
        end
      end
    end

    context "when user not found" do
      it_behaves_like "finishing sign up" do
        let(:name) { "Joe Bloggs" }
        let(:email) { "mailer@mail.com" }
        let(:password) { "123456qwe" }
      end
    end
  end

  context "when oauth not confirmed" do
    include_context :stub_not_verified_omniauth

    context "when user found by uid" do
      let!(:social_profile) { create(:social_profile, user: user) }
      let!(:user) { create(:user, :confirmed) }

      before { click_sign_in_with_fb }

      it_behaves_like "success sign in"
    end

    context "when user found by email" do
      let!(:user) { create(:user, :confirmed, :from_auth_hashie) }

      before { click_sign_in_with_fb }

      scenario "User sees alert message" do
        expect(page).to have_text("Please, connect your account from profile page.")
        expect(current_path).to eq(root_path)
      end
    end

    context "when user not found" do
      it_behaves_like "finishing sign up" do
        let(:name) { "Joe Bloggs" }
        let(:email) { "mailer@mail.com" }
        let(:password) { "123456qwe" }
      end
    end
  end

  def click_sign_in_with_fb
    visit new_user_session_path
    click_link "Sign in with Facebook"
  end
end
```

```ruby
# spec/interactors/connect_social_profile_spec.rb
require "rails_helper"

describe ConnectSocialProfile do
  include_context :auth_hashie

  let(:user_1) { create(:user) }
  let(:user_2) { create(:user) }
  let(:service) { described_class.new(user_2, auth_hashie) }

  subject { service.call }

  context "when social_profile exists" do
    let!(:social_profile) do
      create(:social_profile, uid: auth_hashie.uid, provider: auth_hashie.provider, user: user_1)
    end

    it "updates user record" do
      expect { subject }.to change { social_profile.reload.user }.from(user_1).to(user_2)
    end
  end

  context "when social profile not exists" do
    it "creates related social profile" do
      expect { subject }.to change { user_2.social_profiles.count }.by(1)
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

  it "creates new user from auth hash" do
    expect { subject }.to change { User.count }.by(1)
    expect(sent_emails).to eq(0)
    expect(user.email).to eq(auth_hashie.info.email)
    expect(user.full_name).to eq(auth_hashie.info.name)
  end
end
```

```ruby
# spec/interactors/find_user_by_email_service_spec.rb
require "rails_helper"

describe FindUserByEmailService do
  include_context :auth_hashie

  let(:service) { described_class.new(auth_hashie) }

  subject { service.call }

  context "when user not exists" do
    it { is_expected.to be_nil }
  end

  context "when user exists" do
    let!(:user) { create(:user, :from_auth_hashie) }

    it "creates new social_profile" do
      expect { subject }.to change { user.social_profiles.count }.by(1)
      expect(subject).to eq(user)
    end
  end
end
```

```ruby
# spec/interactors/oauth_connect_organizer_spec.rb
require "rails_helper"

describe OauthConnectOrganizer do
  include_context :auth_hashie

  let(:service) { described_class.new(auth_hashie, user) }

  subject { service.call }

  context "when user confirmed" do
    let(:user) { create(:user, :confirmed) }

    before do
      allow(ConnectSocialProfile).to receive_message_chain(:new, :call)
      service.call
    end

    it "creates social profile" do
      expect(ConnectSocialProfile).to have_received(:new).with(user, auth_hashie)
    end
  end

  context "when user not confirmed" do
    let(:user) { create(:user) }

    context "when auth not verified" do
      include_context :invalid_auth_hashie

      it "raises error" do
        expect { subject }.to raise_error(AuthVerificationPolicy::OauthError)
      end
    end

    context "when auth verified" do
      before do
        allow(user).to receive(:confirm)
        allow(user).to receive(:send_reset_password_instructions)
        service.call
      end

      it "confirms user" do
        expect(user).to have_received(:confirm)
        expect(user).to have_received(:send_reset_password_instructions)
      end
    end
  end
end
```

```ruby
# spec/interactors/unverified_auth_organizer_spec.rb
require "rails_helper"

describe UnverifiedAuthOrganizer do
  include_context :auth_hashie

  let(:service) { described_class.new(auth_hashie) }

  subject { service.user }

  context "when user found by email" do
    before { create(:user, :from_auth_hashie) }

    it "raises error" do
      expect { subject }.to raise_error(AuthVerificationPolicy::OauthError)
    end
  end

  context "when user found by uid" do
    let!(:social_profile) { create(:social_profile, provider: auth_hashie.provider, uid: auth_hashie.uid, user: user) }

    let(:emails) { ActionMailer::Base.deliveries }

    context "when user confirmed" do
      let!(:user) { create(:user, :confirmed, :from_auth_hashie) }

      it "not sends confirmation notification" do
        expect(emails).to be_empty
      end
    end

    context "when user not confirmed" do
      let!(:user) { create(:user, :from_auth_hashie) }
      let(:email) { emails.last }

      it "sends confirmation notification" do
        expect(email.subject).to eq("Confirmation instructions")
        expect(email.to).to eq([user.email])
      end
    end
  end

  context "when user not found" do
    let(:user) { User.last }

    it "creates new user from auth_hashie" do
      expect { subject }.to change { User.count }.by(1)
      expect(subject).to eq(user)
    end
  end
end
```

```ruby
# spec/interactors/verified_auth_organizer_spec.rb
require "rails_helper"

describe VerifiedAuthOrganizer do
  include_context :auth_hashie

  let(:service) { described_class.new(auth_hashie) }

  subject { service.user }

  context "when social profile exists" do
    let!(:social_profile) { create(:social_profile, uid: auth_hashie.uid, provider: auth_hashie.provider) }

    it { is_expected.to eq(social_profile.user) }
  end

  context "when social profile not exists" do
    context "when user exists" do
      let!(:user) { create(:user, :from_auth_hashie) }

      it "creates related social profile" do
        expect { subject }.to change { user.social_profiles.count }.by(1)
        expect(subject).to eq(user)
      end
    end

    context "when user not exists" do
      it "creates new one" do
        expect { subject }.to change { User.count }.by(1)
      end
    end
  end
end
```

```ruby
# spec/models/social_profile_spec.rb
require "rails_helper"

describe SocialProfile do
  subject { create(:social_profile, uid: "abc123") }

  it { is_expected.to belong_to :user }
  it { is_expected.to validate_presence_of :user }
  it { is_expected.to validate_presence_of :uid }
  it { is_expected.to validate_presence_of :provider }
  it { is_expected.to validate_uniqueness_of(:uid).scoped_to(:provider) }

  describe ".from_omniauth" do
    include_context :auth_hashie

    subject { described_class.from_omniauth(auth_hashie) }

    context "when record exists" do
      let!(:social_profile) { create(:social_profile, uid: auth_hashie.uid, provider: auth_hashie.provider) }

      it { is_expected.to eq(social_profile) }
    end

    context "when record not exists" do
      it { is_expected.to be_nil }
    end
  end
end
```

```ruby
# spec/models/user_spec.rb
...
it { is_expected.to have_many(:social_profiles).dependent(:destroy) }
...
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

    context "when provider is not in the case statement" do
      let(:provider) { "another" }

      it "raises Exception" do
        expect { subject }
          .to raise_error(ArgumentError, I18n.t("omniauth.verification.not_implemented", kind: provider))
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
RSpec::Matchers.define :have_connected_account do |social_profile|
  match do
    within ".js-social-profiles" do
      have_text(social_profile)
    end
  end
end
```

```ruby
# spec/support/shared_examples/finishing_sign_up.rb
shared_examples_for "finishing sign up" do
  before do
    visit new_user_session_path
    click_link "Sign in with Facebook"
  end

  scenario "User finishing registration" do
    expect(page).to have_text("Finish Signup")

    fill_in :user_email, with: email
    fill_in :user_password, with: password
    click_button "Finish Signup"

    open_email(email)
    expect(current_email).to have_subject("Confirmation instructions")
    expect(current_email).to have_body_text(name)

    visit_in_email("Confirm my account")
    expect(page).to have_content("Your email address has been successfully confirmed")
    expect(page).to have_text(name)
    expect(page).to have_text(email)
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

shared_context :invalid_auth_hashie do
  let(:auth_hashie) do
    OmniAuth::AuthHash.new(
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
    expect(page).to have_text(user.full_name)
    expect(current_path).to eq(edit_user_registration_path)
    expect(page).to have_connected_account("Facebook")
  end
end
```
