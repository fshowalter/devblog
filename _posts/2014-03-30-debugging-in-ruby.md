---
layout: post
title: Debugging in Ruby
---

Debugging Ruby is easy.[1]

## Step 1: Reproduce the Problem
First, write a quick test that reproduces your problem. It's a lot easier to debug by running a test than potentially bouncing your development server and reloading your browser.

## Step 2: Get out your Toolbox
Now that we've got a working test, lets get out our debugging toolbox. Don't worry, it's really light. To start with we'll only focus on four methods (one of which we probably won't even need).

### Tool 1: puts
`puts` is your friend. Use him liberally. Want to make sure a method/condition is getting called? Add `puts 'called'` and run our test. See 'called' in the output? That'll tell you whether or not your code made it that far.

You can also use `puts` to check the state of variables. Consider the following:

```ruby
def add_to_stack(item)
	@stack << item
end
```

Suppose we were curious about what `@stack` was before adding `item`? We could add `puts @stack.inspect` before appending the item to find out.

### Tool 2: raise
`raise` is an alternative to `puts` that's useful if your program is capturing standard out (common in console app tests). Just make sure you disable any global rescue handlers in your program.

### Tool 3: caller
`caller` lets us view the current call stack. When combined with `puts` we get a nice stack trace. Useful if you know what's screwing up your program but you can't figure out how that bit of logic is getting called.

### Tool 4: method
`method` lets us reflect on a method and get the meta info.
Consider the following:

```ruby
class Movie
	has_many :actors
end
```

Let suppose we were having problems with our `has_many` association and we wanted to get a look at the source behind it. Calling:

```
Movie.method(:has_many).source_location
```

Would give us something like:

["/vendor/bundle/gems/activerecord-4.0.2/lib/active_record/associations.rb", 1185]

Nice, huh? No more wondering 'Where is this method defined?'

## Step 3: Debug
Armed with these method's we're ready to deep-dive into the code to find the cause of our problem. Since Ruby is interpreted instead of compiled, you'll always have access to the source, so it's just a matter of digging until you find the offending code. Remember, if you've know where the offending code is, but can't figure outside how it's getting called, `caller` is your friend.

## Step 4: Make things easier with PRY.
Pry is powerful tool that lets you open up an IRB session bound to a running program. Put another way, instead of adding `puts` statements, running your tests, then checking the output, you "pry" open an IRB session with your program's current state, in which you can utilize any of the above tools and get immediate feedback. See (http://pryrepl.org)[Pry's site] for more info.

1. Actually debugging _in_ Ruby is easy. Debugging a problem with the actually MRI interpreter probably means going down to the C code which is no fun. Switch to a different Ruby version/patch number instead.
