---
layout: post
title: Currying and Partial Application in F#
---

Currying is one of those subjects that continually seems to slip out of the back of my brain when I'm not looking. So I figured the best course of action would be to put my reasoning on the subject somewhere more permanent.

<!--end_excerpt-->

## So, currying...

F#, not uniquely among functional programming languages, has origins in [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus). One of the core tenets of this school of mathematical thought is "functions take one value and return one value".

So consider the following function declaration in F#. Two inputs, one output. Is this breaking the rules of lambda calculus?

{% highlight Haskell %}
let add x y = x + y
{% endhighlight %}

The answer is no (of course). When the above method is declared, what actually gets created by the compiler is a chain of functions, each with a single argument. This under-the-hood legwork allows functions with multiple inputs to be declared in a nice, syntactically clean way, while the code still holds true to the fundamental rules. And this general process, in which a function with multiple arguments is transformed into a chain of single-argument functions, is referred to as currying. (Named for Haskell Curry, of course. There, saves you a trip to [wikipedia...](https://en.wikipedia.org/wiki/Currying))

More detail?  OK.

We can agree that the *outerAdd* function shown below does indeed meet the rule of one in, one out.

{% highlight Haskell %}
let outerAdd x = 
    let innerAdd y = 
        x + y
    innerAdd
{% endhighlight %}

We can also agree that the *outerAdd* and *add* functions both have the same signature.

Wait - you don't agree? Then throw them both into FSI, and let's compare:

{% highlight Haskell %}
val outerAdd : x:int -> (int -> int)

val add : x:int -> y:int -> int
{% endhighlight %}

The *outerAdd* function takes one thing: a single *int* parameter (named *x*), and returns one thing: another function. And that function itself takes a single *int* parameter, and returns a single *int* parameter.

The *add* function actually does the same - it has been automatically curried into a form that satisfies that basic tenet of lambda syntax. One argument in, one argument out.

The signatures expose the underlying transformation. The -> symbol, that's a [function declaration delimiter](https://msdn.microsoft.com/en-us/visualfsharpdocs/conceptual/functions-%5bfsharp%5d#function-values). Any time you see it in F# code, it means a function is being declared; the type immediately to the left of it is the type of the input parameter to the function, and _everything_ to the right is the output. (Yes, the parentheses are missing from the *add* signature, but since the function declaration delimiter is right associative, it works the same with or without braces.)

Great. Wait, why is this great again?

## Partial application!

Well, for one thing, the fact that F# internally curries your functions leads to the concept of *partial application*. This allows us to pass an incomplete set of arguments to a function, and partially execute that function using those arguments. This results in a new function being returned to the caller.

{% highlight Haskell %}
let add1 = outerAdd 1
{% endhighlight %}

As we know, when we run the *outerAdd* function, we will get a function as the return value (which is actually the *innerAdd* function, with the *x* argument fixed to 1 in the outer scope).

When we run this resulting function (at a later time), it will demand the remaining argument, and will add it to the fixed value.

So here's the good bit: because the original *add* function is actually being curried for us, we're free to partially apply it, and get a function that's identical to *innerAdd*:

{% highlight Haskell %}
let add1' = add 1
{% endhighlight %}

And this extends to functions with any number of arguments - you can fix these arguments, in order left-to-right, to produce new functions.

## How handy!

Partial application can make code a lot cleaner, and increase reuse and readability. It's also used a lot when working with collections.

{% highlight Haskell %}
let incBy x y = x + y
let ints = [1..10]
{% endhighlight %}

If we wished to apply *incBy* to *ints* as a means of increasing each value in the list by 1, we could do this as follows:

{% highlight Haskell %}
ints
|> List.map (fun x -> incBy 1 x)
{% endhighlight %}

The first argument supplied to *List.map* needs to be a function that takes a single argument. So we can satisfy this by passing a [lambda expression](https://msdn.microsoft.com/en-us/visualfsharpdocs/conceptual/lambda-expressions-the-fun-keyword-%5Bfsharp%5D) that calls *incBy*. This accomodates the two-argument signature of *incBy* and satisfies the requirement of *List.map*.

However, the use of lambda syntax could distract a little from the purpose of the code. We can use partial application to clarify things:

{% highlight Haskell %}
ints
|> List.map (incBy 1)
{% endhighlight %}

The partially applied *incBy* returns a function that is suitable for supplying as the first argument to *List.map*, so there's no need to define a lambda expression any more.

You could simplify further by defining a symbol for the partially applied function instead:

{% highlight Haskell %}
let incBy1 = incBy 1

ints
|> List.map incBy1
{% endhighlight %}

Or indeed, by partially applying the *List.map* operation itself:

{% highlight Haskell %}
let mapIncBy1 = List.map incBy1

ints 
|> mapIncBy1
{% endhighlight %}

Alright, that example code was already very simple, and didn't really need a lot of clarifying in the first place! But when it comes to learning a language (or anything else), if you really understand how the basic stuff works, it makes it easier to concentrate on the complicated stuff!

## Other resources

That's about it. There are many terrific resources around that go into a lot more detail than this.  Here's a few:

For all things F#, Scott Wlaschin's [fsharpforfunandprofit](http://www.fsharpforfunandprofit.com) is basically the online bible!

In particular, his [article on currying](http://fsharpforfunandprofit.com/posts/currying/) covers some other aspects and gotchas not touched on here.

Also, follow this link for a great write-up of [partial application and currying in JavaScript](http://benalman.com/news/2012/09/partial-application-in-javascript/).
