---
title: Upon Reflection
layout: post
published: true
---

I've been spending most of my time recently performance profiling and optimising one of our main services. Part of its job involves exposing data objects through a generic interface. The abstraction allows public properties on each object to be read using an untyped indexer. To do this we look up all the properties of each data object via reflection and cache them so later reads and writes are quicker. Turns out this caching is not enough. With many short-lived objects we ended up almost half our time in some performance critical paths just populating this reflection cache. The initial implementation is naïve, but what can be done about it?

To get started imagine that we have the following two C# classes:

```csharp
public class Base
{
    private IDictionary<string, PropertyInfo> _props;
    
    public Base()
    {
        _props = GetProps(GetType());
    }
    
    IDictionary<string, PropertyInfo> GetProps(Type t) =>
       t.GetMembers()
       .Where(mi =>
           mi.MemberType == MemberTypes.Property)
       .Cast<PropertyInfo>()
       .ToDictionary(prop => prop.Name);
}

public class User : Base
{
    public string Name { get; }
    public int Age { get; }
}
```

Each time a new `User` is allocated we will populate the cache of properties we can read and write (`Name`, `Age`, etc.). Obviously this  is a tad less than efficient. The properties on a `User` aren't going to change between instantiations. An obvious optimisation here is to cache these results. Lets update our `Base` class a little:

```csharp
public class Base
{
    public Base()
    {
        _props = TypeProps.GetOrAdd(
            GetType(),
            GetProperties);
    }

    // ... As before ... 

    static ConcurrentDictionary<
        Type,
        IDictionary<string, PropertyInfo>> TypeProps =
        new ConcurrentDictionary<
            Type,
            IDictionary<string, PropertyInfo>>();
}
```

Here we've allocated a static dictionary of `Type` to `IDictionary` of properties. For each different type we encounter a new entry will be added to the dictionary containing the property information for that type. As you might expect this speeds things up greatly.

Benchmarking the before and after you'll find that this per-type cache rather than per-instance cache improves the speed of allocating an object by a few orders of magnitude. Allocation went from 1,100ns to 40ns [in my own benchmarking][bench]. It's tempting to claim the win and leave things there, however we can do more. Enter the [Curiously Recurring Template Pattern][crtp]. Instead of allocating our own dictionary of type to properties we'll use C#'s generics and static members to get the language to do this caching for us.

The idea is pretty simple. By making our base class take a type parameter we can lookup all the properties on that type at static initialisation time using `typeof`:

```csharp
public class Base<T>
    where T : Base<T>
{
    static IDictionary<string, PropertyInfo> Props = GetProps(typeof(t));
}
```

We then update each of our derived classes to pass itself in as the template parameter:

```csharp
sealed class User : Base<User>
{
    // ... As Before ...
}
```

With the properties now just stored in a `static` there's no need to look up the correct dictionary at runtime using `GetType`. This update then gets you another order of magnitude in allocation speed; from 40ns to 4ns in my testing.

Another important thing to note in this is the reduction in memory churn for the garbage collector too. The allocation size for the final `User` object is just the object header, method table and the two properties. In the contrived example of the benchmark this results in a 3x reduction in the number of garbage collections.

It's not all rainbows however. When using this pattern a lot of the benefits of object inheritance are lost. Deeper inheritance hierarchies become more of a faff requiring an extra type parameter to be passed up the chain. Adding this pattern into an existing codebase will require a lot of tedious changes.

In conclusion the Curiously Recurring Template Pattern can provide powerful opportunities for optimisation, but don't use it unless profiling suggests you need that extra boost!

[bench]: https://gist.github.com/iwillspeak/2f97766aef7a1f44f5c4feb89f65ffd8#gistcomment-2347002
[crtp]: https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern