---
layout: post
title: Keeping an eye on efficiency
date:  2012-07-14
tags: [programming]
---

<img style="float: right" src="../assets/big-o.png" />

### The good

We live in a great era for computing, we have plenty of computing power, memory,
storage, etc, is often quoted that an iPhone has more computing power than
[NASA](http://www.nasa.gov/) had when they sent the [Apollo
11](http://en.wikipedia.org/wiki/Apollo_11) to the moon. Along with these
advancements in hardware, we get a lot of advancements in software too, in the
means of improved tools, powerful programming languages and libraries which
allow us to create better software in a more rapid, easier fashion.

Things like garbage collection, large API’s and frameworks, like
[Java](http://docs.oracle.com/javase/6/docs/api/) or
[.NET](http://www.microsoft.com/net), [Ruby On Rails](http://rubyonrails.org/),
mind you, some of these aren’t really
[new](http://en.wikipedia.org/wiki/Garbage_collection_\(computer_science\)), but
they have been coming into the mainstream as computers become more powerful and
their cost less prohibitive.

The larger the bricks we have to build, the less visibility we have into
how these bricks work inside, and sometimes, they are so easy to use
that we simply do not pay much attention to what it is that we are
actually doing, we just *glance* a bit on the documentation and go on
our happy, rushed way.

### The bad

This of course, has its cost in terms of efficiency, sometimes in a
noticeable way, sometimes is so seemingly innocuous, so deceptively
tiny, that we simply don’t even know, or don’t notice, or don’t care
about it, but these little things add up and instead of needing a
machine with 4GB of memory, we need 8GB, or instead of being able to run
decently on a low-end CPU, you need a high-end one instead, or you end
up needing a database server with a few zillion-byte hard drives and
gigabytes and gigabytes of memory.

We can of course rationalize this. It’s enterprise software\!, we have a
lot of users\!, we have lots of data\!, and so on and so on.

I am not stating of course that we shouldn’t use the tools we have at
hand, but in order to make the most out of them we have to know them, we
have to use them properly and in the right context, we have to know our
data structures and we have to understand the basics of [analysis of
algorithms](http://en.wikipedia.org/wiki/Analysis_of_algorithms) even if
we program in Perl, Ruby or Java, so we do not run around creating
n-squared algorithms all over the place like [Shlemiel the
painter](http://www.joelonsoftware.com/articles/fog0000000319.html).

To illustrate this point, I am going to share with you something that
happened to me this week.

### ETL

One of my recent tasks at [INgrooves](http://www.ingrooves.com) was
writing an [ETL](http://en.wikipedia.org/wiki/Extract,_transform,_load)
process for product ingestion, these products come from an external
database and we have to convert, validate and ingest into our own. This
was implemented in application (C\#) code that is usable either through
a [CLI](http://en.wikipedia.org/wiki/Command-line_interface) driven by
given keys or running as a background process in our in-house systems
pulling work items from a queue.

This ETL has to be very fast, it has to process many products a day,
either new or updated, and some of these products have lots of data
(particularly classical albums). I and a few other guys in my team went
to great lengths to achieve this goal.

One very key part of the ETL is that in the name of speed we load a
whole product from the external database and convert into a memory
representation to process as a whole, we do the same with the data in
our side, after all, we have to know if this is an updated product or a
new product, we do not want to save rows redundantly so we preload the
whole product that exists currently in our side in order to compare and
process.

In order to process rows side by side, I create a collection that holds
both rows, one may be missing though, if the source row is missing that
means I have an extra row so I have to delete it, if the destination row
is missing then it means this is new data.

The operation I am describing is analogous to a SQL [Full Outer
Join](http://en.wikipedia.org/wiki/Join_\(SQL\)), so I came with this
method:

{% highlight csharp %}
public static IEnumerable<TResult> FullOuterJoin<TOuter, TInner, TKey, TResult>(
    this IEnumerable<TOuter> outer,
    IEnumerable<TInner> inner,
    Func<TOuter, TKey> outerKeySelector,
    Func<TInner, TKey> innerKeySelector,
    Func<TOuter, IEnumerable<TInner>, TResult> outerResultSelector,
    Func<TInner, IEnumerable<TOuter>, TResult> innerResultSelector,
    IEqualityComparer<TResult> resultEqualityComparer)
    where TInner : class
    where TOuter : class
{
    return outer.GroupJoin(
               inner,
               outerKeySelector,
               innerKeySelector,
               outerResultSelector
           ).Union(
               inner.GroupJoin(
                   outer,
                   innerKeySelector,
                   outerKeySelector,
                   innerResultSelector),
               resultEqualityComparer
           );
}
{% endhighlight %}

Although it looks a bit cryptic due to the generics, it is more or less
straightforward, this is what it does:

- Do a group join of the first list with the second using a common key extracted from the items on each collection by `outerKeySelector` and `innerKeySelector`, calling `outerResult` selector with each item in the first list and matching items in the second list (or none).
- Do the same thing just reversed, thus giving you the items in the second list that have no match in the first.
- Do a set union of both results.

Pretty clear uh?

There are so many things wrong with it that I feel really ashamed and now I am punishing myself by making it public.

#### Cluelessness

I am doing the same operation twice, the GroupJoin operation is not very
expensive, it’s O(N) in time complexity, so I figured it wouldn’t be
much of a problem, but it lead me to:


The set union operation is O(N \* M) and kills any hope for
performance.

Because I chose to do two groupings, if there’s a match for an item in
the first list and the second list, that match will appear in both
grouping operations and although we have two different selectors we have
no idea what the code is, so repetitions are very likely and the way to
get rid of them is by using the set union operation.

For most products this wasn’t much of a problem, until I tested with a
product that has 130,536 rows. Yes, 130,536. To perform the Union
operation when both lists were populated (a product update) it had to do
**17,039,647,296** compares. I don’t care if the CPU is 10GHz, this
would be **very** noticeable. (This particular product took 3 hours to
update in my machine, and that’s without almost no DB writes because the
ETL is optimized to write only when something actually changed).

And all this just because I said “meh” when thinking about the two
groupings… I cannot blame the abstractions provided to me by the
framework of course, I can only blame me and my stupidity for shooting
myself in the foot with them.

So here’s the rematch :

#### Final version

{% highlight csharp %}
public static IEnumerable<TResult> FullOuterJoin<TOuter, TInner, TKey, TResult>(
    this IEnumerable<TOuter> outer,
    IEnumerable<TInner> inner,
    Func<TOuter, TKey> outerKeySelector,
    Func<TInner, TKey> innerKeySelector,
    Func<TOuter, TInner, TResult> resultSelector)
    where TInner : class
    where TOuter : class
{
    Dictionary<TKey, TOuter> ok = outer.ToLookup(outerKeySelector).ToDictionary(i => i.Key, i => i.FirstOrDefault());
    Dictionary<TKey, TInner> ik = inner.ToLookup(innerKeySelector).ToDictionary(i => i.Key, i => i.FirstOrDefault());

    HashSet<TResult> result = new HashSet<TResult>();

    foreach (var o in ok.Keys)
    {
        var e = ok[o];
        var d = ik.ContainsKey(o) ? ik[o] : default(TInner);

        result.Add(resultSelector(e, d));

        if (ik.ContainsKey(o))
        {
            ik.Remove(o);
        }
    }

    foreach (var i in ik.Keys)
    {
        var e = default(TOuter);
        var d = ik[i];

        result.Add(resultSelector(e, d));
    }

    return result;
}
{% endhighlight %}

This other version is my redemption for the god awful code that I wrote
originally.

What it does:

- Create lookup tables by the keys provided for both lists. This makes finding things in each collection very fast (I could bet that `GroupJoin` uses this approach too).
- Iterate over the first lookup table, using the key to find a matching element in the second collection or default if none, calling the `resultSelector` with both items and inserting them into a `HashSet`. Then removing the match (if any) from the second lookup table.
- Iterate over the leftovers on the second table (if any) and calling `resultSelector` with them and default() as the item from the first list.
- Return the result which now contains the matches between both list and items without match.

Now, doing things by hand, we can reduce time complexity to O(N). There are
several O(N) loops here, but time complexity stays linear. The core are two
loops which in the worst case (two list without no matches in common) will be
O(N + M) + lookup tables creation. So for the large product before, the time it
takes is reduced massively, as I said, it took 3 hours before this change, after
it, it takes *7:30* minutes. (Not instantaneous, but hey, is an improvement\!).

### Conclusion

Always, always, run a profiler.

Do invest some time learning about algorithms and data structures, this
knowledge will pay off. Something as innocent looking as
`List<T>.Contains()` is O(N) because it has to traverse all the
list to know if the given item is actually in the collection. Stuck a
call to it inside a loop and you got yourself a O(N \* M) algorithm.

Be wary of the dark magic in the API calls you make, try to understand
what they are doing.
