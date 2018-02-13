---
layout: post
title:  "ChefDK: Rakefile for common tasks"
date:   2018-02-07 12:08:21 -0700
categories: Chef ChefDK
---

## Introduction

Good practice is to run rubocop, foodcritic, and ChefSpec (rspec) tests, and test kitchen before pushing your changes, as we've [talked about before]().

And it's really not THAT much to type every time to go:

{% highlight bash %}
$ rubocop
$ rubocop -D
$ rubocop -a
$ foodcritic
$ foodcritic . # I always forget the dot...
$ rspec --color -f d
$ kitchen test
{% endhighlight %}

But wouldn't it be nice to automate this, so we just have one command to type?

Actually, you can now do that with `delivery` locally.  But that comes with some other considerations which deserves its own post.

So for now, enter, Ruby's Make! aka [rake](https://github.com/ruby/rake)

## Rakefile

Rake really is a very powerful tool.  I recommend reading over the documentation linked as well as [this article](https://edelpero.svbtle.com/everything-you-always-wanted-to-know-about-writing-good-rake-tasks-but-were-afraid-to-ask).  We'll focus on just enough rake to automate our tests.


### Basic Rakefile

{% highlight ruby %}
task default: :reticulate_splines

desc 'Reticulate splines'
task :reticulate_splines do
  puts 'Reticulating Splines...'
  # doing something useful like reticulating splines...
end
{% endhighlight %}

Then we can run it.  Also we can run rake with the `-T` flag to see all tasks avilable.

{% highlight bash %}
$ rake -T
rake reticulate_splines  # Reticulate splines
$ rake
Reticulating Splines...
{% endhighlight %}

This isn't very useful, but is a good structure to show the basicas of a Rakefile.

### Rakefile explanation

{% highlight ruby %}
task default: :reticulate_splines
{% endhighlight %}

The first line describes the default task.  This is the task that will be run when `rake` is called without any arguments.

{% highlight ruby %}
desc 'Reticulate splines'
{% endhighlight %}

The next line is the description of the task.  The `desc` function call must preceede a `task` block.

{% highlight ruby %}
task :reticulate_splines do
  puts 'Reticulating Splines...'
  # doing something useful like reticulating splines...
end
{% endhighlight %}

Finally, the task block.  The task is named as a symbol, then within the block a function is performed.  In this case we're only printing a message, but it's just Ruby, so we can do any useful functions.

## Chef Rakefile

What are the things we always do?

* [RuboCop](https://github.com/bbatsov/rubocop)
* [ChefSpec](https://github.com/chefspec/chefspec)
* [FoodCritic](https://github.com/Foodcritic/foodcritic)
* [Test Kitchen](https://github.com/test-kitchen/test-kitchen)

To that end, let's get started!

### Rake Rubocop

The first thing to do, include RuboCop!  Since version 0.10 RuboCop comes with a rake task.

{% highlight ruby %}
require 'rubocop/rake_task'

RuboCop::RakeTask.new do |rubocop|
  rubocop.options = ['--display-cop-names']
end
{% endhighlight %}

First we require the library with the rake task.

Then we initialize the task, by calling RuboCop::RakeTask.new.

Personally, I like to have the cop names displayed when I run rubocop, so we take the opportunity to add that option when we run rubocop.  You can pass any flags here you would pass on the command line.

Then when we run `rake -T` we get:

{% highlight bash %}
$ rake -T
rake rubocop               # Run RuboCop
rake rubocop:auto_correct  # Auto-correct RuboCop offenses
{% endhighlight %}

Now we can have `rake` run `rubocop` for us!

### Rake Foodcritic

Similar to RuboCop, FoodCritic has built in rake tasks.

{% highlight ruby %}
require 'foodcritic'

FoodCritic::Rake::LintTask.new do |foodcritic|
  foodcritic.options[:fail_tags] = %w(any)
end
{% endhighlight %}

First we again require the library.  Then we initialize the FoodCritic tasks.  Additional options can be set as keys:value pairs to #options.

Now we should have:

{% highlight bash %}
$ rake -T
rake foodcritic            # Lint Chef cookbooks
rake rubocop               # Run RuboCop
rake rubocop:auto_correct  # Auto-correct RuboCop offenses
{% endhighlight %}

### Rake RSpec

Finally, RSpec tests.  This also has a rake task.  So following the same pattern...

{% highlight ruby %}
require 'rspec/core/rake_task'

RSpec::Core::RakeTask.new do |rspec|
  rspec.rspec_opts = "--color --format documentation"
end
{% endhighlight %}

And now we should have one more rake task:

{% highlight bash %}
$ rake -T
rake foodcritic            # Lint Chef cookbooks
rake rubocop               # Run RuboCop
rake rubocop:auto_correct  # Auto-correct RuboCop offenses
rake spec                  # Run RSpec code examples
{% endhighlight %}

### Rake Test Kitchen

Finally Test Kitchen.  And we follow the same pattern.

{% highlight ruby %}
require 'kitchen/rake_tasks'
Kitchen::RakeTasks.new
{% endhighlight %}

We don't really have any options to change, so we just call "new".  Now we have something along the lines of:

{% highlight bash %}
$ rake -T
rake foodcritic                   # Lint Chef cookbooks
rake kitchen:all                  # Run all test instances
rake kitchen:default-centos-7     # Run default-centos-7 test instance
rake kitchen:default-ubuntu-1604  # Run default-ubuntu-1604 test instance
rake rubocop                      # Run RuboCop
rake rubocop:auto_correct         # Auto-correct RuboCop offenses
rake spec                         # Run RSpec code examples
{% endhighlight %}

### Pulling it all together

Remember I said I was lazy?  Probably too lazy to say it.  It's great that we have one command to run everything, but I still have to remember and give rake a subcommand...  if only there was a way that we could have one rake task call another...?  Also note, we haven't declared a default task yet.

{% highlight ruby %}
task default: :all

desc 'Run all tests'
task all: [:test, :kitchen:all]

desc 'Run Rubocop and Foodcritic style checks'
task style: [:rubocop, :foodcritic]

desc 'Run all style checks and unit tests'
task test: [:style, :spec]
{% endhighlight %}

This gives us a few new commands that will call the various linters and test tools for us directly.  You should have something like:

{% highlight bash %}
$ rake -T
rake all                          # Run all tests
rake foodcritic                   # Lint Chef cookbooks
rake kitchen:all                  # Run all test instances
rake kitchen:default-centos-7     # Run default-centos-7 test instance
rake kitchen:default-ubuntu-1604  # Run default-ubuntu-1604 test instance
rake rubocop                      # Run RuboCop
rake rubocop:auto_correct         # Auto-correct RuboCop offenses
rake spec                         # Run RSpec code examples
rake style                        # Run Rubocop and Foodcritic style checks
rake test                         # Run all style checks and unit tests
{% endhighlight %}

## Final Form

{% gist 39f1b2e6e2f24181e307841ba180ef41 %}

## Conclusion

Hopefully this will encourage you to run your tests before you push your code!

- Q
