---
layout: post
title: Well-Tested Ruby - Stubs
---

Let's build off our previous [discussion](http://nathanielwroblewski.github.io/2014/03/28/well-tested-ruby-intro-to-unit-testing/)
of cancelling an order.  Currently, `#cancel` will update the `status` attribute
of our `order` object to `'cancelled'`.  Here is the code we ended with:

*app/models/order.rb*

```rb
class Order < ActiveRecord::Base

  def cancel
    update_attributes(state: 'cancelled')
  end
end
```

*spec/models/order_spec.rb*

```rb
require 'spec_helper'

describe Order, '#cancel' do
  it 'changes the order status to cancelled' do
    order = Order.new(status: 'complete')

    order.cancel

    expect(order.status).to eq 'cancelled'
  end
end
```

Before we get started, let's replace our constructor with a factory; we'll use
the popular [factory_girl](https://github.com/thoughtbot/factory_girl) library.

Now our code looks like this:

*spec/models/order_spec.rb*

```rb
require 'spec_helper'

describe Order, '#cancel' do
  it 'changes the order status to cancelled' do
    order = build(:order, status: 'complete') # factory

    order.cancel

    expect(order.status).to eq 'cancelled'
  end
end
```

And we have a factory file where we define our factories:

*spec/factories.rb*

```rb
FactoryGirl.define do
  factory :user
end
```

Factory girl is simple, it allows you to define a factory which is basically a
model, assign default attributes to model objects you plan on using in your tests,
and gives you a simple syntax for creating objects.

<table>
  <tr>
    <th>
      Rails
    </th>
    <th>
      Factory Girl
    </th>
  </tr>
  <tr>
    <td>
      <code>Order.new(status: :complete)</code>
    </td>
    <td>
      <code>build(:order, status: :complete)</code>
    </td>
  </tr>
  <tr>
    <td>
      <code>Order.create(status: :complete)</code>
    </td>
    <td>
      <code>create(:order, status: :complete)</code>
    </td>
  </tr>
</table>

Now, onto business.

Let's introduce another model: inventory units.  Now, we're tasked with `#cancel`
having to restock the inventory units associated with our order.  For
simplicity's sake, let's assume an order can only be for one type of item,
and that our inventory units represent the items being purchased, i.e. an item
being purchased does not consist of components that are represented as inventory units.
Bascially, an order consists of one or more of the same inventory unit.  Let's start with a
unit test on the inventory unit model.

*spec/factories.rb*

```rb
FactoryGirl.define do
  factory :user
  factory :inventory_unit, class: Inventory::Unit
end
```

*spec/models/inventory/unit_spec.rb*

```rb
require 'spec_helper'

describe Inventory::Unit, '#restock' do
  it 'increments the inventory quantity on hand' do
    unit = build(:inventory_unit, on_hand: 10)

    unit.restock(20)

    expect(unit.on_hand).to eq 30
  end
end
```

The tests will drive the code:

*app/models/inventory/unit.rb*

```rb
class Inventory::Unit < ActiveRecord::Base
  def restock(quantity)
    update_attributes(on_hand: on_hand + quantity)
  end
end
```

There may be some setup involved if you want to namespace the inventory unit, and
have it play nice with active record, but that's for another time.  Let's also
check that our quantity is always a positive number.

*spec/models/inventory/unit_spec.rb*

```rb
require 'spec_helper'

describe Inventory::Unit, '#restock' do
  it 'increments the inventory quantity on hand' do
    unit = build(:inventory_unit, on_hand: 10)

    unit.restock(20)

    expect(unit.on_hand).to eq 30
  end

  it 'will not deduct inventory' do # check for positive input
    unit = build(:inventory_unit, on_hand: 10)

    unit.restock(-5)

    expect(unit.on_hand).to eq 10
  end
end
```

Our resulting code:

*app/models/inventory/unit.rb*

```rb
class Inventory::Unit < ActiveRecord::Base
  def restock(quantity)
    update_attributes(on_hand: on_hand + quantity) if quantity > 0
  end
end
```

Great, now our unit tests are in order, but if our order model is going to
communicate to our inventory unit model, they should probably be related.  Because
orders can only be for one inventory unit in this example, and inventory units will
therefore have many orders/purchases, we could have a situation like this:

*app/models/inventory/unit.rb*

```rb
class Inventory::Unit < ActiveRecord::Base
  has_many :orders, class_name: '::Order'
end
```

*app/models/order.rb*

```rb
class Order < ActiveRecord::Base
  belongs_to :inventory_unit, class_name: 'Inventory::Unit'
end
```

But, do we really need to load the relation both ways?  Not really.  Right now,
only our order model needs to communicate to our inventory unit.  So, let's
not load both associations, and let's only make the association one way.  To test it, we can use the popular
[shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers) from thoughtbot.

*spec/models/order_spec.rb*

```rb
require 'spec_helper'

# here '{ }' replaces 'do end', the magic 'subject' replaces
# 'Order.new', and the it 'description' is ommitted
describe Order, 'associations' do
  it { expect(subject).to belong_to(:inventory_units) }
end

describe Order, '#cancel' do
  it 'changes the order status to cancelled' do
    order = build(:order, status: 'complete')

    order.cancel

    expect(order.status).to eq 'cancelled'
  end
end
```

And the code:

*app/models/order.rb*

```rb
class Order < ActiveRecord::Base
  belongs_to :inventory_unit, class_name: 'Inventory::Unit'

  def cancel
    update_attributes(state: 'cancelled')
  end
end
```

Now, we can restock the inventory when `#cancel` is called.  And here is where we
use a stub.  A stub is used to control the effects of a method call.  If we stub
a method `#bark` on a `Dog` object, we can override the real effects of `#bark` and return anything we want.  We can also tell if our `Dog` object ever had `#bark`
called on it.  For example:

```rb
dog = build(:dog)
dog.stub(:bark) # stub bark so nothing happens when it's called
dog.stub(:bark).and_return('quack') # override bark
dog.stub(:bark).and_yield('moo') # yield to a block

dog.bark

expect(dog).to have_received(:bark) # => true
```

With respect to our order model, we can stub the restocking communication sent to the
inventory unit model like so:

*spec/models/order_spec.rb*

```rb
require 'spec_helper'

describe Order, 'associations' do
  it { expect(subject).to belong_to(:inventory_units) }
end

describe Order, '#cancel' do
  it 'changes the order status to cancelled' do
    order = build(:order, status: 'complete')

    order.cancel

    expect(order.status).to eq 'cancelled'
  end

  it 'restocks the inventory associated with the order' do
    order = build(:order, quantity: 5)
    order.inventory_unit.build
    order.inventory_unit.stub(:restock) # le stub

    order.cancel

    expect(order.inventory_unit).to have_received(:restock).with(5)
  end
end
```

Notice how the preparation section of our unit test is getting bigger?  That's no good.
One thing we can do to refactor is pull out shared factories and put them in a `let` block.

The `let` block lets us define a variable and set its value:

```rb
let(:order) { build(:order, status: :complete) }
# order = Order.new(status: :complete)
let(:user) { create(:user, email: 'boom@pop.com'}
# user = User.create(email: 'boom@pop.com')
```

So now, our code looks like this:

*spec/models/order_spec.rb*

```rb
require 'spec_helper'

describe Order, 'associations' do
  it { expect(subject).to belong_to(:inventory_units) }
end

describe Order, '#cancel' do
  let(:order) { build(:order, status: 'complete', quantity: 5) }

  it 'changes the order status to cancelled' do
    order.cancel

    expect(order.status).to eq 'cancelled'
  end

  it 'restocks the inventory associated with the order' do
    order.inventory_unit.build
    order.inventory_unit.stub(:restock)

    order.cancel

    expect(order.inventory_unit).to have_received(:restock).with(5)
  end
end
```

Now, let's drive the code:

```rb
class Order < ActiveRecord::Base
  belongs_to :inventory_unit, class_name: 'Inventory::Unit'

  def cancel
    inventory_unit.restock(quantity)
    update_attributes(state: 'cancelled')
  end
end
```

The reason we write it this way is because we're only concerned about testing
the model, we're not interested in testing the inventory unit.  We'll let the inventory
unit's unit tests ensure that the methods are doing what they should.  Notice we
didn't write code like this:

```rb
class Order < ActiveRecord::Base
  belongs_to :inventory_unit, class_name: 'Inventory::Unit'

  def cancel
    inventory_unit.update_attributes(
      on_hand: inventory_unit.on_hand + quantity
    )
    update_attributes(state: 'cancelled')
  end
end
```

With this code, the order model knows too much about the internals of the inventory
unit model.  By stubbing it, we ensure that the communication between the models
has occurred, but neither model knows too much about the internals of the other.


Hopefully, this elucidates the purpose of stubs as well as providing a reasonable example.  In practice, your model shouldn't know about the internals of other models.
If you're testing a model with stubs, you can ensure communications occur between models and be certain that models aren't aware of each others internals.

Mocks to come.
