---
layout: post
title: Testing GraphQL queries with FsCheck library - Shrinking input
description: "GraphQL essentially allows users to define queries from a front-end to gather concrete data. So as it looks it is completely different than a standard REST endpoint. GraphQL differs queries which only gather data from those which also mutates them. In this article, I want to concentrate on queries which only gather some data and how to test them via FsCheck and F#. This articles is a continuation of previous post https://www.mnie.me/2018-05-07-graphQLTesting/"
thumb_image: "documentation/sample-image.jpg"
tags: [fsharp, graphql, cleancode, tests, tdd, automatedtests, patterns]
---

Hi,

today I want to continue a topic about testing graphQL queries thanks to the FsCheck library. How we could achieve that I describe briefly [here](https://www.mnie.me/2018-05-07-graphQLTesting/).

So I want to focus on a situation when our tests spot some problems. By default, we should get in a unit test output window information about the query which caused a failed test thanks to that we could copy-paste this query and check manually what is wrong.

But what would be when our query or graphTypes are very complex and not every field is incorrect? In a situation like this we would like to get the minimum query which produced a failure, so we could leverage time needed to spend on a manual check. We could achieve that by writing a custom shrinker. If some test fails shrinker tries to change somehow an input for next iterations of a test so we could get the minimum failure query in our case.

So to see this in some example, previously we write tests for a Query and GraphType which are defined as follows:

{% highlight csharp %}
public class Engine
{
    public string Indicator { get; set; }
    public int Power { get; set; }
    public int Displacement { get; set; }
}
public class Car
{
    public string Name { get; set; }
    public string Model { get;set; }
    public Engine Engine { get;set; }
}
{% endhighlight %}

GraphType:

{% highlight csharp %}
public class EngineGraphType : ObjectGraphType<Engine>
{
    public EngineGraphType(IEngineProvider engineProvider)
    {
        this.Name = "Engine";

        this.Field<NonNullGraphType<StringGraphType>>(
            "indicator",
            resolve: ctx => ctx.Source.Indicator
        );
        this.Field<NonNullGraphType<IntGraphType>>(
            "power",
            resolve: ctx => ctx.Source.Power
        );
        this.Field<NonNullGraphType<IntGraphType>>(
            "displacement",
            resolve: ctx => ctx.Source.Displacement
        );
}

public class CarGraphType : ObjectGraphType<Car>
{
    public CarGraphType(IEngineProvider engineProvider)
    {
        this.Name = "Car";

        this.Field<NonNullGraphType<StringGraphType>>(
            "name",
            resolve: ctx => ctx.Source.Name
        );
        this.Field<StringGraphType>(
            "model",
            resolve: ctx => ctx.Source.Model
        );
        this.Field<EngineGraphType>(
            "engine",
            resolve: ctx => engineProvider.Provide(ctx.Source.Model)
        );
    }
}
{% endhighlight %}

And finally a query:

{% highlight csharp %}
public class CarQuery : RootQueryField
{
    private readonly ICarResolver _carResolver;

    public CarQuery(ICarResolver carResolver)
    {
        this._carResolver = carResolver;
        this.Name = "car";

        this.Arguments = new QueryArguments(
            new QueryArgument<NonNullGraphType<StringGraphType>>
            {
                Name = "name"
            }
        );

        this.Resolver = new FuncFieldResolver<Task<Car>>(this.Resolve);

        this.Type = typeof(CarGraphType);
    }

    private Task<Car> Resolve(ResolveFieldContext ctx)
    {
        var param = ctx.Arguments.ToObject<ConcreteArguments>();
        return _carResolver
            .ResolveAsync(param);
    }
}
{% endhighlight %}

We could assume right now that `model` field is resolved incorrectly. This bug was spotted by our FsCheck tests, but because we don't implement a shrinker, query for which our test fails looks like this:

{% highlight javascript %}
car(name: "name of a car")
{
    name
    model
    engine {
        indicator
        power
        displacement
    }
}
{% endhighlight %}

What we want is to get a minimum query which caused tests to be red, which should look like this:

{% highlight javascript %}
car(name: "name of a car") { model }
{% endhighlight %}

To implement shrinker in our case we have to modify our test input type. Previously it looks like this:

{% highlight csharp %}
internal class QueryTest
{
    public readonly string Query;
    public QueryTest(string query) =>
        Query = query;
}
{% endhighlight %}

Now we want this type to contains information about name, arguments, and fields and how to build the whole query. So it would look like this:

<script src="https://gist.github.com/MNie/37f4f9e9215185e92cd5fb6592c2897b.js"></script>

We modified an input test object, right now we have to go to the BuildArb function, which was responsible for building an Arb object. We have to modify it so at the end of a function based on a Gen object it would also accept a shrinker function which would be responsible for minimizing an input data.

{% highlight csharp %}
public static Arbitrary<QueryTest> BuildArb(
    Func<IEnumerable<string>, Gen<TArguments>> fetchArgsFunc
)
{
    ...
    var gen = hasArguments
        ? GenWithArguments(fetchArgsFunc, query, graphType, container)
        : GenWithoutArguments(query, graphType, container);

    return Arb.From(gen, Shrink);
}
{% endhighlight %}


As we could see instead of previously used `gen.ToArbitrary`, right now we use an `Arb.From` function which also accepts a shrinker parameter. How does shrinker function look like?

{% highlight csharp %}
public static IEnumerable<QueryTest> Shrink(QueryTest qt)
{
    var count = qt.Fields.Count();
    if (count == 1)
        yield break;
    for(var i = 0; i < count; ++i)
        yield return new
            QueryTest(
                qt.Name,
                qt.Arguments,
                qt.Fields.Where((_, index) => index != i)
            );
}
{% endhighlight %}

Implementation of a shrinker is pretty straightforward. The first thing that comes to mind is a return type which is an IEnumerable of QueryTest instead of simply QueryTest.  This is because we create k combinations of an input against which our test would be run in n shrink phases. If we go deeply into implementation we could see that we want to generate some "shrunk examples" till the query would contain a single field (graphQL expect at least one field in a query). Then we go through all fields and we remove one of them. I think this example would explain it.

As we remember the field `model` in a `car` was resolved incorrectly. FsCheck produces following test case in the first phase of a test (shrink = 0). The test was run against the following query.

{% highlight javascript %}
car(name: "name of a car")
{
    name
    model
    engine {
        indicator
        power
        displacement
    }
}
{% endhighlight %}

In the next phase (shrink = 1) tests were run against:

{% highlight javascript %}
car(name: "name of a car")
{
    model
    engine {
        indicator
        power
        displacement
    }
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    name
    engine {
        indicator
        power
        displacement
    }
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    name
    model
}
{% endhighlight %}

next phase (shrink = 2):

{% highlight javascript %}
car(name: "name of a car")
{
    engine {
        indicator
        power
        displacement
    }
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    model
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    engine {
        indicator
        power
        displacement
    }
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    name
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    model
}
{% endhighlight %}

{% highlight javascript %}
car(name: "name of a car")
{
    name
}
{% endhighlight %}

As we could see after two phases (shrink = 2) we could gather a minimum query which produces failure:

{% highlight javascript %}
car(name: "name of a car")
{
    model
}
{% endhighlight %}

![result](https://mnie.github.com/img/21-05-2018-GraphQLShrink/result.png)

Of course, a solution like that has some drawbacks.  For example what with very complex fields? With such minimal implementation of a shrinker, we could get a query with a single field, but this field could have a lot of fields for example:

{% highlight javascript %}
car(name: "name of a car") { a { b { c { d { e { … } } } } } }
{% endhighlight %}

Although in our case such simple implementation is enough and it helps gather an information what exactly is wrong and also leverage the time needed to spend on manual checking of a query.

Thanks for reading! :)
