---
title: FizzBuzz with Pattern Matching
---

_Note: This chapter was written by [Paweł Świątkowski](https://github.com/katafrakt)._


Tom Dalling recently posted a blog post about playing with FizzBuzz in Ruby,
to get a grasp of different approaches to problem solving. I decided to create a small addition to this topic: how to do FizzBuzz in Ruby using pattern matching techniques.

Wait, pattern matching in Ruby? Yes. As you may know, I have written a gem called
[Noaidi](https://github.com/katafrakt/noaidi) which is an attempt to get pattern matching goodies in Ruby. Even though some of reader were not convinced about this being beneficial for the code, here are more examples.

If you want to know more about Noaidi itself (I'm not going to explain it in detail here), read one of the previous posts:
[A Quest for Pattern Matching in Ruby](http://katafrakt.me/2016/02/13/quest-for-pattern-matching-in-ruby/) or
[Refactoring Rails with Noaidi](http://katafrakt.me/2016/05/24/refactoring-rails-with-noaidi/).


## Example 1: Simple pattern matching

In the first step we are going to rewrite a solution number two from Tom's post:

``` ruby
1.upto(100) do |i|
  fizz = (i % 3 == 0)
  buzz = (i % 5 == 0)
  puts case
       when fizz && buzz then 'FizzBuzz'
       when fizz then 'Fizz'
       when buzz then 'Buzz'
       else i
       end
end
```

How? For each `i` we are going to create a pair: `i mod 3` and `i mod 5`
and pass it to the matcher. If both are `0`, we have a FizzBuzz; if only first one is zero, we have a Fizz, etc.

``` ruby
1.upto(100).each do |i|
  results = [i % 3, i % 5]
  Noaidi.match(results) do |m|
    m.(0, 0) { puts 'FizzBuzz' }
    m.(0, Fixnum) { puts 'Fizz' }
    m.(Fixnum, 0) { puts 'Buzz' }
    m.(Fixnum, Fixnum) { puts i }
  end
end
```

This is quite simple and nicely looking. There are some drawback, though:

- We need to use `Fixnum` to match any number. This is not really that bad,
  but having a way to denote "anything, I don't care" would be a bit nicer.
  This is on Noaidi's roadmap.
- We use closure to get a value of `i` in the last match. This prevents us from compiling
  a match beforehand to gen a performance boost (see Noaidi’s README for more details).
  To avoid that, we would need to pass an array of three: `[i, i % 3, i % 5]`
  and this is a bit too much.



## Example 2: A module

Modules of Noaidi were inspired by Erlang. You can (and should - this is the idea) define many functions (called "funs") with the same name but with different signature. The “best match” (i.e. the first one that matches) will be picked and executed.

We could create a direct translation of the matcher above to a module. But to add more fun, we are going to use guard clauses this time to check if we "like" the arguments we get.

Here is the code:

``` ruby
fizz = Noaidi.module do
  fun :buzz, [->(i) { i % 5 == 0 && i % 3 == 0}] do
    'FizzBuzz'
  end
  fun :buzz, [->(i) { i % 3 == 0}] do
    'Fizz'
  end
  fun :buzz, [->(i) { i % 5 == 0 }] do
    'Buzz'
  end
  fun :buzz, [Fixnum] do |n|
    n
  end
end

1.upto(100).each do |i|
  puts fizz.buzz(i)
end
```

Here, the logic was moved to the module completely. In the actual loop we only
call fun `buzz` of module `fizz` with the current number.
Module does its work by checking first if the number is divisible by both 3 and 5, then by each one of them and in the end for just any number. A correct value (string or number) is returned from the module, which is then printed in the loop.

This is faster than previous implementation, as module is created only once, before the loop. We also don’t need intermediate values calculated before passing to the module and thus we don’t need any closures. If you ask me, this is most "correct" solution.


## Example 3: Using dry-matcher

Noaidi is not the only solution for pattern matching in Ruby. Folks from dry.rb have their own implementation, called [dry-matcher](http://dry-rb.org/gems/dry-matcher/).
I'm not a big fan of this solution, as it requires to write a lot of boliterplate code, but I'm giving example of it too.

``` ruby
fizzbuzz_case = Dry::Matcher::Case.new(
  match: -> value { value % 3 == 0 && value % 5 == 0 },
  resolve: -> value {}
)
fizz_case = Dry::Matcher::Case.new(
  match: -> value { value % 3 == 0 },
  resolve: -> value {}
)
buzz_case = Dry::Matcher::Case.new(
  match: -> value { value % 5 == 0 },
  resolve: -> value {}
)
other_case = Dry::Matcher::Case.new(
  match: -> value { true },
  resolve: -> value { value }
)

matcher = Dry::Matcher.new(fizzbuzz: fizzbuzz_case,
 	fizz: fizz_case, buzz: buzz_case, other: other_case)

1.upto(100).each do |i|
  matcher.(i) do |m|
    m.fizzbuzz { puts 'FizzBuzz' }
    m.fizz { puts 'Fizz' }
    m.buzz { puts 'Buzz' }
    m.other { |x| puts x }
  end
end
```

Again, there are four cases, which is no surprise. Each of them has a `value` key,
which is an actual matcher. A `resolve` key denotes what will be passed to an action block,
when matcher is executed.

After having written those four cases, we create an actual matcher. And then it becomes somewhat pleasure. We execute the matcher in the loop and take action depending on what has been matched.

It is not that bad, but clearly requires us to write a lot more code than in previous examples. That being said, it is almost certain that it is much more flexible on edge cases. When we have to do more manual work, we definitely can do crazier things if we want.


## Summary

This is all just fun, of course. It probably won’t convince you to use pattern matching.
But I think it is worth to know that there are libraries in Ruby ecosystem that let you do that.
Maybe you'll find it useful at some point?
