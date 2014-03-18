---
layout: post
title: Showing Disabled Lines In RuboCop
---

[RuboCop](https://github.com/bbatsov/rubocop) is a great analysis tool for Ruby. Not only does it help normalize style across teams, it promotes quality code by flagging things like long methods and high cyclomatic complexity.

Sometimes, however, it's warnings need to be ignored. The candidates most likely to fall into this category are the [MethodLength](https://github.com/bbatsov/rubocop/blob/master/lib/rubocop/cop/style/method_length.rb) and [ClassLength](https://github.com/bbatsov/rubocop/blob/master/lib/rubocop/cop/style/class_length.rb) cops. Maybe your method is only calling two or three other methods, but between their names and the names of their arguments you exceed the default 10 line limit, or maybe your class has a lot of associations and accessors and exceeds the default limit of 100 lines. Either way, the _intent_ of the rule --to reduce complexity by enforcing single responsibility-- is intact. Fortunately, RuboCop allows you to note these exceptions via code comments, like so:

```ruby
class LongClass #rubocop:disable ClassLength
  # Longer than 100 line class that won't
  # trigger a warning.

  def some_long_method #rubocop:disable MethodLength
    # Longer than 10 line method that won't
    # trigger a warning....
  end
end
```

But with great power comes great responsibility, and `# robocop:disable` is easily abused. Maybe it's a junior developer who doesn't understand the warning, or maybe it's an experienced coder who intended to go back and fix the problem, either way, it's easy for unintended disable comments to slip into the code base. So let's add a way to see all the inline excludes that were encountered during a run.

Looking at the source, we can see the application entry point in is the `run` method in `cli.rb`. From there we can see that we'll need to modify `options.rb` to add support for our new option. Since this is either an on or off option, we can append it to the `add_boolean_flags` method.

```ruby
def add_boolean_flags(opts)
  # ...
  option(opts, '-x', '--show-disabled',
         'Display cops disabled in comments.')
end
```

Next, let's look inside `file_inspector.rb`, which is responsible for parsing the files. Inside the `inspect_file` method, we can see it instantiates a new instance of the `Cop::Team` class, and uses that class to get the file's offenses. Within `team.rb`  we can see it also has an `inspect_file` method that instantiates an instance of `SourceParser` and `source_parser.rb` instantiates an instance of `ProcessedSource`, and `processed_source.rb` instantiates an instance of `CommentConfig`.

That's a lot of drill-down, but we've found what we're after. `CommentConfig` is responsible for parsing the `rubocop:disable` comments. It has an instance variable `@cop_disabled_line_ranges` that's initialized when the file is parsed. We can expose this via an accessor method. Since the `@cop_disabled_line_ranges` hash has keys for each mentioned cop regardless if they actually trigger warnings or not, we'll filter out the entries with no line ranges, like so:

```ruby
def disabled_line_ranges
  disabled = @cop_disabled_line_ranges || {}
  disabled.select { |cop, line_ranges|  line_ranges.any? }
end
```

Now it's just a matter of merging these values up the call chain and printing the aggregated result in `run` in `cli.rb`. I've opened [a pull request](https://github.com/bbatsov/rubocop/pull/900) with these changes as a starting point. Other features, like whitelisting comments or comment thresholds could use this as a starting point.