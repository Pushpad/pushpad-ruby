# Pushpad - Web Push Notifications

[![Build Status](https://travis-ci.org/pushpad/pushpad-ruby.svg?branch=master)](https://travis-ci.org/pushpad/pushpad-ruby)
[![Gem Version](https://badge.fury.io/rb/pushpad.svg)](https://badge.fury.io/rb/pushpad)

[Pushpad](https://pushpad.xyz) is a service for sending push notifications from your web app. It supports the **Push API** (Chrome, Firefox, Opera) and **APNs** (Safari).

Features:

- notifications are delivered even when the user is not on your website
- users don't need to install any app or plugin
- you can target specific users or send bulk notifications

Currently push notifications work on the following browsers:

- Chrome (Desktop and Android)
- Firefox (44+)
- Opera (42+)
- Safari

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'pushpad'
```

And then execute:

    $ bundle install

Or install it yourself as:

    $ gem install pushpad

## Getting started

First you need to sign up to Pushpad and create a project there.

Then set your authentication credentials:

```ruby
Pushpad.auth_token = '5374d7dfeffa2eb49965624ba7596a09'
Pushpad.project_id = 123 # set it here or pass it as a param to methods later
```

- `auth_token` can be found in the user account settings. 
- `project_id` can be found in the project settings. If your application uses multiple projects, you can pass the `project_id` as a param to methods (e.g. `notification.deliver_to user, project_id: 123`).

## Collecting user subscriptions to push notifications

Pushpad offers two different products. [Learn more](https://pushpad.xyz/docs)

### Pushpad Pro

Choose Pushpad Pro if you want to use Javascript for a seamless integration. [Read the docs](https://pushpad.xyz/docs/pushpad_pro_getting_started)

If you need to generate the HMAC signature for the `uid` you can use this helper:

```ruby
Pushpad.signature_for current_user.id
```

### Pushpad Express

If you want to use Pushpad Express, add a link to your website to let users subscribe to push notifications: 

```erb
<a href="<%= Pushpad.path %>">Push notifications</a>

<!-- If the user is logged in on your website you should track its user id to target him in the future  -->
<a href="<%= Pushpad.path_for current_user # or current_user_id %>">Push notifications</a>
```

When a user clicks the link is sent to Pushpad, asked to receive push notifications and redirected back to your website.

## Sending push notifications

```ruby
notification = Pushpad::Notification.new({
  body: "Hello world!", # max 120 characters
  title: "Website Name", # optional, defaults to your project name, max 30 characters
  target_url: "http://example.com", # optional, defaults to your project website
  icon_url: "http://example.com/assets/icon.png", # optional, defaults to the project icon
  image_url: "http://example.com/assets/image.png", # optional, an image to display in the notification content
  ttl: 604800, # optional, drop the notification after this number of seconds if a device is offline
  require_interaction: true, # optional, prevent Chrome on desktop from automatically closing the notification after a few seconds
  custom_data: "123", # optional, a string that is passed as an argument to action button callbacks
  # optional, add some action buttons to the notification
  # see https://pushpad.xyz/docs/action_buttons
  actions: [
    {
      title: "My Button 1", # max length is 20 characters
      target_url: "http://example.com/button-link", # optional
      icon: "http://example.com/assets/button-icon.png", # optional
      action: "myActionName" # optional
    }
  ],
  starred: true, # optional, bookmark the notification in the Pushpad dashboard (e.g. to highlight manual notifications)
  # optional, use this option only if you need to create scheduled notifications (max 5 days)
  # see https://pushpad.xyz/docs/schedule_notifications
  send_at: Time.utc(2016, 7, 25, 10, 9)
})

# deliver to a user
notification.deliver_to user # or user_id

# deliver to a group of users
notification.deliver_to users # or user_ids

# deliver to some users only if they have a given preference
# e.g. only "users" who have a interested in "events" will be reached
notification.deliver_to users, tags: ['events']

# deliver to segments
# e.g. any subscriber that has the tag "segment1" OR "segment2"
notification.broadcast tags: ['segment1', 'segment2']

# you can use boolean expressions 
# they must be in the disjunctive normal form (without parenthesis)
notification.broadcast tags: ['zip_code:28865 && !optout:local_events || friend_of:Organizer123']
notification.deliver_to users, tags: ['tag1 && tag2', 'tag3'] # equal to 'tag1 && tag2 || tag3'

# deliver to everyone
notification.broadcast
```

You can set the default values for most fields in the project settings. See also [the docs](https://pushpad.xyz/docs/rest_api#notifications_api_docs) for more information about notification fields.

If you try to send a notification to a user ID, but that user is not subscribed, that ID is simply ignored.

The methods above return an hash:

- `"id"` is the id of the notification on Pushpad
- `"scheduled"` is the estimated reach of the notification (i.e. the number of devices to which the notification will be sent, which can be different from the number of users, since a user may receive notifications on multiple devices)
- `"uids"` (`deliver_to` only) are the user IDs that will be actually reached by the notification because they are subscribed to your notifications. For example if you send a notification to `['uid1', 'uid2', 'uid3']`, but only `'uid1'` is subscribed, you will get `['uid1']` in response. Note that if a user has unsubscribed after the last notification sent to him, he may still be reported for one time as subscribed (this is due to [the way](http://blog.pushpad.xyz/2016/05/the-push-api-and-its-wild-unsubscription-mechanism/) the W3C Push API works).
- `"send_at"` is present only for scheduled notifications. The fields `"scheduled"` and `"uids"` are not available in this case.

The `id` and `scheduled_count` attribute are also stored on the notification object:

```ruby

notification.deliver_to user

notification.id # => 1000
notification.scheduled_count # => 5
```

## Getting push notification data

You can retrieve data for past notifications:

```ruby
notification = Pushpad::Notification.find(42)

# get basic attributes
notification.id # => 42
notification.title # => "Foo Bar",
notification.body # => "Lorem ipsum dolor sit amet, consectetur adipiscing elit.",
notification.target_url # => "http://example.com",
notification.ttl # => 604800,
notification.require_interaction # => false,
notification.icon_url # => "http://example.com/assets/icon.png",

# `created_at` is a `Time` instance
notification.created_at.utc.to_s # => "2016-07-06 10:09:14 UTC",

# get statistics
notification.scheduled_count # => 1
notification.successfully_sent_count # => 4,
notification.opened_count # => 2
```

Or for mutliple notifications of a project at once:

```ruby
notifications = Pushpad::Notification.find_all(project_id: 5)

# same attributes as for single notification in example above
notifications[0].id # => 42
notifications[0].title # => "Foo Bar",
```

If `Pushpad.project_id` is defined, the `project_id` option can be
omitted.

The REST API paginates the result set. You can pass a `page` parameter
to get the full list in multiple requests.

```ruby
notifications = Pushpad::Notification.find_all(project_id: 5, page: 2)
```

## Getting subscription count

You can retrieve the number of subscriptions for a given project,
optionally filtered by `tags` or `uids`:

```ruby
Pushpad::Subscription.count(project_id: 5) # => 100
Pushpad::Subscription.count(project_id: 5, uids: ['user1']) # => 2
Pushpad::Subscription.count(project_id: 5, tags: ['sports']) # => 10
Pushpad::Subscription.count(project_id: 5, tags: 'sports && travel') # => 5
Pushpad::Subscription.count(project_id: 5, uids: ['user1'], tags: 'sports && travel') # => 1
```

If `Pushpad.project_id` is defined, the `project_id` option can be
omitted.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
