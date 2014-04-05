---
layout: post
title: Creating Pipelines with State Machine
---

[State machine](https://github.com/pluginaweek/state_machine) is a gem that
let's you assign state to your Rails models, easily create transitions between states,
and create callbacks that are triggered during transitions.  It can make Rails
programming feel a little javascripty, and it can be great for making 'pipelines'.

To illustrate what I mean, imagine having an `Order` model on an e-commerce site.
Our order model could have states like 'cart', 'complete', and 'cancelled'.  We
could implement transitions like `#cancel`, which would allow the order to move
from 'complete' to 'cancelled', but not from 'cart' to 'cancelled' (unless we wanted it
to).  We could also set callbacks that could restock inventory of purchased items
when an order was transitioned from 'complete' to 'cancelled', or anything else for that matter.

Let's dive in to state machine and create an email marketing pipeline.
For this example, we'll assume that we have a `User` model and that our user model
has an attribute which we'll call `email_list`, that holds a string designating
where they are in the email marketing pipeline.  I may also assume a few relations
with a `Subscription` model or `Order` model just to keep things moving.

An Email Marketing Pipeline
---

In designing our email marketing pipeline, we want the following to occur:

When a user signs up at our site, they enter the pipeline.  After 24 hours, if
they've made a purchase we'll segment them based on what they've purchased, otherwise
we'll move them to a 't0_no_action' bucket, where t0 represents an arbitrary time frame they are in.  As for segmenting them, if they
purchased our subscription service, we'll place them in the 'subscriber' email list,
otherwise we'll bucket them in the 'VIP' email list.  After another 48 hours, we'll check on our user again.  If they hadn't purchased before, but have in the meantime,
we'll rebucket them, again segmented on what they've purchased.  Next, we'll check
to see if they added something to their cart.  If they did, we'll bucket them in
the 'added_to_cart' email list.  Otherwise, if they indicated some level of
interest in our subscription service, but didn't complete the sign up, we'll place
them in the 'interested' email list.  Finally, if they didn't do any of the above, we'll move them to the 't1_no_action' email list.  In our last step, we'll check in on our user again, this time another 48 hours later.  If our user has purchased, we'll segment them based on their purchase just as before.  If they still have something in their cart, they'll be added to the 'still_shoppin' email list.  If they still haven't subscribed, we'll assign them to the 'discounted_subscription' email list.  If they hadn't taken any action before and still haven't, we'll put
them on an email list called 't2_no_action'.  After that, we'll continually check back in every 48 hours to see if they've purchased, so we can rebucket them and segment them if need be.  I've illustrated the process below where
words preceeded by a colon indicate an email list and words not preceeded by a colon
indicate an action:

```rb
#                      ### Fantasy Marketing Pipeline ###

#        :unbucketed                                                 Day 0
#         /         \
#       *purchase    no-action                                       Day 1
#       /       \             \
# :subscriber   :vip       :t0_no_action
#                         /   |    |     \
#                 *purchase cart interest no-action                  Day 3
#                           /      |              \
#              :added_to_cart  :interested     :t1_no_action
#              / \              /      \         /    \
#     *purchase  no-action no-action    *purchase   no-action        Day 5
#                 |            |                        |
#      :still_shoppin    :discounted_subscription  :t2_no_action
```

This rather complex pipeline is easily modeled with state machine.  The first step is
to tell state machine what field to use on your model.  In this case, our user model
has an 'email_list' attribute.  We could add this directly to the model, but adding all the
email pipeline logic to the user model could really fatten up our user model, which is
probably fat enough already.  Because everything we're doing is going to relate to
email marketing, let's pull all the functionality out into a concern.  Now, we'll start with this:

*app/models/concerns/email_marketable.rb*

```rb
require 'active_support/concern'

module EmailMarketable
  extend ActiveSupport::Concern

  included do
    state_machine :email_list, initial: :unbucketed # set a default email list
  end
end
```

Let's include the concern in our user model, so the user receives the functionality.

*app/models/user.rb*

```rb
class User < ActiveRecord::Base
  include EmailMarketable
end
```

Now, our user will behave as if t\he following were true:

```rb
class User < ActiveRecord::Base
  state_machine :email_list, inital: :unbucketed
end
```

The next step, is to tell state machine what states we're going to be using.  In
this case, the states are going to correspond to email lists.  We can set that up
easily enough:

*app/models/concerns/email_marketable.rb*

```rb
require 'active_support/concern'

module EmailMarketable
  extend ActiveSupport::Concern

  included do
    state_machine :email_list, initial: :unbucketed do
      state :unbucketed,             # adding possible states
            :subscriber,
            :vip,
            :t0_no_action,
            :t1_no_action,
            :t2_no_action,
            :added_to_cart,
            :interested,
            :still_shoppin,
            :discounted_subscription
    end
  end
end
```

At this point, a user can have one of several states: 'unbucketed', 'subscriber', 'vip', 't0_no_action', 't1_no_action', 't2_no_action', 'added_to_cart', 'interested', 'still_shoppin', or 'discounted_subscription'.  But, we have not yet declared a way to transition between states.  Let's add one.

*app/models/concerns/email_marketable.rb*

```rb
require 'active_support/concern'

module EmailMarketable
  extend ActiveSupport::Concern

  included do
    state_machine :email_list, initial: :unbucketed do
      state :unbucketed,
            :subscriber,
            :vip,
            :t0_no_action,
            :t1_no_action,
            :t2_no_action,
            :added_to_cart,
            :interested,
            :still_shoppin,
            :discounted_subscription

      event :took_no_action do      # define a transition
        transition interested:      :discounted_subscription
        transition added_to_cart:   :still_shoppin
        transition t1_no_action:    :t2_no_action
        transition t0_no_action:    :t1_no_action
        transition unbucketed:      :t0_no_action
      end
    end
  end
end
```

Here we've defined a transition from one set of states to another.  If we had a
user, and we called `#took_no_action` on that user, his state will change to the
corresponding state.  For example, a user with a current state of `t0_no_action` will
change state to `t1_no_action` when we call `user.took_no_action`.

```rb
> user = User.create(email_list: :added_to_cart)
> user.took_no_action
> user.email_list
=> :still_shoppin
```

We have a few special cases not covered above, like when a user transitions to
'added_to_cart' from 't0_no_action', or when a user transitions to 'interested' from 't0_no_action'.  Let's make special cases for each of those.

*app/models/concerns/email_marketable.rb*

```rb
require 'active_support/concern'

module EmailMarketable
  extend ActiveSupport::Concern

  included do
    state_machine :email_list, initial: :unbucketed do
      state :unbucketed,
            :subscriber,
            :vip,
            :t0_no_action,
            :t1_no_action,
            :t2_no_action,
            :added_to_cart,
            :interested,
            :still_shoppin,
            :discounted_subscription

      event :took_no_action do
        transition interested:      :discounted_subscription
        transition added_to_cart:   :still_shoppin
        transition t1_no_action:    :t2_no_action
        transition t0_no_action:    :t1_no_action
        transition unbucketed:      :t0_no_action
      end

      event :added_item_to_cart do               # handle special cases
        transition t0_no_action: :added_to_cart
      end

      event :interested_in_subscribing do        # handle special cases
        transition t0_no_action: :interested
      end
    end
  end
end
```

That handles all cases except for when we want to segment users that have purchased.  Thankfully,
state machine allows us to add conditionals in transitions.  Let's whip something up for detecting
which segment the user belongs in.

*app/models/concerns/email_marketable.rb*

```rb
require 'active_support/concern'

module EmailMarketable
  extend ActiveSupport::Concern

  included do
    state_machine :email_list, initial: :unbucketed do
      state :unbucketed,
            :subscriber,
            :vip,
            :t0_no_action,
            :t1_no_action,
            :t2_no_action,
            :added_to_cart,
            :interested,
            :still_shoppin,
            :discounted_subscription

      event :took_no_action do
        transition interested:      :discounted_subscription
        transition added_to_cart:   :still_shoppin
        transition t1_no_action:    :t2_no_action
        transition t0_no_action:    :t1_no_action
        transition unbucketed:      :t0_no_action
      end

      event :added_item_to_cart do
        transition t0_no_action: :added_to_cart
      end

      event :interested_in_subscribing do
        transition t0_no_action: :interested
      end

      event :made_a_purchase do    # conditional transitions
        transition all => :subscriber, if: :purchased_subscription?
        transition all => :vip,    unless: :purchased_subscription?
      end
    end
  end

  def purchased_subscription?      # method to be used in conditional above
    subscriptions.any?             # assumes a relationship on user model
  end
end
```
That wraps up all of our state and transitions, now let's add a few callbacks.
Ideally, we will limit the number of callbacks to limit the complexity of this
state machine, but a few won't hurt.

*app/models/concerns/email_marketable.rb*

```rb
require 'active_support/concern'

module EmailMarketable
  extend ActiveSupport::Concern

  included do
    state_machine :email_list, initial: :unbucketed do
      state :unbucketed,
            :subscriber,
            :vip,
            :t0_no_action,
            :t1_no_action,
            :t2_no_action,
            :added_to_cart,
            :interested,
            :still_shoppin,
            :discounted_subscription

      event :took_no_action do
        transition interested:      :discounted_subscription
        transition added_to_cart:   :still_shoppin
        transition t1_no_action:    :t2_no_action
        transition t0_no_action:    :t1_no_action
        transition unbucketed:      :t0_no_action
      end

      event :added_item_to_cart do
        transition t0_no_action: :added_to_cart
      end

      event :interested_in_subscribing do
        transition t0_no_action: :interested
      end

      event :made_a_purchase do
        transition all => :subscriber, if: :purchased_subscription?
        transition all => :vip,    unless: :purchased_subscription?
      end

      before_transition any => any, do: :email_list_unsubscribe # callback
      after_transition  any => any, do: :email_list_subscribe   # callback
    end
  end

  def purchased_subscription?
    subscriptions.any?
  end

  def email_list_subscribe         # callback triggers this method
    Resque.enqueue(SubscribeUserToList, id, email_list) # starts background job
  end

  def email_list_unsubscribe       # callback triggers this method
    Resque.enqueue(UnsubscribeUserFromList, id, email_list) # starts bg job
  end
end
```

Here, we're not too concerned with what our callbacks are doing, so I obscure
the functionality by putting them in a background job, which would presumably
make some API call to some third-party service.  Whatever it does, it's not important.  What is important is understanding that the `email_list_unsubscribe` call back
will fire before any transition from any one state to any another.  Same with the
`email_list_subscribe` callback.  We could also specify a callback to be called when
we transition between a specific state to any other specific state as well, but this suffices for this example.

Our state machine is ready to go, let's whip up a quick rake task to demonstrate
how transitions could be called on our user object.

*lib/tasks/assign_email_list.rake*

```rb
desc "Updates a user's email list"

task assign_email_list: :environment do
  updateable_users = User.where.not(email_list: [:subscriber, :vip])
  updateable_users.each do |user|
    if user.orders.any?
      user.made_a_purchase
    elsif user.subscriptions.any?(&:incomplete?)
      user.interested_in_subscribing
    elsif user.cart.any?
      user.added_item_to_cart
    else
      user.took_no_action
    end
  end
end
```

This scheduled task simply runs every day or so, pulls users that haven't reached
the highest priority email list (users that could be rebucketed into another list), checks which action the user should receive base on our criteria, and simply calls the state machine transition on the user, letting state machine handle all the rest.

The more I work with the state machine pattern, the more I enjoy it.  I find that
it can greatly simplify my backend rails code and that it is excellent for creating pipelines.  The next time you need to create some sort of pipeline, give state machine a shot, and let me know what you think of it.
