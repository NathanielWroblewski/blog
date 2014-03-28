---
layout: post
title: Javascript is Ruby
---

Ruby was my first language, and when I was introduced to javascript,
I found the syntax beastly, the patterns confusing, and anonymous functions
baffling (wtf is an anonymous function!?).  Hopefully, this exercise will convince you,
 that ruby and javascript can be looked at similarly.  Note: this exercise assumes
  some familiarity with ruby and js.

Part 1
---

**Challenge**

Simply put, your challenge is to create a hash whose keys can execute methods.
Let me demonstrate with some pseudocode using a method we're all familiar with:
printing.  Note: `p` in ruby both prints and returns the value in question,
which is different behavior than both `puts` and `print` which print the value
with or without a newline and return `nil`).

```rb
# psuedocode
> hash[:key] = p 'hello' # we store the method in the hash
> hash[:key]             # we access the key
"hello"                  # the method is executed
=> "hello"               # the value is returned
```

Obviously, we can't just do `hash[:key] = p 'hello'`, but it gets the point
across of what we're trying to achieve.  Can you think of a solution?

**Solution**

In ruby, there are objects called procs and lambdas, which are similar to
javascripts' anonymous functions.  We could store a proc/lambda in the hash like
so:

```rb
> hash[:hello] = -> { p 'world'}              # store the lambda
> hash[:hello]                                # access it
=> #<Proc:0x007f8531586430@(irb):3 (lambda)>  # ruby returns lambda object
```

This is close.  It's returning the "function", but it isn't calling it like our
challenge would like.

There are a number of ways to accomplish this first challenge, but this is one:

```rb
class SpecialHash < Hash
  def [](key)
    super.call
  end
end
 ```

Here, I've made a special hash object that inherits from the standard hash, so
we get all the goodies and we can still use `super`.  `super` here will call the
regular hash `hash[:key]` and we'll get the returned proc object that we're storing.
Now, we simply `call` the proc to execute the method.  Let's see if it works.

```rb
> hash = SpecialHash.new
> hash[:hello] = -> { p 'world'}
> hash[:hello]
"world"
=> "world"
```

Sweet.  We even chain them for a cool effect:

```rb
> hash['world'] = -> { p 'inception hash' }
> hash[hash[:hello]] # here, the return value from the first key is the second key
"world"
"inception hash"
=> "inception hash"
```

Part 2
---

**Challenge**

Let's ensure that we can still insert data into our hash.  By data, I mean
non-methods.  Right now, our hash explodes:

```rb
> hash[:hello] = 'world'
> hash[:hello]
NoMethodError: undefined method `call' for "world":String
```

Side note: if you used `eval` to solve the first
challenge, you'll get a different error:

```rb
# solution using eval
class SpecialHash < Hash
  def [](key)
    eval super
  end
end

> hash = SpecialHash.new
> hash[:hello] = 'p "world"'
> hash[:hello]
"world"
=> "world"
> hash[:hello] = "world"
> hash[:hello]
NameError: undefined local variable or method `world' for {:key=>"hello"}:SpecialHash
```

**Solution**

Let's simply detect if its a method or not.

```rb
class SpecialHash < Hash
  def [](key)
    super.is_a?(Proc) ? super.call : super
  end
end
```

Now our hash detects if our value is a method or if it's data.  If it's a method,
it calls it/executes it; if it's data, it returns it.

```rb
> hash[:hello] = -> { p 'world' }
> hash[:hello]
"world"
=> "world"
> hash[:hello] = 'world'
> hash[:hello]
=> "world"
```

Before, if we had tried to nest our hashes:

```rb
> hash[:inception_hash] = -> { SpecialHash.new }
> hash[:inception_hash][:hello] = 'world'
> hash[:inception_hash][:hello]
=> nil
```

Nil?  Why isn't it 'world'?  Well, in this case, a new special hash is created
every time the value is accessed; therefore, none of the data store in it will
persist.  The bonus with our new way is that we can now nest our special hashes
and have the data persist.

```rb
> hash[:nested_hash] = SpecialHash.new
> hash[:nested_hash][:hello] = -> { p 'world' }
> hash[:nested_hash][:hello]
"world"
=> "world"
```

Part 3
---

**Challenge**

Let's say the method I'm storing wants to access the hash I'm storing it in and
the key it's going to be assigned to.  How can I give it access to those values?

```rb
# illustrating the problem
> hash[:key] = -> { p hash; p key}
"{:key => #<Proc:0x007f8531586430@(irb):3 (lambda)> }"
# the proc can access the whole hash
":key" # the proc also knows what key it's being bound to
```

**Solution**

```rb
class SpecialHash < Hash
  def [](key)
    super.is_a?(Proc) ? super.call(self, key) : super
  end
end
```

What we've done here is to pass `self`, which is the hash and `key`, which is the
key to the proc when it's being called.  So now, we can write:

```rb
> hash[:hello] = Proc.new { |entire_hash, key| p entire_hash; p key }
> hash[:hello]
{:hello=>#<Proc:0x007f8531586430@(irb):3>} # prints entire hash
:hello  # prints key
```

But, we have a problem.

Part 4
---
**Final Challenge**

Now, our shit breaks if our proc doesn't take arguments :/

```rb
> hash[:hello] = -> { p 'world' }
> hash[:hello]
ArgumentError: wrong number of arguments (2 for 0)
```

**Solution**

We can overcome this by testing the waters, seeing how many arguments our proc
takes, and feeding it what it wants.

```rb
class SpecialHash < Hash
  def [](key)
    return super unless super.is_a?(Proc)
    case super.arity
      when 2 then super.call(self, key)
      when 1 then super.call(self)
      else super.call
    end
  end
end
```

So now, it works in all cases:

```rb
> hash = SpecialHash.new
> hash[:hello] = -> { p 'world'}
> hash[:hello]
"world"
=> "world"
> hash[:hello] = 'world'
> hash[:hello]
=> "world"
> hash[:hello] = lambda{ |entire_hash| p entire_hash }
> hash[:hello]
{:key=>#<Proc:0x007f85338f0948@(irb):75 (lambda)>}
=> {:key=>#<Proc:0x007f85338f0948@(irb):3 (lambda)>}
```

Takeaway
---

The takeaway here is the similarities between javascript and ruby.  In js,
an object is basically a hash, and we can assign functions to that hash:

```js
> var jsObject = {}
> jsObject.hello = function(){ console.log('hello') }
> jsObject.hello()
"hello"
```

Can you see the similarity to this?

```rb
> hash = {}
> hash[:hello] = -> {puts 'hello'}
> hash[:hello].call
"hello"
```

And how these relate:

<table>
  <tr>
    <th>Ruby</th>
    <th>Javascript</th>
  </tr>
  <tr>
    <td><code>-> { puts 'hello' }</code></td>
    <td><code>function() { console.log('hello') }</code></td>
  </tr>
  <tr>
    <td>
      <code>greet = -> {</code><br/>
      <code>&nbsp;&nbsp;puts 'hello'</code><br/>
      <code>}</code>
    </td>
    <td>
      <code>greet = function(){</code><br/>
      <code>&nbsp;&nbsp;console.log('hello')</code><br/>
      <code>}</code>
    </td>
  </tr>
  <tr>
    <td>
      <code>def greet</code><br/>
      <code>&nbsp;&nbsp;puts 'hello'</code><br/>
      <code>end</code>
    </td>
    <td>
      <code>function greet(){</code><br/>
      <code>&nbsp;&nbsp;console.log('hello')</code><br/>
      <code>}</code>
    </td>
  </tr>
</table>

Ruby procs are basically anonymous functions in js.  If you think about it differently,
you could think of ruby objects as being hashes too, hashes of key value pairs
and hashes of functions, but when you call a function on a ruby object/hash, it's
as if it's called implicitly (i.e. you don't need parens on `hash.hello()` you
could just call `hash.hello` and it knows to call the method).

Hopefully this helps somewhat with your transition from rubyland into js, and
hopefully the exercises, while admittedly a wee bit convoluted, were also fun and educational.

Cheers
