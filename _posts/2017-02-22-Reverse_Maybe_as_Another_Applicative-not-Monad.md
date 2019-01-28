---
title: Reverse Maybe as Another Applicative-not-Monad
published: true
excerpt:
    Every time somebody wants to show the difference between Applicative and
    Monad,
    they always pick ZipList, which is an Applicative but not a Monad.
    I recently found yet another example which is less common.
tags: haskell, applicative, monad
---

## Applicative vs. Monad

Every time somebody wants to show the difference between _[Applicative](https://hackage.haskell.org/package/base/docs/Control-Applicative.html#t:Applicative)_ and _[Monad](https://hackage.haskell.org/package/base/docs/Control-Monad.html#t:Monad)_, they always pick _[ZipList](https://hackage.haskell.org/package/base/docs/Control-Applicative.html#t:ZipList)_, which is an _Applicative_ but not a _Monad_.

I recently found yet another example which is less common.

## First'

Consider the following type:

```haskell
newtype First' a b = First' (Maybe a)
```

It is bivariant (both covariant and contravariant) in its tail argument `b`. Compiler sees this clearly:

```haskell
{-# LANGUAGE DeriveFunctor #-}
newtype First' a b = First' (Maybe a)
    deriving (Functor)
```

I leave writing non-derived _Functor_ instance to you as an excercise.

Please note that this functor ignores its argument completely, unlike pure _Maybe_.

The name _First'_ isn't a coincidence. We're going to use semantics of _[Data.Monoid.First](https://hackage.haskell.org/package/base/docs/Data-Monoid.html#t:First)_ here. Namely, _First_ allows to _mappend_ two _Maybe_ values without descending into them:

```haskell
λ> Nothing <> Just 'A'

<interactive>:
    No instance for (Monoid Char) arising from a use of ‘<>’
    ...
```

```haskell
λ> First Nothing <> First (Just 'A')
First {getFirst = Just 'A'}
```

## Monoid

Let's write _Monoid_ for our _First'_ with the same semantics:

```haskell
instance Monoid (First' a b) where
    mempty = First' Nothing
    x@(First' (Just _)) `mappend` _ = x
    _ `mappend` y = y
```

Let's try it in repl:

```haskell
λ> First' Nothing <> First' (Just 'A')
First' (Just 'A')
```

It works!

## Applicative

Let's go next level. We're going to write _Applicative_ instance with this semantics. We also can reuse monoid instance.

```haskell
instance Applicative (First' a) where
    pure _ = First' Nothing
    First' x <*> First' y = First' x <> First' y
```

Check:

```haskell
λ> First' Nothing <*> First' (Just 'A')
First' (Just 'A')
λ> First' Nothing *> First' (Just 'A')
First' (Just 'A')
```

## Reverse _Maybe_

Why this is a reverse _Maybe_?

_Maybe_ semantics may be expressed as:

  - _Nothing_ -> failure, stop
  - _Just x_ -> continue with _x_

```haskell
λ> Nothing *> Just 1
Nothing
λ> Just 1 *> Just 2
Just 2
```

_First'_ semantics is:

  - _Nothing_ -> continue
  - _Just x_ -> result is _x_, stop

```haskell
λ> First' Nothing *> First' (Just 2)
First' (Just 2)
λ> First' (Just 1) *> First' (Just 2)
First' (Just 1)
```

## Monad?

_First'_ doesn't seem to be a _Monad_ though. But I'm too lazy to prove it. If you can provide such a proof, please let me take a look at it.

## Const


Vladislav Zavialov [noted](https://twitter.com/int_index/status/834126005577060353) that _First'_ is merely

```haskell
type First' a b = Const (First a) b
```

where _First_ is from _Data.Monoid_.

Indeed,

```haskell
λ> Const (First Nothing) *> Const (First (Just 2))
Const (First {getFirst = Just 2})
λ> Const (First (Just 1)) *> Const (First (Just 2))
Const (First {getFirst = Just 1})
```

And _Const_ is yet another interesting Applicative-not-Monad example.
