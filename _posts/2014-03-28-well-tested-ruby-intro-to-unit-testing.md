---
layout: post
title: Well-Tested Ruby - Intro to Unit Testing
---

Test-driven development (TDD) is a style of programming where
the developer writes specs/tests before writing a line of code.
The developer ensures the test is failing, writes the minimal
amount of code to make the test pass, and then refactors (cleans
up the code).  TDD forces the developer to think about the
design of the program and the interfaces before writing any code.  TDD also makes it easier to change or refactor existing code, because
the developer will immediately know if the changes he or she introduced
broke any existing functionality.

The best argument I've
heard for TDD came from Robert Martin, a Java developer,
speaking at Rails Conf '09 (see: [What Killed Smalltalk Could Kill Ruby](https://www.youtube.com/watch?v=YX3iRjKj7C0)).  Robert argued that ruby is nice because it's so easy to write code in, it gives the developer a lot of freedom, too much freedom in fact.  "It's just so easy to make a mess."  It allows developers to hack out solutions with celerity because it's so easy, but the result of such spikes quickly becomes unmanagable, and this same problem is what invariably killed smalltalk, a ruby predecessor.  He offers TDD as a restraint, a discipline that could save ruby from a similar fate.

Really, the benefits of test-driven development (TDD) are
self-evident once you adopt it, but switching to a TDD style
of programming can be daunting.  Hopefully, this post helps make
TDD more approachable and understandable.

Unit Tests
---

The easiest way to get into TDD is writing unit tests.  Unit tests ensure the atomic pieces of code you write behave as you expect them to.  If you're using Rails, it's simple: unit tests test your model methods.  In general, if I'm going to touch the model layer, I make sure I have a spec for it.  It's especially easy to catch if you get in the habit of  proofreading your own pull requests.  Every method, class method or instance method, has a spec.

Let's walk through a few examples.  I'll assume a familiarity with Rails and I'll use the popular [RSpec](https://relishapp.com/rspec) library, but really the principles are the same irrespective of language, and actually the syntax is often very similar (jasmine/rspec).

Our first problem is a simple one: to create an order that can be cancelled.  Let's TDD it.  The first step is to create an order object.

*spec/models/order_spec.rb*

```rb
require 'spec_helper' # loads the test configuration, etc.

describe Order        # I'll explain the describe in a min
```

If I have [guard](https://github.com/guard/guard) running (*hint hint*), I can configure it to have tests run automatically for me when I alter files.  Otherwise, I can run `rspec spec` in my console.  When I do, I get the following error:

```sh
uninitialized constant Order (NameError)
```

Ignoring the contrived nature of this example, this is a real benefit of TDDing.  I know exactly what I need to do next because the errors tell me exactly what to do next.  Additionally, I write the minimal amount of code to get the system working.  In this example, I can make the corresponding order class.  In rails, if it's a model, I also need to set the inheritance chain.

*app/models/order.rb*

```rb
class Order < ActiveRecord::Base

end
```

Now, I run the test, and nothing explodes.

The next step is to define a method on the order class that will cancel the order.  Let's keep it simple, but also TDD it, and let's assume our order has a status attribute which holds a string.

*spec/models/order_spec.rb*

```rb
require 'spec_helper'

describe Order, '#cancel' do
  it 'changes the order status to cancelled'
end
```

Rspec uses two blocks: `describe` blocks and `it` blocks.  They're basically a way
for you to identify the method and model being tested (`describe`), and put into english what your spec should be doing (`it`).  When your test fails, the output will tell you `Order#cancel changes the order status to cancelled` is failing, i.e. it doesn't actually change the order status to cancelled.  You should have one `describe` block for each method on your model, and you can have many `it` blocks depending on how much each method is doing.  In general, each `it` block should only test one thing.  If you had a `Dog` model with sleep, eat, and bark methods, your related spec-file would have three `describe` blocks: one for eat, one for sleep, and one for bark.  If eat could have three possible outcomes depending on the food passed as an argument, then the one eat `describe` block would have three `it` blocks: one to test each possible outcome.

Example:

```rb
class Dog
  def sleep
    p 'Zzz'
  end

  def bark
    p 'woof.'
  end

  def eat(food)
    if food
      p 'om nom nom'
    else
      p '...'
    end
  end
end
```

```rb
require 'spec_helper'

describe Dog, '#sleep' do
  it 'prints some zzzs'
end

describe Dog, '#bark' do
  it 'gives me a woof'
end

# multiple it blocks for multiple outcomes
describe Dog, '#eat' do
  it 'noms the food if there is food'

  it 'is not entertained if it is being teased'
end
```

Back to our `Order` model.  We're cancelling the order, and we have the `describe`
and `it` blocks set up.  Now we need the nougaty center of the `it` block, the actual
test.  Don't fret, let's break it down.  Each test has three parts:

* Preparation
* Execution
* Assertion

In preparation, we prepare everything we need to execute the function.  In execution,
we just call our method.  In assertion, we just check to see if the result is what
we expected.  For our order to be cancelled, we prepare by making an order that is
not already cancelled.  To execute, we'll just call cancel on it.  And to assert,
we'll just check that the status is now cancelled.  Easy.

Typical pattern

```rb
require 'spec_helper'

describe Model, '#method' do
  it 'does something in plain english' do
    # preparation

    # execution

    # assertion
  end
end
```

For our order, something like this:

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

The only line that's really new here is the assert.  Rspec is currently trending
toward `expect` syntax over `should` syntax, so let's break that down.

All `expect` syntax looks like this:

```rb
expect(test_thing).to eq expected_thing
```

The `expect` and `#to` wrap the subject being tested, unless we use a `#to_not`:

```rb
expect(test_thing).to_not eq expected_thing
```

You may also see `be_something` in place of `eq expected_thing` as in:

```rb
expect(test_thing).to be_true
expect(test_thing).to be_even
expect(test_thing).to be_present
```

Basically, `be_` takes a method that you would normally call on the subject.
  <table>
    <tr>
      <th>
        Ruby
      </th>
      <th>
        RSpec
      </th>
    </tr>
    <tr>
      <td>
        <code>2.even?</code>
      </td>
      <td>
        <code>expect(2).to be\_even</code>
      </td>
    </tr>
    <tr>
      <td>
        <code>nil.present?</code>
      </td>
      <td>
        <code>expect(nil).to\_not be\_present</code>
      </td>
    </tr>
    <tr>
      <td>
        <code>nil</code>
      </td>
      <td>
        <code>expect(nil).to be\_false</code>
      </td>
    </tr>
  </table>

The exception here is `be_true`/`be_false` which tests for the truthiness/falsiness of objects (currently).

Let's write the code now.

*app/models/order.rb*

```rb
class Order < ActiveRecord::Base

  def cancel
    update_attributes(state: 'cancelled')
  end
end
```

Hopefully this is an easy starting point for you to ease on into TDD, but I realize production code can be more complicated.  To address this, I'll be sure to talk about stubs and mocks in my next post.
