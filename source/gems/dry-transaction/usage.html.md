---
title: Usage
layout: gem-single
---

### Container

All you need to use dry-transaction is a container to hold your application’s operations. Each operation must respond to `#call(input)`.

The operations will be resolved from the container via `#[]`. For our examples, we’ll use a plain hash:

```ruby
container = {
  process:  -> input { {name: input["name"], email: input["email"]} },
  validate: -> input { input[:email].nil? ? raise(ValidationFailure, "not valid") : input },
  persist:  -> input { DB << input and true }
}
```

For larger apps, you may like to consider something like [dry-container](https://github.com/dryrb/dry-container).

### Defining a transaction

Define a transaction to bring your opererations together:

```ruby
save_user = Dry.Transaction(container: container) do
  map :process
  try :validate, catch: ValidationFailure
  tee :persist
end
```

Operations are formed into steps using _step adapters._ Step adapters wrap the output of your operations to make them easy to integrate into a transaction. The following adapters are available:

* `step` – the operation already returns an `Either` object (`Right(output)` for success and `Left(output)` for failure), and needs no special handling.
* `map` – any output is considered successful and returned as `Right(output)`
* `try` – the operation may raise an exception in an error case. This is caught and returned as `Left(exception)`. The output is otherwise returned as `Right(output)`.
* `tee` – the operation interacts with some external system and has no meaningful output. The original input is passed through and returned as `Right(input)`.

### Calling a transaction

Calling a transaction will run its operations in their specified order, with the output of each operation becoming the input for the next.

```ruby
DB = []

save_user.call("name" => "Jane", "email" => "jane@doe.com")
# => Right({:name=>"Jane", :email=>"jane@doe.com"})

DB
# => [{:name=>"Jane", :email=>"jane@doe.com"}]
```

Each transaction returns a result value wrapped in a `Left` or `Right` object (based on the output of its final step). You can handle these results (including errors arising from particular steps) with a match block:

```ruby
save_user.call(name: "Jane", email: "jane@doe.com") do |m|
  m.success do |value|
    puts "Succeeded!"
  end

  m.failure :validate do |error|
    # In a more realistic example, you’d loop through a list of messages in `errors`.
    puts "Please provide an email address."
  end

  m.failure do |error|
    puts "Couldn’t save this user."
  end
end
```

### Passing additional step arguments

Additional arguments for step operations can be passed at the time of calling your transaction. Provide these arguments as an array, and they’ll be [splatted](https://endofline.wordpress.com/2011/01/21/the-strange-ruby-splat/) into the front of the operation’s arguments. This means that transactions can effectively support operations with any sort of `#call(*args, input)` interface.

```ruby
DB = []

container = {
  process:  -> input { {name: input["name"], email: input["email"]} },
  validate: -> allowed, input { input[:email].include?(allowed) ? raise(ValidationFailure, "not allowed") : input },
  persist:  -> input { DB << input and true }
}

save_user = Dry.Transaction(container: container) do
  map :process
  try :validate, catch: ValidationFailure
  tee :persist
end

input = {"name" => "Jane", "email" => "jane@doe.com"}
save_user.call(input, validate: ["doe.com"])
# => Right({:name=>"Jane", :email=>"jane@doe.com"})

save_user.call(input, validate: ["smith.com"])
# => Left("not allowed")
```

### Subscribing to step notifications

As well as pattern matching on the final transaction result, you can subscribe to individual steps and trigger specific behaviour based on their success or failure:

```ruby
NOTIFICATIONS = []

module UserPersistListener
  extend self

  def persist_success(user)
    NOTIFICATIONS << "#{user[:email]} persisted"
  end

  def persist_failure(user)
    NOTIFICATIONS << "#{user[:email]} failed to persist"
  end
end


input = {"name" => "Jane", "email" => "jane@doe.com"}

save_user.subscribe(persist: UserPersistListener)
save_user.call(input, validate: ["doe.com"])

NOTIFICATIONS
# => ["jane@doe.com persisted"]
```

This pub/sub mechanism is provided by the [Wisper](https://github.com/krisleech/wisper) gem. You can subscribe to specific steps using the `#subscribe(step_name: listener)` API, or subscribe to all steps via `#subscribe(listener)`.

### Extending transactions

You can extend existing transactions by inserting or removing steps. See the [API docs](http://www.rubydoc.info/github/dry-rb/dry-transaction/Dry/Transaction/Sequence) for more information.

### Working with a larger container

In practice, your container won’t be a trivial collection of generically named operations. You can keep your transaction step names simple by using the `with:` option to provide the identifiers for the operations within your container:

```ruby
save_user = Dry.Transaction(container: large_whole_app_container) do
  map :process, with: "attributes.user"
  try :validate, with: "validations.user", catch: ValidationFailure
  tee :persist, with: "persistance.commands.update_user"
end
```
