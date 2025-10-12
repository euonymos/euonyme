---
title: Use Show and Read judiciously
---

## Read instances: typechecks, but doesn't work

One of the commonly acknowledged advantages of Haskell and its powerful type system is the ability to refactor programs in a safe manner. Indeed, by using thoroughly defined types and leveraging the ease of introducing zero run-time cost `newtype`s, we can eliminate a whole class of errors caused by representing many different things by means of one and the same type.

That being said, we should exercise extra precautions when it comes to munging data at the boundaries of an application, where everything likely gets a representation based on very basic types, most often `String` (or more probably `Text`).

Consider the following example of reading an environment variable, taken from a real production codebase:

```haskell
pgPort :: Word16
pgPort = fromMaybe 5432 $ readMaybe =<< lookupEnv "PG_PORT"

lookupEnv :: String -> Maybe String
lookupEnv = ...
```

Someone had to move from `Word16` to `String` here for some reason and came up with these lines, having done just mechanical changes:

```haskell
pgPort :: String
pgPort = fromMaybe "5432" $ readMaybe =<< lookupEnv "PG_PORT"
```

While this code compiles, it doesn't work as expected. Reading a `String` will fail unless the value is properly quoted, but no one will expect this to be the case when setting the value of an environment variable. As a result, you will most probably get the default value regardless of what has been set:

```haskell
>>> readMaybe "5432" :: Maybe String
Nothing
>>> readMaybe "\"5432\"" :: Maybe String
Just "5432"
```

Remembering that `String` is effectively a list of `Char`s, we can also specify the value as follows (which is even more weird):

```haskell
λ> readMaybe "['5','4','3','2']" :: Maybe String
Just "5432"
```

Although the fact that we can read `\"5432\"` and even `['5','4','3','2']`, but not `5432` as a `String` can be quite frustrating, the actual implementation of `Read` makes sense since it holds a *social contract* of being a counterpart to `Show`. But let sleeping dogs lie, and let's accept the way `Show` is implemented without further questions. The rationale behind quoting strings becomes obvious when `Text` is used as part of a complex data type:

```haskell
>>> data Person = Person String (Maybe Int) deriving Show
>>> Person "Jon Snow" (Just 42)
Person "Jon Snow" (Just 42)
```

The lesson we learned from this example is that it is intrinsically unsafe to use `Read`, or at least some standard instances, for parsing values. While it might work well when the string representation is aligned with what we expect, it can break abruptly when it is not. And indeed, `Show` and `Read` have been designed as means to debug programs in GHCi. But people will use them for other purposes in the wild, so keep your eyes open!

## Show instances: multi-escaping

Another area that tends to make extensive use of `Show` is logging. Once we expose a logging action in terms of `Show`, which means we handle a value using `show`, we become responsible for cases when someone calls another `show` from a `Text` / `String` instance before our call, leading to somewhat ugly log entries that are tricky to parse correctly:

```haskell
λ> show $ show $ show "some \" payload"
"\"\\\"\\\\\\\"some \\\\\\\\\\\\\\\" payload\\\\\\\"\\\"\""
```

Using `Show` is believed to be highly contagious, and it is hard to get rid of once it has been introduced and spread over the codebase. One of the options to pursue in cases like this might be using reflection to decide whether we have to call `show` again:

```haskell
logMessage :: forall p . (Typeable p, Show p)
           => LogLevel -> payload -> Logger ()
logMessage lvl tag = lift $ LogMessage lvl textPayload
  where
    textPayload :: Text
    textPayload
      | Just HRefl <- eqTypeRep (typeRep @tag) (typeRep @Text  ) = tag
      | Just HRefl <- eqTypeRep (typeRep @tag) (typeRep @String) = toText tag
      | otherwise = show tag
```

So, again, we would be better off not using `Show`, but finding other ways of transforming things to be logged into `Text` / `String` representation.
