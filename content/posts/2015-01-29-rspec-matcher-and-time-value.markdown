---
layout: post
title: "RSpec matcher and time value"
date: 2015-01-29 20:30:29 -0500
comments: true
categories: ruby rspec time
plugins: backtick_codeblock
---

This will be a quick post that documents an interesting find in RSpec.

## Background

I encountered a classic Ruby testing failure. I have an object whose method takes 2 time values, a start datetime and an end datetime. The intent of the method is to find records matching those values and send an e-mail to a related User account.

As many experienced Ruby programmers have seen, the failure looks like this:

```
Failure/Error: response = put :update, {id: event.id,
  <WaitlistNotifier (class)> received :alert_matching with unexpected arguments
    expected: (Thu, 29 Jan 2015 23:56:42 UTC +00:00, Fri, 30 Jan 2015 00:01:42 UTC +00:00)
    got: (Thu, 29 Jan 2015 23:56:42 UTC +00:00, Fri, 30 Jan 2015 00:01:42 UTC +00:00)
```

Upon closer inspection, you will notice that the values appear to be exactly the same. You may ask yourself then, "Why is there a failure if the values are the same?".

## Equality things

One of the possible reasons why this spec failed is that while the datetime values match down to the second, they're probably mismatching on the millisecond. The accepted solution is to use the *be_within* matcher.

``` ruby
it "should match as close as possible" do
  expect(time_value_under_test).to be_within(0.1).of(expected_time_value)
end
```
If you're comparing time or date/time values, you can stop there. However, my use case has a twist on it. Instead of comparing whether or not the time value is what I expect, I wanted to ensure that the method I'm calling is passed the appropriate time values.

``` ruby
class WaitlistNotifier
  def alert_matching(datetime_start, datetime_end)
end

describe SomeController do
  it "should pass the appropriate time values to the notifier" do
      expect(WaitlistNotifier).to receive(:alert_matching).with(start_time, end_time)
  end
end
```

As you can see, I don't care if the values are equal down to the millisecond, all I care is that it's the appropriate value passed in. RSpec makes no such distinction, nor should it.

Having consulted a few peeps on Twitter, including [Myron Marston](https://twitter.com/myronmarston), I got my answer. The *be_within* matcher can be used within a *with* call!

``` ruby
describe SomeController do
  it "should pass the appropriate time values to the notifier" do
      expect(WaitlistNotifier).to receive(:alert_matching).
      								with(a_value_within(1).of(start_time),
      								     a_value_within(1).of(end_time))
  end
end
```
According to the [RSpec documentation](http://rspec.info/documentation/3.2/rspec-expectations/RSpec/Matchers.html#be_within-instance_method), *a_value_within* is an alias of *be_within*.

I think this is a pretty cool and pretty powerful part of RSpec and I confess that I was not aware of it AT ALL.

Hopefully, this will save someone time in the future.