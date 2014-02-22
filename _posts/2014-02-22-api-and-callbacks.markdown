---
layout: post
title:  "Rails Callbacks and APIs"
date:   2014-02-22 12:17:10
categories: ruby rails
---

How to include API calls in Rails callback hooks without blowing up your tests

While integrating an Instagram media consumer into our app, I ran into a unique use case- I needed to call the Instagram API during model creation/modification, but including after_create and after_save callback hooks in the model would make my RSpec tests fail. This was because if I used `FactoryGirl.create` to create that model, it would try to make an unnecessary API call.

Here is the original model:

{% highlight ruby %}
class InstagramAccount < ActiveRecord::Base
  after_create :get_instagram_id
  after_save :sync_following

  ...

  private
  def get_instagram_id
    user = Instagram.user_search(name).first
    self.instagram_id = user["id"]
    self.full_name = user["full_name"]
    self.save
  end

  def sync_following
    if self.import_enabled_changed?
      self.import_enabled ? follow_user : unfollow_user
    end
  end

  def follow_user
    Instagram.follow_user(self.instagram_id) if Rails.env == 'production'
  end

  def unfollow_user
    Instagram.unfollow_user(self.instagram_id) if Rails.env == 'production'
  end
end
{% endhighlight %}

As you can see, it includes an after_create callback to ping the Instagram API to get the user's instagram_id and full_name. It also has an after_save callback to ping the Instagram API to follow or unfollow the user on our configured Instagram account.

This resulted in a complicated model test where I had to instantiated the `InstagramAccount` object, then stub the two callbacks out, before actually saving it to the test database. Being in Rails 2 made it extra ugly because I couldn't use the `#any_instance` RSpec helper.

{% highlight ruby %}
require 'spec_helper'
 
describe InstagramSyncFollowingWorker do
  it 'should update import_enabled for accounts that are not being followed anymore' do
    VCR.use_cassette 'lib/instagram_user_follows_2' do
      account = FactoryGirl.build(:instagram_account)
      account.stub(:sync_following)
      account.stub(:get_instagram_id)
      account.save
      InstagramSyncFollowingWorker.perform
 
      InstagramAccount.find(:all, :conditions => ["import_enabled = ?", true]).count.should == 2
      InstagramAccount.find(:all, :conditions => ["import_enabled = ?", false]).count.should == 1
    end
 
  end
end
{% endhighlight %}

The controller spec was even more convoluted, because I could actually instantiate an object and stub out the instance methods, since the controller handles all of that. So I had to monkey-patch the model, and also un-monkey-patch it so it didn't affect subsequent tests.

{% highlight ruby %}
require 'spec_helper'
 
describe Admin::TeamStreamsController do
  context "instagram accounts" do
    before(:all) do
      class InstagramAccount
        alias_method :sync_following_old, :sync_following
        alias_method :get_instagram_id_old, :get_instagram_id
        def sync_following
        end
        def get_instagram_id
        end
      end
    end
 
    after(:all) do
      class InstagramAccount
        alias_method :sync_following, :sync_following_old
        alias_method :get_instagram_id, :get_instagram_id_old
      end
    end
 
    it "should create an instagram account" do
      ...
    end
  end
end
{% endhighlight %}

This clearly was a code smell, so after some research, I tried re-factoring the callbacks into an Observer class. This resulted in the after_create and after_save hooks being removed from the InstagramAccount model, and moved into InstagramAccountObserver.

{% highlight ruby %}
class InstagramAccountObserver < ActiveRecord::Observer
  def after_save(account)
    if account.import_enabled_changed?
      account.import_enabled ? account.send(:follow_user) : account.send(:unfollow_user)
    end
  end

  def after_create(account)
    user = Instagram.user_search(account.name).first
    account.instagram_id = user["id"]
    account.full_name = user["full_name"]
    account.save
  end
end
{% endhighlight %}

Observers are placed in the `app/observers/` directory, with a naming convention of `#{model_name}Observer` in camel case.

The last step to getting the observer correctly hooked up to observe a model is to add it into your `config/environment.rb` file.
{% highlight ruby %}
  config.active_record.observers = :instagram_account_observer
{% endhighlight %}

Now, for the tests, I added the no-peeping-toms gem by Pat Maddox, which essentially allows you to turn off all observers during RSpec test runs. After adding the gem, it is as simple as adding `ActiveRecord::Observer.disable_observers` into your `spec_helper.rb` file. This then allowed me to remove the `before(:all)` and `after(:all)` hooks in my controller test.

In the InstagramAccount model spec, I wanted to test one callback, but didn't want the other to run. So I enabled observers for that particular test and then stubbed out the one I didn't need:

{% highlight ruby %}
require 'spec_helper'

describe InstagramAccount do
  context "follow/unfollow" do
    before(:each) do
      @account = FactoryGirl.build(:instagram_account)
      InstagramAccountObserver.instance.stub(:after_create => true)
    end

    it 'should send an api call to instagram when an account is disabled' do
      ActiveRecord::Observer.with_observers(:instagram_account_observer) do
        @account.save
        @account.should_receive(:unfollow_user)
        @account.update_attribute(:import_enabled, false)
      end
    end
  end
end
{% endhighlight }


After all this, I started reading up on how, in general, callbacks may be code smells in general - even those in Observers:
* http://adequate.io/culling-the-activerecord-lifecycle

I'm not sure if I could implement the collaborating object per that blog post, but I'm going to continue to mull over it.
