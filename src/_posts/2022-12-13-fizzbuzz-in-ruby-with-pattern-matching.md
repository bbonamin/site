---
layout: post
title: "FizzBuzz in Ruby with Pattern Matching"
date: 2022-12-13 19:22:33 -0300
categories: ruby
---

The addition of [Pattern Matching to Ruby since version 2.7](https://www.ruby-lang.org/en/news/2019/12/25/ruby-2-7-0-released/) allow for really expressive code... learning more about Elixir and coming back to Ruby being able to do this feels like cheating:

```ruby
def fizz_buzz(number)
  r = case {r3: number % 3, r5: number % 5, number:}
    in { r3: 0, r5: 0} then "FizzBuzz"
    in { r3: 0} then "Fizz"
    in { r5: 0} then "Buzz"
  else number
  end
  puts r
end

(10..17).each { |n| fizz_buzz n}
#=> Buzz
#=> 11
#=> Fizz
#=> 13
#=> 14
#=> FizzBuzz
#=> 16
#=> 17
```
