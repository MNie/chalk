---
layout: post
title: How to define "proper" function in Fable
description: "Little insight, how the definition of a function in F# would result in a Javascript code."
thumb_image: "documentation/sample-image.jpg"
tags: [fable, javascript]
---

Hi,

if you spot the name "proper" in a title of this post I wanna say, that I don't want to be a person that says "you should do that"/"you shouldn't do that", but simply give you an insight, how the definition of a function in F# would result in a Javascript code.

So to give a little bit of a context for this article. When I worked with a guacamole protocol and it's javascript implementation I have to invoke a function named `oninstruction(arg1, arg2)`. There is no available binding for it in `Fable.Browser` nor available nuget package for it, so we do our implementation. But after some tries, we discover, that we don't invoke in the javascript method that we want. So how we could define a function in F# with multiple parameters and how they would be transformed via Fable to Javascript?

* Define function parameters as `partial application`

{% highlight fsharp %}
member val function: `TArg1 -> `TArg2 -> `TResult
{% endhighlight %}

And thanks to a Fable we would get the following javascript function

{% highlight javascript %}
function (arg1){
    function (arg2) { ... }
}
{% endhighlight %}

Not really what we want to achieve, right?

* Define function parameters as `tuple`

{% highlight fsharp %}
member val function: `TArg1 * `TArg2 -> `TResult
{% endhighlight %}

Here is the Javascript representation:

{% highlight javascript %}
function (tupleArg) {...}
{% endhighlight %}

The `tupleArg` would be a tuple with 2 elements inside. So still not the result we want to achieve

* Define a function with a `Func<_>`

{% highlight fsharp %}
member val function: Func<`TArg1, `TArg2, `TResult>
{% endhighlight %}

And here is our lovely javscript:

{% highlight javascript %}
function (arg1, arg2) { ... }
{% endhighlight %}

As you could see there are 3 options (or even more, but we don't try them) to define a function in F# which would result in different Javascript code. So keep that in mind, when you define your `marker interface` for Fable binding.

Thanks for reading!