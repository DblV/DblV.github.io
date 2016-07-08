---
layout: post
title: Currying and Partial Application in F#
---

Currying is one of those subjects that continually seems to slip out of the back of my brain when I'm not looking, so I figured the best course of action would be to put my reasoning on the subject somewhere more permanent.

<!--end_excerpt-->

## So, currying...

F#, not uniquely among functional programming languages, has origins in lambda calculus.  One of the core tenets of this school of mathematical thought is "functions take one value and return one value".
So consider the following function declaration in F#.  Is this breaking the rules of lambda calculus?

{% highlight fsharp %}
let add x y = x + y
{% endhighlight %}

The answer is no (of course). There's some legwork being performed under the hood to allow functions with multiple inputs to be declared in a nice syntactically clean way, while the code still holds true to the fundmaental rules.
When the above method is declared, what actually gets created by he compiler is a chain of functions, each with a single argument.  And the process by which a function with multiple arguments is transformed into a chain of single-argument functions is referred to as currying.
Here's my stab at explaining.
We can agree that the *add* function above has the same signature as the one shown below:

{% highlight fsharp %}
let outerAdd x = 
    let innerAdd y = 
        x + y
    innerAdd
{% endhighlight %}

If you don't agree, throw them both into FSI and compare the resulting signatures:

{% highlight fsharp %}
val add : x:int -> y:int -> int

val outerAdd : x:int -> (int -> int)
{% endhighlight %}

The signatures expose the underlying transformation. The *outerAdd* function takes a single *int* parameter (named *x*), and returns (->) one thing: a function which itself takes a single *int* parameter, and returns (->) a single *int* parameter.
And the *add* function does the same thing - it has been automatically curried into a form that satisfies that basic tenet of lambda syntax - one argument in, one argument out.
(Yes, the parentheses are missing from the signature, but since the function declaration delimiter (->) is right associative, it works the same with or without braces.)

Great. Wait, why is this great again?

## Partial application!

Well, for one thing, the fact that F# curries your functions for you internally opens the door to the concept of *partial application*. This allows us to pass a subset of arguments to a function, and partially execute that function using those arguments, resulting in a new function being returned to the caller.  Look see:

{% highlight fsharp %}
let add1 = outerAdd 1
{% endhighlight %}

As we now know, when we run the *outerAdd* function, we will get back a function as the return value (actually the *innerAdd* function, but with one of the values passed in fixed to 1).
When we run the resulting function, it will demand the remaining argument, and will always add it to the fixed value.

And because the original add function is actually curried, we can do the same thing with it:

{% highlight fsharp %}
let add1' = add 1
{% endhighlight %}

This is partial application at work.  The arguments for a multiple-argument function can be fixed (in order left-to-right) to produce new functions.

## How handy!

Partial application can make code a lot cleaner, increase reuse and increase readability. It's also used a lot when working with collections.

{% highlight fsharp %}
let incBy x y = x + y
let ints = [1..10]
{% endhighlight %}

If we wished to apply incBy to the list of ints as a means of increasing each value by 1, we could do this as follows:

{% highlight fsharp %}
List.map (fun x -> incBy 1 x) ints
{% endhighlight %}

We need to use lambda syntax to invoke incBy since it has two arguments, and List.map takes as an argument a function to be applied to each value in the list, and that function can itself only take one argument.
But the lambda syntax can distract from the purpose, so we can use partial application to clarify:

{% highlight fsharp %}
List.map (incBy 1) ints
{% endhighlight %}

Actually, this reads a bit better as

{% highlight fsharp %}
ints
|> List.map (incBy 1)
{% endhighlight %}

Of course, you could define a symbol separately for the partially applied function instead:

{% highlight fsharp %}
let incBy1 = incBy 1
ints
|> List.map incBy1
{% endhighlight %}

Or indeed, for the List operation *and* the partially applied function:

{% highlight fsharp %}
let mapIncBy1 = List.map incBy1
ints |> mapIncBy1
{% endhighlight %}

Alright, that got a bit exaggerated for a simple example, but you get the idea.


## Credits

There is a fantastic (and ever-increasing) amount of information around on the subjects of currying and partial application in F# and other languages, but if you only choose to look at one, make it Scott Wlaschin's [fsharpforfunandprofit](http://www.fsharpforfunandprofit.com) - basically the canonical single source for all things F#!  I attended an introduction to F# by Mr Wlaschin recently, so he is indirectly and unknowingly responsible for the existence of this blog post!
 
