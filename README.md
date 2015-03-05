# purescript-aff

An asynchronous effect monad for PureScript.

The moral equivalent of `ErrorT (ContT Unit (Eff (async :: Async | e)) a`, for effects `e`.

`Aff` lets you say goodbye to monad transformers and callback hell!

# Example

```purescript
main = launchAff $ 
  do response <- Ajax.get "http://foo.bar"
     liftEff $ trace response.body
```

# Getting Started

## Installation

```
bower install purescript-aff
```

## Introduction

An example of `Aff` is shown below:

```purescript
deleteBlankLines path =
  do  contents <- loadFile path
      let contents' = S.join "\n" $ A.filter (\a -> S.length a > 0) (S.split "\n" contents)
      saveFile path contents'
```

This looks like ordinary, synchronous, imperative code, but actually operates asynchronously without any callbacks (error handling is baked in so you only deal with it when you want to).

The library contains instances for `Semigroup`, `Monoid`, `Apply`, `Applicative`, `Bind`, `Monad`, `Alt`, `Plus`, `MonadPlus`, `MonadEff`, and `MonadError`. These instances allow you to compose `Aff`-ectful code as easily as `Eff`, as well as interop with existing `Eff` code.

## Escaping Callback Hell

Hopefully, you're using libraries that already use the `Aff` type, so you don't even have to think about callbacks!

If you're building your own library, or you have to interact with some native code that expects callbacks, then *purescript-aff* provides a `makeAff` function you can use:

```purescript
makeAff :: forall e a. ((Error -> Eff e Unit) -> (a -> Eff e Unit) -> EffA e Unit) -> Aff e a
```

This function expects you to provide a handler, which should call a user-supplied error callback or success callback with the result of the asynchronous computation.

For example, let's say we have an AJAX request function that expects a callback:

```purescript
foreign import ajaxGet """
function ajaxGet(callback) { // accepts a callback
  return function(request) { // and a request
    return function() { // returns an effect
      doNativeRequest(request, function(response) {
        callback(response)(); // callback itself returns an effect
      });
    }
  }
}
""" :: forall e. (Response -> Eff e Unit) -> Request -> EffA e Unit
```

We can wrap this into an asynchronous computation like so:

```purescript
ajaxGet' :: forall e. Request -> Aff e Response
ajaxGet' req = makeAff (\error success -> ajaxGet success req)
```

This eliminates callback hell and allows us to write code simply using `do` notation:

```purescript
do response <- ajaxGet' req
   liftEff $ trace response.body
```

## Converting from Eff

All purely synchronous computations (`Eff`) can be converted to `Aff` computations with `liftEff` defined in `Control.Monad.Eff.Class` (see [here](https://github.com/paf31/purescript-monad-eff)).

```purescript
import Control.Monad.Eff.Class

liftEff $ trace "Hello world!"
```

This lets you write your whole program in `Aff`, and still call out to synchronous `Eff` code.

If your `Eff` code throws exceptions (`err :: Exception`), you can remove the exceptions using `liftEff'`
to bring the exception to the value level as an `Either Error a`:

```purescript
do e <- liftEff' myExcFunc
   liftEff $ either (const $ trace "Oh noes!") (const $ trace "Yays!") e
```

## Failure

The `Aff` monad has error handling baked in, so ordinarily you don't have to worry about it.

If you want to attempt a computation but recover from failure, you can use the `attempt` function:

```purescript
attempt :: forall e a. Aff e a -> Aff e (Either Error a)
```

This returns an `Either Error a` that you can use to recover from failure.

```purescript
do e <- attempt $ Ajax.get "http://foo.com"
   liftEff $ either (const $ trace "Oh noes!") (const $ trace "Yays!") e
```

### Alt

Because `Aff` has an `Alt` instance, you may also use the operator `<|>` to provide an alternative computation in the event of failure:

```purescript
do result <- Ajax.get "http://foo.com" <|> Ajax.get "http://bar.com"
   return result
```

### MonadError

`Aff` has a `MonadError` instance, which comes with two functions: `catchError`, and `throwError`.

These are defined in [purescript-transformers](http://github.com/purescript/purescript-transformers).
Here's an example of how you can use them:

```purescript
do resp <- (Ajax.get "http://foo.com") `catchError` (\e -> pure defaultResponse)
   if resp.statusCode != 200 then throwError myErr 
   else pure resp.body
```

Thrown exceptions are propagated on the error channel, and can be recovered from using `attempt` or `catchError`.

# Documentation

[MODULES.md](MODULES.md)
