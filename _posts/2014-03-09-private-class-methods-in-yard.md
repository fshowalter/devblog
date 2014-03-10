---
layout: post
title: Private Class Methods in YARD
---

doc·u·men·ta·tion:
_noun_

1. something developers don't want to write.
2. something that those same developers will moan about if it's not present in a third-party library they're using.

Lets get two things out of the way right off:

1. Documentation is a pain. It requires maintaining two copies of your domain logic, one in the documentation and another in the code. Keeping these in sync is difficult, made more so by the inability to rely on automated tests or code analysis to warn you when the two have diverged.

2. Self-documenting code is something we should all be working towards, but it's not here yet. No matter how well-named your method and its arguments are, it can't tell me what kind of exceptions it may throw, provide an example of how to call it, or all the various options it could take. Someday, perhaps, but we're coding in the here and now.

That said, there are tools to make documentation less of a chore. Right now, one of the best is [YARD](http://yardoc.org/). If you've browsed the [documentation for Capybara](http://rubydoc.info/github/jnicklas/capybara/master/frames/file/README.md), or [FriendlyId](http://norman.github.io/friendly_id/file.Guide.html) you've seen YARD in action.

One of YARD's strengths is its ability to automatically exclude private (and optionally protected) methods, thus easing the documentation burden by reducing it to public entry points.

Except currently YARD doesn't recognize Ruby's [`private_class_method`](http://ruby-doc.org/core-2.1.0/Module.html#method-i-private_class_method), which means that all class methods defined by `def self.class_method_name` are always included. 

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

Don't do that. The `private` in this case does nothing, as it only applies to instance methods because you're calling it on the instance class. To make `some_private_method` private, you need to either do `private_class_method(:some_private_method)` or define your class methods on the class's meta-class like this:

```ruby
class SomethingService
  class << self
    def public_method(some_args)
      #...
    end
  
    private
  
    def some_private_method(some_more_args)
      #...
    end
  end
end
```

The Rails convention is to use `class << self` (which incidentally works with YARD too)  but I prefer `self.my_class_method` as it's more explicit, so let's enhance YARD. Granted, we could add an `@private` comment to the top of all our intended-private methods, but now we're duplicating not only the business logic in documentation, but also the method visibility. We can do better.

YARD currently has support for the `private_constant` method, which should provide a good starting point for our implementation. That code currently looks like this:

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

Looking at the code, we can deduce that it handles the `private_constant` method call, passing each parameter to the `privatize_constant` method which first checks if the param is a literal (meaning a string or symbol) or a variable referencing a constant. If so, it finds the constant definition, ensures it's valid, and then sets its visibility to private in the metadata repository.

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

On another related note, in the process of submitting [a pull request for this update](https://github.com/lsegal/yard/pull/747), the maintainer of YARD addressed a feature I was looking to add in the future, namely the ability to whitelist class methods via the `private_class_method(*(methods(false) - [:whitelisted, :class, :methods]))` idiom. It something I frequently use on the aforementioned service objects, as they usually have a single entry point, which makes whitelisting much easier than blacklisting. Unfortunately, YARD doesn't support dynamic evaluation like that, which makes sense because YARD isn't _interpreting_ your code, it's only parsing it, and the result of `*(methods(false) - [:whitelisted, :class, :methods])` can only really be known at runtime. That said, you could _kind of_ make it work by updating the metadata registry to support wildcards or callbacks, but neither of these implementations would cover all the use cases, and each would be a bear to maintain. Remember, [you don't know what you do until you know what you don't do](http://blogs.msdn.com/b/oldnewthing/archive/2007/03/21/1922203.aspx), so bravo to the YARD team for knowing when to say no.
