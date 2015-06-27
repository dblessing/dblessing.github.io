---
layout: post
title:  "Fun with Ruby Arrays"
date:   2015-06-27 12:50:00
categories: ruby
---

This week alone I have lost hours of my life to strange behavior with Ruby arrays and hashes.
Now that I know what's going on it all makes sense but I can see how someone can easily be
confused by this. I'm sharing it in hopes that others might avoid the same pitfall
and as a reminder to myself in the future.

## Arrays

The issue I ran into with arrays involves iteration. Specifically, while iterating over an array
I was also selectively deleting that item from the array if it matched some criteria. What I found
is that I was 'randomly' failing to iterate over some values in my array.

Here's an example of what I was doing:

```ruby
current_rules = [
  {
    rule_number: 100,
    action: :deny,
    protocol: -1,
    cidr_block: '10.0.0.0/24',
    egress: false,
    port_range: -1
  }
]
desired_rules = [
  {
    rule_number: 100,
    action: :deny,
    protocol: -1,
    cidr_block: '10.0.0.0/24'
  },
  {
    rule_number: 200,
    action: :allow,
    protocol: -1,
    cidr_block: '0.0.0.0/0'
  },
  {
    rule_number: 300,
    action: :allow,
    protocol: 6,
    port_range: 22..23,
    cidr_block: '172.31.0.0/22'
  }
]

desired_rules.each do |desired_rule|
  matching_rule = current_rules
    .select { |r| r[:rule_number] == desired_rule[:rule_number]}.first

  if matching_rule
    # Anything unhandled will be removed
    current_rules.delete(matching_rule)
    # Anything unhandled will be added
    desired_rules.delete(desired_rule)

    ...
  end
end
```

Since `current_rules` and `desired_rules` both have `rule_number: 100` they are deleted from both arrays.
That seems fine, but what happened when that element got removed from the `desired_rules` array I am currently
iterating over?

`desired_rules` now looks like this:

```ruby
[
  {
    rule_number: 200,
    action: :allow,
    protocol: -1,
    cidr_block: '0.0.0.0/0'
  },
  {
    rule_number: 300,
    action: :allow,
    protocol: 6,
    port_range: 22..23,
    cidr_block: '172.31.0.0/22'
  }
]
```

All the remaining elements in the array shifted forwarded by one index. Element with
`rule_number: 200` shifted from index `1` to index `0`. The next iteration over the array is index `1`, now
`rule_number: 300`. Whoops!

Knowing the issue here is most of the battle. The solution is rather easy - just clone the array before iterating:

```ruby
desired_rules.clone.each do |desired_rule|
  ...
end
```

`desired_rules` elements still shift one index forward but we're iterating over a clone that is unaffected by the
delete action.

Special thanks to Christopher Webber (@cwebber) for suggesting this post. It's a great, small topic to get my blog
off the ground. I will remember to blog about small issues like this more in the future.

Unfortunately I don't have comments enabled on my blog yet. If you have comments, please send them to @drewblessing
on Twitter.
