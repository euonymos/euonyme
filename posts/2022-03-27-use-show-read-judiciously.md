---
title: Use Show and Read judiciously
---

## Read instances: typechecks, but doesn't work

One of the commonly admitted advantages of Haskell and its powerful type system is ability to refactor programs in a safe manner. Indeed, by  using thoroughly defined types and just by leveraging the easiness of introducing zero run-time cost `newtype`s, we can eliminate a whole class of errors caused by representing many different things by means of one and the same type.

That being said, we should exercise extra precautions when it comes to munging data at the boundaries of an application, where everything likely gets a representation based on very basic types, most often `String` (or more probably `Text`).

Consider the following example of reading an environment variable, taken from a real production codebase:

```Haskell
pgPort :: Word16
pgPort = fromMaybe 5432 $ readMaybe =<< lookupEnv "PG_PORT"

lookupEnv :: String -> Maybe String
lookupEnv = ...
```

Someone had to move from `Word16` to `String` here for some reason, and came up with these lines, having done just mechanical changes:

```Haskell
pgPort :: String
pgPort = fromMaybe "5432" $ readMaybe =<< lookupEnv "PG_PORT"
```

While this code compiles, it doesn't work as expected. Reading a `String` will fail unless the value is properly quoted, but no one will expect this being the case when setting a value of an environment variable. As a result, you most probably will get the default value regardless what has been set:

```Haskell
>>> readMaybe "5432" :: Maybe String
Nothing
>>> readMaybe "\"5432\"" :: Maybe String
Just "5432"
```

Remembering that `String` is effectively a list of `Char`s, so we also might specify the value as following (which is even more weird):

```
λ> readMaybe "['5','4','3','2']" :: Maybe String
Just "5432"
```

Albeit the fact that we can read `\"5432\"` and even `['5','4','3','2']`, but not `5432` as a `String` can be quite frustrating, the actual implementation of `Read` makes sense since it holds a *social contract* of being a counterpart to `Show`. But let sleeping dogs lie, and let's take the way how `Show` is implemented without further questions. The rational behind quoting strings becomes obvious when `Text` is used as a part of a complex data type:

```Haskell
>>> data Person = Person String (Maybe Int) deriving Show
>>> Person "Jon Snow" (Just 42)
Person "Jon Snow" (Just 42)
```

The lesson we learned from this example is that it is intrinsically unsafe to use `Read` or at least some standard instances for parsing values. While it might work well when the string representation is aligned with what we expect, it can break abruptly when it is not. And indeed, `Show` and `Read` have been designed as means to debug programs in GHCi. But people will use them for other purposes in the wild, so keep your eyes open!

## Show instances: multi-escaping

Another area which tends to make extensive use of `Show` is logging. Once we expose a logging action in terms of `Show`, which means we handle a value using `show`, we become responsible for cases when someone calls another `show` from `Text` / `String` instance before our call, leading to somewhat ugly log entries which are tricky to parse correctly:

```Haskell
λ> show $ show $ show "some \" payload"
"\"\\\"\\\\\\\"some \\\\\\\\\\\\\\\" payload\\\\\\\"\\\"\""
```

Using `Show` is believed to be highly contagious, and it is hard to get rid of it once it has been introduced and spread over the codebase. One of the options to go in cases like that might be using of the reflection to decide whether we have to call `show` again:

```Haskell
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

So, again, we would better not use `Show`, but find other ways of transforming things to be logged into `Text` / `String` representation.