---
layout: post
title: Testing GraphQL queries with FsCheck library - Union GraphTypes
description: "GraphQL essentially allows users to define queries from a front-end to gather concrete data. So as it looks it is completely different than a standard REST endpoint. GraphQL differs queries which only gather data from those which also mutates them. In this article, I want to concentrate on queries which only gather some data and how to test them via FsCheck and F#. This articles is a continuation of previous post: https://www.mnie.me/2018-05-21-graphQLTestingPart2/"
thumb_image: "documentation/sample-image.jpg"
tags: [fsharp, graphql, cleancode, tests, tdd, automatedtests, patterns]
---

Hi,

Todays once more I want to focus on testing GraphQL queries via FsCheck library. This post is a continuation of previous two posts ([first](https://www.mnie.me/2018-05-07-graphQLTesting/), [second](https://www.mnie.me/2018-05-21-graphQLTestingPart2/)). Some time ago I changed one of the queries in our project, so it could accept as a value for one field value A or B. I decided to use for that an UnionGraphType which is available in GraphQL. Implementation of a car query and graph types from previous articles looks as follows.

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

Query:

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

In our case the use of an UnionGraphType could be named as an additional option available in a car. To visualize it better, we could assume that car could have some media system which could be a CD radio, Android radio, Cassette radio. Above three classes we implement as following GraphTypes.

{% highlight csharp %}
public class CDRadio
{
    public bool discChanger { get; set; }
    public string Model { get;set; }
}

public class AndroidRadio
{
    public int systemVersion { get; set; }
    public string Model { get;set; }
}

public class CassetteRadio
{
    public string Model { get;set; }
}

public class CDRadioGraphType : ObjectGraphType<CDRadio>
{
    public CDRadioGraphType()
    {
        this.Name = "CD";

        this.Field<BooleanGraphTyp>(
            "discChanger",
            resolve: ctx => ctx.Source.DiscChanger
        );
        this.Field<StringGraphType>(
            "model",
            resolve: ctx => ctx.Source.Model
        );
    }
}

public class AndroidRadioGraphType : ObjectGraphType<AndroidRadio>
{
    public AndroidRadioGraphType()
    {
        this.Name = "Android";

        this.Field<IntGraphType>(
            "systemVersion",
            resolve: ctx => ctx.Source.SystemVersion
        );

        this.Field<StringGraphType>(
            "model",
            resolve: ctx => ctx.Source.Model
        );
    }
}

public class CassetteRadioGraphType : ObjectGraphType<CassetteRadio>
{
    public CassetteRadioGraphType()
    {
        this.Name = "Cassette";

        this.Field<StringGraphType>(
            "model",
            resolve: ctx => ctx.Source.Model
        );
    }
}
{% endhighlight %}

Definition of a UnionGraphType strapping all these types looks like this:

{% highlight csharp %}
public class MultimediaSystem : UnionGraphType
{
    public MultimediaSystem(
        CDRadioGraphType cd,
        AndroidRadioGraphType android,
        CassetteRadioGraphType cassette
    )
    {
        this.Type<CDRadioGraphType>();
        this.Type<AndroidRadioGraphType>();
        this.Type<CassetteRadioGraphType>();
    }
}
{% endhighlight %}

Right now our custom UnionGraphType is implemented, although we would like that our queries would be still testable by FsCheck code written in previous posts. So to do that, we have to make a couple of changes so that query would be generated correctly. Right now the full query with information about the multimedia system in a car looks like this:

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
    mediaSystem {
        __typename
        ... on CD {
            discChanger
            model
        }
        ... on Android {
            systemVersion
            model
        }
        ... on Cassette {
            model
        }
    }
}
{% endhighlight %}

As we could see a query for gathering fields specified as a union type are different than a query for a simple field. Instead of `field`, we have to write `... on field`. Because of that while scanning all available fields we have to extract those fields which are defined as a UnionGraphType. Important is that UnionGraphType doesn't inherit from IComplexGraphType, this is why test without any change would fail when trying to produce a query. So right now the first thing when scanning the available fields is to check if the field is a UnionGraphType or not. If it is, we want to collect all available GraphTypes for this Union and for every of this type generate a part of a query but also keep the actual behavior for fields not defined as unions. Here is a piece of code which is responsible for serializing all the fields while they are scanning:

<script src="https://gist.github.com/MNie/6bd8fc7f851ad865544dcaa8b8152b73.js"></script>

Thanks to the above fixes we were able to test also queries which contain a UnionGraphTypes as fields. The only dilemma here is that we ask in a query for all possible types specified in a union. The solution to that is to run some warmup or startup query which would infer based on a `__typename` field, what type is available. But in our situation, this problem (to ask for all possible types) is rather small, and a solution to this would be overwhelming, so we decide to keep it as simple as possible.

Thanks for reading!
