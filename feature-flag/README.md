# Feature flag

## Configuration

```ruby
# config/initializers/flipper.rb
require "flipper"
require "flipper/adapters/memory"

Feature.configure do |config|
  config.flipper = Flipper.new(Flipper::Adapters::Memory.new)
  config.dev_mode = ENV["FEATURE_DEV_MODE"].present? # all features considered enabled
end
```

## Usage

* In views and controllers: `feature?(:feature)`
* In PORO: `Feature.enabled?(:feature, user)`


## Implementation

```ruby
# Gemfile
gem "flipper"

# app/models/user.rb
class User < ActiveRecord::Base
  def flipper_id
    "User:#{id}"
  end
end

# db/migrate/XXX_add_tester_to_users.rb
class AddTesterFlagToUsers < ActiveRecord::Migration
  def change
    add_column :users, :tester, :boolean, default: false, null: false, index: true
  end
end


# app/models/feature.rb
class Feature
  cattr_accessor :dev_mode
  self.dev_mode = false

  cattr_accessor :features
  self.features = %i()

  cattr_accessor :flipper
  self.flipper = nil

  attr_reader :feature
  private :feature

  def initialize(feature)
    @feature = feature
  end

  def enabled?(user)
    return true if dev_mode?
    flipper[feature].enabled?(user.to_model)
  end

  def dev_mode?
    !! dev_mode
  end

  def self.enabled?(feature, user)
    new(feature).enabled?(user)
  end

  def self.configure
    yield(self)

    Flipper.register(:testers, &:tester?)

    features.each do |feature|
      flipper[feature].enable(flipper.group(:testers))
    end
  end

  def self.reset!
    Flipper.unregister_groups
  end
end

# spec/models/feature_spec.rb
require "factory_girl"

describe Feature do
  let(:tester) { FactoryGirl.build(:tester, id: 1) }
  let(:user) { FactoryGirl.build(:user, id: 2) }

  before do
    described_class.reset!

    described_class.configure do |config|
      config.flipper = Flipper.new(Flipper::Adapters::Memory.new)
      config.features = %i(archived)
      config.dev_mode = false
    end
  end

  it "has cattr accessors" do
    expect(described_class.dev_mode).to be_falsey
    expect(described_class.flipper).to be
    expect(described_class.features).to eql([:archived])
  end

  it "togles feature for testers" do
    expect(described_class.enabled?(:archived, tester)).to be_truthy
  end

  it "toggles feature for decorated testers" do
    expect(described_class.enabled?(:archived, tester.decorate)).to be_truthy
  end

  it "does not toggle feature for other users" do
    expect(described_class.enabled?(:archived, user)).to be_falsey
  end
end

# app/controllers/concerns/featurization.rb
module Featurization
  extend ActiveSupport::Concern

  included do
    helper_method :feature?
  end

  private

  def feature?(feature, user = current_user)
    Feature.enabled?(feature, user)
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Featurization
end
```

