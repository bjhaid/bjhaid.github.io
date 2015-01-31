---
layout: post
title: Understanding the Erlang Syntax
excerpt: In this post I would give a basic walk-through of the erlang syntax
modified: 2015-01-31
tags: [erlang, syntax, beginners]
comments: true
---

As a beginner(enthusiast) the erlang syntax could throw you away if you have not seen haskell or prolog before. In this post I would give a basic walk-through of the erlang syntax, and hopefully help you better understand the erlang syntax.

### Punctuation:

{% highlight erlang %}
-module(fact).
-export([factorial/1]).

factorial(0) -> 1;
factorial(N) when N > 0, is_integer(N) -> N * factorial(N-1);
factorial(_) -> error.
{% endhighlight %}

{% highlight erlang %}
             .
                      .

                 ;
                       ,                                    ;
                     .
{% endhighlight %}

Erlang likes punctuation.

The `.`(period) is used to terminate module attributes and function declarations, therefore the period represents the end of a statement.

The `;`(semicolon) acts as a clause separator, both for function clauses and expression branches.

The `,`(comma) is an expression separator. If a comma follows an expression, it means there's another expression after it in the clause.


### Module Declaration:

{% highlight erlang %}
-module(fact).
{% endhighlight %}

This defines the name of the module (fact), the module name should be the same as the file name minus the extension erl. If the file was name `a.erl` at compilation you would get an error `a.beam: Module name 'fact' does not match file name 'a'`


### Function Exports:

{% highlight erlang %}
-export([factorial/1]).
{% endhighlight %}

This specifies the list of function(s) visible outside the module, it can be thought of as a list of _public functions_. Erlang functions are identified by their arity(number of arguments or operands the function or operation accepts), so `factorial/1` refers to the `factorial` function that takes one argument.


### Functions:

{% highlight erlang %}
factorial(0) -> 1;
factorial(N) when N > 0, is_integer(N) -> N * factorial(N-1);
factorial(_) -> error.
{% endhighlight %}

Above shows definition of the function `factorial`, but the question arises why 3 declarations of the same function? Let's take a look at each line and understand what they do.

- `factorial(0) -> 1;` whenever the `factorial` function is called with argument 0 it should return 1, this feature is regarded as [pattern matching](http://erlang.org/doc/reference_manual/patterns.html)
- `factorial(N) when N > 0, is_integer(N) -> N * factorial(N-1);` there are couple of ideas going on in this line, let's break it into smaller chunks:
  - `when N > 0, is_integer(N)` is a Guard and it infers that the line should only be evaluated if N is greater than 0 and N is an integer
  - `N * factorial(N-1)` is Recursion
  - `factorial(_)` `_` is a catch all pattern, this line would return `error` when none of the previous clauses match, if you pass a string, float or a negative integer to `factorial`



Thanks for reading!
