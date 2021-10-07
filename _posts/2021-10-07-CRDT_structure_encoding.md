---
title: CRDT Structure Encoding
published: true
excerpt:
    The problem any application may encounter when using LWW-based structures
    and an OR-set-based solution to it.
tags: CRDT
---

When we write an application
that employs CRDT to deal with distributed mutable data,
we sooner or later face a problem of
concurrent mutation of objects referencing other objects.
Here I show what are indirect consequeces of using LWW for structures
and how OR-Set copes with them.

This article uses [RON data model and syntax](https://replicated.cc/)
to describe distributed objects.

## Sample problem

Let's take an example of such object:

```rs
blog_post = {
    author_id: 1324632589,
    title: "yard accident",
    text: "sick charge joined mood...",
}
```

## LWW approach

This is how we may represent it as a set of CRDT (RON) objects:

```rs
// text
@100+Alpha :rga,
    "sick charge joined mood...";

// blog_post
@200+Alpha :lww,
    'author_id' 1324632589,
    'title' 'yard accident',
    'text' 100+Alpha;
```

Now we can mutate it. Let replica Beta change its title:

```rs
// operation
@300+Beta :200+Alpha 'title' 'fire principle';

// result
    @200+Alpha  :lww,
    @201+Alpha  'author_id' 1324632589,
//  @202+Alpha  'title' 'yard accident', // shadowed by LWW
    @300+Beta   'title' 'fire principle',
    @203+Alpha  'text' 100+Alpha;
```

## The new field issue

Now let's suppose the schema of data in our happlication has changed,
and we want to add a new field and also want to keep all existing data.
Let the new field be `tags` containing set of strings,
and this set must be mutable, so we create an OR-Set object for it.
If there's no such field in a blog post, the tag set is considered empty.

```rs
// replica Gamma creates a new set for tags
@400+Gamma :set,
    'storm',
    'report';

// and puts the reference to it to the 'tags' field;
@500+Gamma :200+Alpha 'tags' 400+Gamma;
```

Independently of Gamma, replica Delta wants to do the same,
using an old version
(we can describe it with version vector `203+Alpha 300+Beta`) of the object,
which doesn't have `tags` yet.
So, Delta creates another OR-Set object.

```rs
// replica Delta creates a new set for tags
@400+Delta :set,
    'storm',
    'face';

// and puts the reference to it to the 'tags' field;
@500+Delta :200+Alpha 'tags' 400+Delta;
```

Then, they exchange ops and merge, and come to a single version:

```rs
    @200+Alpha  :lww,
    @201+Alpha  'author_id' 1324632589,
    @300+Beta   'title' 'fire principle',
    @203+Alpha  'text' 100+Alpha,
//  @500+Delta  'tags' 400+Delta, // shadowed by LWW
    @500+Gamma  'tags' 400+Gamma;
```

Oops!
The reference to the object `400+Delta` and all tags added by Delta is lost!

## OR-Map-structure solution

OR-Map-structure represent an object with a
[multimap](https://en.wikipedia.org/wiki/Multimap) with string (or UUID) keys.

In its turn, an OR-multimap (with keys of type K and values of type V)
is an OR-Set [whose items are tuples (K, V)].

Let's walk through the same operations, but with OR-Map (or set, on low level):

Initial representation is the same, except the type:

```rs
// text
@100+Alpha :rga,
    "sick charge joined mood...";

// blog_post
@200+Alpha :set,
    'author_id' 1324632589,
    'title' 'yard accident',
    'text' 100+Alpha;
```

And the final state of oplog is almost the same:

```rs
    @200+Alpha  :set,
    @201+Alpha              'author_id' 1324632589,
//  @202+Alpha              'title' 'yard accident', // tombstoned by 300+Beta
    @300+Beta   :202+Alpha,
    @301+Beta               'title' 'fire principle',
    @203+Alpha              'text' 100+Alpha,
    @500+Delta              'tags' 400+Delta,
    @500+Gamma              'tags' 400+Gamma;
```

But now we don't lose anything!

Raw object view:

```rs
blog_post = {
    author_id: {1324632589},
    title: {"fire principle"},
    text: {"sick charge joined mood..."},
    tags: {
        {"report", "storm"},
        {"face", "storm"},
    },
}
```

Each key has a set of values, because the we based our structure on a multimap.

Here we also can voluntarily drop some information from CRDT
using application-level knowledge on how fields can be mutated:

```rs
blog_post.author_id.last()
= 1324632589

blog_post.title.last()
= "fire principle"
```

What can we do with sets of complex CRDT objects?
Do you remember CRDT is a semigroup? (RON objects are even monoids!)
Yes, we can merge arbitrary objects!

```rs
blog_post.text.merge()
= "sick charge joined mood..."

blog_post.tags.merge()
= {"face", "report", "storm"}
```

Now all needed information is saved. And we have a convenient way to access it.

And it may be even more convenient!
If we declare field access strategies in a separate place:

```rs
struct BlogPost {
    author_id: LWW(Integer),
    title: LWW(String),
    text: Merge(RgaString),
    tags: Merge(ORSet),
}

db.load<BlogPost>('200+Alpha').decode()
= {
    author_id: 1324632589,
    title: "fire principle",
    text: "sick charge joined mood...",
    tags: {"face", "report", "storm"},
}
```
