---
layout: post
title: Private Class Methods in YARD
---

doc·u·men·ta·tion:
_noun_

1. something developers don't want to write.
2. something that those same developers will whine about if it's not present in a third-party library they're using.

Lets get two things out of the way right off:

1. Documentation is a pain. It requires maintaining two copies of your domain logic, one in the documentation and another in the code. Keeping these in sync is difficult, made more so by the inability to rely on automated tests or code analysis to warn you when the two have diverged.

2. Self-documenting code is a great ideal that we should all be working towards, but it's not here yet. No matter how well-named your method and its arguments are, it can't tell me what kind of exceptions it may throw, provide an example of how to call it, or all the various options it could take. Someday, perhaps, but we're coding in the here and now.

That said, there are tools to make documentation less of a chore. Right now, one of the best is [YARD](http://yardoc.org/). If you've browsed the [documentation for Capybara](http://rubydoc.info/github/jnicklas/capybara/master/frames/file/README.md), or [FriendlyId](http://norman.github.io/friendly_id/file.Guide.html) you've seen YARD in action.

Let's say we have this code:

```ruby
class Blog
  def add_post(post)
    posts.add(post)
    post
  end

  private

  def posts
    @posts ||= []
  end
end
```

Yard would give you something akin to this:
`## Class:Blog ##`
`### Instance Method Summary ###`
`- (Object) add_post(post)`

Not exactly ideal in terms of documentation, but certainly better than nothing, especially for 3rd party consumers of your libraries.

Now add some comments on your public methods documenting the parameters, return value, exceptions raised, and maybe an usage example or two.

```ruby
class Blog
  #
  # Responsible for adding a post.
  #
  # @param post [Post] The post to add.
  #
  #   blog.add_post(post)
  #
  # @return [Post] The added post.
  def add_post(post)
    posts.add(post)
    post
  end

  private

  def posts
    @posts ||= []
  end
end
```

Re-run YARD and you'll get something like:

Yard would give you something akin to this:

{::options parse_block_html="true" /}
<div class="message" markdown="block">
**Class:Blog**

**Instance Method Details**

`- (Post) add_post(post)`

Responsible for adding a post.

<pre>
    blog.add_post(post)
</pre>

#### Parameters ####
`post(Post)` -- The post to add.
</div>

And now you're in business.

One of YARD's strengths is its ability to automatically exclude private (and optionally protected) methods, thus easing the documentation burden by reducing it to public entry points.

Except currently YARD doesn't recognize Ruby's [`private_class_method`](http://ruby-doc.org/core-2.1.0/Module.html#method-i-private_class_method), which means that all class methods are always included. This really becomes a problem when you're building stateless service objects. To work around it, you can add an `@private` comment to the top of all your intended-private methods, but now we're duplicating not only the business logic in comments, but also the method visibility. We can do better.

On a related note, I see this a lot:

```ruby
class SomethingService
  def self.public_method(some_args)
    #...
  end

  private

  def self.some_private_method(some_more_args)
    #...
  end
end
```

The `private` in this case does nothing, as it only applies to instance methods. To make `some_private_method` private, you need to do `private_class_method(:some_private_method)`.

Okay, back to YARD. It currently has support for the `private_constant` method, which should provide a good starting point for our implementation. That code currently looks like this:

```ruby
  # Sets visibility of a constant (class, module, const)
class YARD::Handlers::Ruby::PrivateConstantHandler < YARD::Handlers::Ruby::Base
  handles method_call(:private_constant)
  namespace_only

  process do
    errors = []
    statement.parameters.each do |param|
      next unless AstNode === param
      begin
        privatize_constant(param)
      rescue UndocumentableError => err
        errors << err.message
      end
    end
    if errors.size > 0
      msg = errors.size == 1 ? ": #{errors[0]}" : "s: #{errors.join(", ")}"
      raise UndocumentableError, "private constant#{msg} for #{namespace.path}"
    end
  end

  private

  def privatize_constant(node)
    if node.literal? || (node.type == :var_ref && node[0].type == :const)
      node = node.jump(:tstring_content, :const)
      const = Proxy.new(namespace, node[0])
      ensure_loaded!(const)
      const.visibility = :private
    else
      raise UndocumentableError, "invalid argument to private_constant: #{node.source}"
    end
  rescue NamespaceMissingError
    raise UndocumentableError, "private visibility set on unrecognized constant: #{node[0]}"
  end
end
```

(There's also an implementation for the legacy 1.8.7 parser, but we'll focus on the newer parser for the sake of the article.)

Looking at the code, we can deduce that it handles the `private_constant` method call, passing each parameter to the `privatize_constant` method which first checks if the param is a literal (meaning a string or symbol) or a variable referencing a constant. If so, it find the constant definition, ensures it's valid and then sets its visibility to private in the metadata repository.

That seems pretty straightforward. With a little trial and error we can come up with something like this:

```ruby
# Sets visibility of a class method
class YARD::Handlers::Ruby::PrivateClassMethodHandler < YARD::Handlers::Ruby::Base
  handles method_call(:private_class_method)
  namespace_only

  process do
    errors = []
    statement.parameters.each do |param|
      next unless AstNode === param
      begin
        privatize_class_method(param)
      rescue UndocumentableError => err
        errors << err.message
      end
    end
    if errors.size > 0
      msg = errors.size == 1 ? ": #{errors[0]}" : "s: #{errors.join(", ")}"
      raise UndocumentableError, "private class_method#{msg} for #{namespace.path}"
    end
  end

  private

  def privatize_class_method(node)
    if node.literal?
      method = Proxy.new(namespace, node[0][0][0], :method)
      ensure_loaded!(method)
      method.visibility = :private
    else
      raise UndocumentableError, "invalid argument to private_class_method: #{node.source}"
    end
  rescue NamespaceMissingError
    raise UndocumentableError, "private visibility set on unrecognized method: #{node[0]}"
  end
end
```

The logic here is fairly similar, though we don't check for variable constant references, and we need to parse down to the method name to add it to the metadata repository. Now YARD correctly recognizes our private class methods and excludes them appropriately.

On another related note, in the process of submitting [a pull request for this update](https://github.com/lsegal/yard/pull/747), the maintainer of YARD addressed a feature I was looking to add in the future, namely the ability to whitelist class methods via the `private_class_method(*(methods(false) - [:whitelisted, :class, :methods]))` idiom. It something I frequently use on the aforementioned service objects, as they usually have a single entry point, which makes whitelisting much easier than blacklisting. Unfortunately, YARD doesn't support dynamic evaluation like that, which makes sense because YARD isn't _interpreting_ your code, it's only parsing it, and the result of `*(methods(false) - [:whitelisted, :class, :methods])` can only really be known at runtime. That said, you could _kind of_ make it work by updating the metadata registry to support wildcards or callbacks, but neither of these implementations would cover all the use cases, and each would be a bear to maintain. Remember, (you don't know what you do until you know what you don't do)[http://blogs.msdn.com/b/oldnewthing/archive/2007/03/21/1922203.aspx], so bravo to the YARD team for knowing when to say no.