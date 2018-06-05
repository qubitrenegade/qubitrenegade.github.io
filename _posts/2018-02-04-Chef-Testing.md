---
layout: post
title:  "Chef: Getting Started with Testing"
date:   2018-02-01 12:13:57 -0700
categories: Chef Testing
---

## Introduction

Last time we talked about how to get started.  We're going to continue down that journey, but we need to have a talk about testing first.

Testing is your friend!

I know, I know...  "If I spend my time writing tests, I have less time to spend writing code" and "Boss says we need these features today, I don't have _time_ to write tests".  It's the age old "I don't have time to save time paradox."

But here's a secret.  You _DO_ have time to write tests and writing tests will actually _save time_.

## Saving time to code by not writing code

Have you ever gone into a grocery store without a shopping list?  It's arduous, painful, and generally takes too long!  If you do it enough, you probably start to form some heuristics "start on this end of the store, hit that end, then go to the middle where the bulky TP/PT is..." which saves time, but you'll tend to forget something and have to go back to the other end...

If you go in with a shopping list, well, it's still grocery shopping, so generally unenjoyable... but you're able to plot a path, check off what you've picked up, which gives you an easy list of what you still need allowing you to further adjust your path on the fly.

Sure, writing that grocery list is a small up front investment, but that pays dividends in reducing the time you spend actually in the grocery store.

## Grocery shopping for code

In many ways, writing unit tests is a lot like writing a grocery list, but for code artifacts not food artifacts.

Let's say we have a grocery list of:

{% highlight bash %}
milk
eggs
bread
{% endhighlight %}

We could easily write this as RSpec stubs:

{% highlight ruby %}
it 'gets milk'
it 'gets eggs'
it 'gets bread'
{% endhighlight %}

Now I have a check list!  They are called stubs because when we run our test suite RSpec will print out a "not yet implemented" notification message.

We can then further extend our RSpec tests to automate validating our code.  if "milk" is a function and it returns an object of type Milk, you could write your test as:

{% highlight bash %}
def get_milk
  Milk.new()
end

it 'gets milk'
  expect(get_milk).to be_an_instance_of(Milk)
end
{% endhighlight %}

Now we've automated validation of our "get_milk" function.  If we make a change and this test fails, we know we've broken an underlying component somewhere and can take corrective action, _BEFORE_ it fails in production.

## Types of Testing

You know, I'd like to blame marketing, and I do... Cloud... Serverless... but we did this one to ourselves.  This of course isn't an exhaustive list...

* Unit - "Does our code have the artifacts I expect, do they behave how I expect?"
* Lint - "Does our code meet style guidelines?"
* Syntax - "Is our code valid?"
* Quality - "Does my code use anti-patterns? Do I have good code coverage with my unit tests?"
* Smoke - "Does our code work? Does it produce the expected outputs with simulated inputs?"
* Integration - "Does our code work with other code/services/applications?"
* User Acceptance Testing (UAT) - "Can my users use it?"

Not all of this is going to happen locally.  E.g.: quality testing can take a long time.  If Unit, Lint, and Syntax don't pass, no sense running quality tests. So we run quality tests as part of a pipeline and just run Unit, Lint, and Syntax tests locally.

## Chef Testing Tools

* [RuboCop](https://github.com/bbatsov/rubocop)
* [FoodCritic](https://github.com/Foodcritic/foodcritic)
* [ChefSpec](https://github.com/chefspec/chefspec)
* [Test Kitchen](https://github.com/test-kitchen/test-kitchen)
* [InSpec](https://www.inspec.io/)

## RuboCop

[RuboCop](https://github.com/bbatsov/rubocop) is Ruby's Style Cop.  It has a [TON](https://github.com/bbatsov/rubocop/blob/master/config/default.yml) of configurable options that can be set with a local .rubocop.yml file.  

Consistency is credibility, so by ensuring you maintain a consistent style, others (or yourself 6mos from now) will be able to pick your code with greater ease.

It's worth calling out there is now (2 years old) a [CookStyle](https://github.com/chef/cookstyle) which seems to set sane defaults and warrants further investigation.

Some settings I like to include in my .rubocop.yml

{% highlight ruby %}
# 80 it WAY to short
Metrics/LineLength:
  Max: 120

# Your test blocks can get long
Metrics/BlockLength:
  Exclude:
    - 'spec/*/*/*_spec.rb'

# This is a matter of making the files more readable.
# But it breaks style rules...
Style/ExtraSpacing:
  Exclude:
    - 'metadata.rb'
    - 'attributes/default.rb'
{% endhighlight%}

RuboCop can also automatically fix a lot of errors with the -a flag.

## FoodCritic

Similar to RuboCop, this looks for Chef anti-patterns and provides suggestions as to appropriate ways to fix the cookbook.  This helps avoid known "gotchas".

It doesn't offer autocorrect, but the list of rules found at [foodcritic.io](https://foodcritic.io) provide fairly comprehensive examples.

## ChefSpec

Unit testing for Chef.

## Test Kitchen

The best way to see if code works is to run it!  To that end, Test Kitchen allows you to create a new instance and converge your cookbook on it.  Test Kitchen is actually not limited to Chef as it is able to converge Puppet modules and Ansible Playbooks.  We don't really care about that here, but it's pretty flexible.  

Test Kitchen is a whole topic unto itself.  The short version is, it automates the creating of an instance, converging the cookbook, and running any validation tests to validate the cookbook does what's expected.

## InSpec

Allows you to automate validation of your Cookbook.  "I expect my cookbook to create /etc/httpd/my-site.conf, with the string qubitrenegade.com"

Annie Hedgpeth wrote an amazing [tutorial on InSpec](http://www.anniehedgie.com/inspec/).  Seriously, go read it.  It will change your life.  It's the definitive guide.

## Conclusion

If we run our RuboCop, FoodCritic, ChefSpec, and Test Kitchen, we can be more confident that our code will do what we expect "in the wild".  By writing our tests first we give our self a check list of features our code needs to have, which enables us to form and execute a plan, instead of meandering through our code.  All of this helps us save time because we're no longer waiting for long build processes to find simple syntax errors we can be finding locally.
