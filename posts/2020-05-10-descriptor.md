---
title: The Descriptor
date: 2020-05-10
abstract:
  Add metadata to record fields, work with higher-kinded data without resorting to Generics and Template Haskell, write safe and efficient FFI code
tags:
  descriptor hkd effective haskell
---
 
One of my goals with this blog is a series self-importantly titled _Effective Haskell_[^alt].
It's a response to the recent [Simple](https://www.simplehaskell.org/)/[Boring](https://www.snoyman.com/blog/2019/11/boring-haskell-manifesto)/[Junior](https://www.parsonsmatt.org/2019/12/26/write_junior_code.html) Haskell movement.
I agree with the sentiment that Haskell is full of footguns, that being smart and being wise are different things, and that type-level spaghetti is still spaghetti.
My issue with the argument is the implication that we're giving something up.
Instead, I'd argue that using "advanced" features like GADTs, Type Families and even type classes is often unnecessary in the first place[^code], and a failure to find a simpler solution.

I'm not sure if I'll ever actually finish a single post in _Effective Haskell_, but what I'm going to do in the meantime is use this blog to collect examples of what I think would classify as Effective Haskell.

[^alt]: Alternate title: _Gospel of `RankNTypes`, the One True Extension_

[^code]: We can actually write incomprehensible code _without_ them!

In this inaugural post, we will be studying a technique (ostensibly) for iterating over record fields and adding metadata to them, without using Template Haskell or Generics.
I'm calling it a _[descriptor](https://en.wikipedia.org/wiki/Data_descriptor)_ because that's what it reminds me of, but if somebody else has named it before, or just knows a more fitting name, please let me know.

## Motivating example

Our initial motivating example is from where I initially stumbled into this technique.
The point is to give an idea of what a good use case for descriptors looks like, we'll tackle the details later.

The goal of the library in this example is to give a way to define graphics pipelines in a safe and efficient way.
The user of the library formulates the arguments to a graphics pipeline on the Haskell side as a data type:
```haskell
data Blinn f = Blinn
  { viewPos       :: f (V3 Float)
  , lightAmbient  :: f (V3 Float)
  , lightDiffuse  :: f (V3 Float)
  , lightSpecular :: f (V3 Float)
  ...
  , shininess     :: f Float
  , mvpMatrices   :: MVP f
  }
```
The shaders (GPU code) are written in a different language, and compiled, externally, at runtime.
So from this Haskell record of shader arguments, the library then needs to:

1. Reject types that the GPU does not support
2. Match up these arguments to the shader code
3. Make sure the types of the variables match
4. Check for missing/unbound/duplicate variables
5. Get the locations of those variables on the GPU
6. Assign initial values
7. Provide a safe way to update the variables at runtime

All of that is handled by `createProgram`, which only requires that the user pass a _descriptor_ of the `Blinn` record:
```haskell
(program, log) <- createProgram "glsl/common.vert" "glsl/blinn.frag" $ \f -> Blinn
    <$> f viewPos       0   "viewPos"
    <*> f lightAmbient  0.2 "light.ambient"
    <*> f lightDiffuse  1   "light.diffuse"
    <*> f lightSpecular 1   "light.specular"
    ...
    <*> f shininess     32  "material.shininess"
    <*> descMvp (\f' -> f (f' . mvpMatrices))
```
The descriptor is everything on the right-hand side of the `$`.
This descriptor contains, for every field, the respective field accessor, an initial value, and the name of the variable in the shader code.
If `createProgram` succeeds it returns a `Program Blinn` value, and now the field accessors of the original struct can e.g. be used as type-safe update functions:
```haskell
withProgram program $ do
  objViewPos    $= x
  (model . mvp) $= modelMat
  (view . mvp)  $= viewMat
  drawMesh mesh
```

The point is this; a descriptor is an ordinary value, that you can have users provide about their data type.
You can then use this value to do complicated things, in a type-safe manner, without the library ever having seen the original type, and without the user having to be aware of the machinery.

What the descriptor ultimately looks like will depend on the application[^blog], but the principles stay the same.
Let's see how they work.

[^blog]: Which is why this works better as a blog post than a library

## Simple Descriptors

Let's say we're writing a library that allows the user to ask for data on the command line.

One approach is to expose a function that, given a label, parsing function, and verification function, asks for and yields a single value:
```haskell
ask :: String -> (String -> Maybe a) -> (a -> Bool) -> IO a
ask label parse check = go where
  go = do
    putStrLn $ "What's your " <> label <> "?"
    parse <$> getLine >>= \case
      Just r | check r -> return r
      _ -> putStrLn "Invalid response" >> go
```

And the user then uses `Applicative` to assemble into a function asking for an entire record:

```haskell
data Person = Person
  { pName :: String
  , pAge  :: Int
  }

askPerson :: IO Person
askPerson = Person
  <$> ask "name" Just      ((>1) . length . words)
  <*> ask "age"  readMaybe (\a -> a >= 18 && a <= 99)
```

This is the basis for our descriptor.
The arguments to `ask` in `askPerson` describe general properties of the fields of `Person` that might be useful in other contexts.
We turn `askPerson` into a descriptor as follows:

1. we factor out the `ask` so it takes a general `field` function as an argument
2. we generalize the `IO` into any `Applicative`
3. we add the respective record field accessor as an argument to `field`

```haskell
descPerson :: Descriptor Person
descPerson field = Person
  <$> field pName "name" Just      ((>1) . length . words)
  <*> field pAge  "age"  readMaybe (\a -> a >= 18 && a <= 99)
```

`Descriptor` is a type synonym.
Its definition is below, but you have all the information to try writing it yourself, if you're looking for an exercise.
You're going to need `RankNTypes`.

```haskell
type Descriptor s = forall m. Applicative m
  => (forall a. (s -> a)
             -> String
             -> (String -> Maybe a)
             -> (a -> Bool)
             -> m a
     )
  -> m s
```

We can of course turn a `Descriptor` back into the original `askPerson`:
```haskell
askDesc :: Descriptor p -> IO p
askDesc desc = desc (const ask)

askPerson :: IO Person
askPerson = askDesc descPerson
```
But what have we _gained_?
Of course, we can now swap out `ask` for a similar function.
More interestingly, because we now have the record field accessor, we can reuse the arguments to `field` and work with _existing_ data.
For example, we can perform _just_ the validation, in a pure way:
```haskell
validate :: Descriptor p -> p -> [String]
validate desc pers = execWriter $ desc $ \field lbl _ p ->
  let a = field pers
   in unless (p a) (tell ["Invalid " <> lbl]) $> a
```
```
λ> validate descPerson (Person "aa" 45)
["Invalid name"]
```
Or, we can enumerate all the fields in a descriptor:
```haskell
fields :: Descriptor p -> [String]
fields desc = execWriter $ desc $ \_ lbl _ _ ->
  tell [lbl] $> undefined
```
```
λ> fields descPerson
["name", "age"]
```

And that's a descriptor; a function applied to each field of a record, with some arguments, polymorphic over any applicative.
What arguments `field` depends on what you use the descriptor for, but this outlines the general idea.
It's a pretty neat trick, but unfortunately, there are some issues here:

1. We can give nonsensical `Descriptor`s that still type check:
```haskell
descNonsense :: Descriptor Person
descNonsense _ = pure $ Person "太郎" 3
```
2. We need an unfortunate `undefined` to make the `fields` definition above type check.
3. `validate` is a bit contrived; only outputting a list of invalid fields is hard to deal with safely.

Where this technique really comes into its own, however, is when using _Higher-Kinded Data (HKD)_.

## Descriptors with Higher-Kinded Data

[Higher-Kinded Data](https://reasonablypolymorphic.com/blog/higher-kinded-data/) is a pattern where you parameterize record fields over some functor, like this:

```haskell
data HPerson f = HPerson
  { hName :: f String
  , hAge  :: f Int
  }
```
With HKD, `HPerson Identity` is equivalent to the original `Person` record, but we also get `HPerson Maybe` that might have missing fields, `HPerson (Const a)` that has a value of type `a` for every field, etc.

We can apply the idea of the descriptor to HKD almost verbatim.
Our new `descPerson` and `askDesc` look pretty much the same at the term level:

```haskell
descHPerson :: HDescriptor HPerson
descHPerson field = HPerson
  <$> field hName "name" Just ((> 1) . length . words)
  <*> field hAge "age" readMaybe (\a -> a > 18 && a < 99)

askHDesc :: HDescriptor s -> IO (s Identity)
askHDesc desc = desc $ \_ lbl parse check -> Identity <$> ask lbl parse check
```

In `HDescriptor s` our `s` is now polymorphic over some base functor.
This is its type:

```haskell
type HDescriptor s = forall m f. Applicative m
  => (forall a. (forall g. s g -> g a) -- or, equivalently, Field s a, see below
             -> String
             -> (String -> Maybe a)
             -> (a -> Bool)
             -> m (f a)
     )
  -> m (s f)
```

Let's revisit the issues with the non-HKD approach.

1. `HDescriptor` cannot choose the underlying functor, it _has_ to use `field` to construct it.
We can no longer construct a nonsensical `HDescriptor` without explicitly using `undefined`.
2. We can write `fields` using `Proxy` instead of `undefined`:
```haskell
hfields :: HDescriptor p -> [String]
hfields desc = execWriter $ desc $ \_ lbl _ _ -> tell [lbl] $> Proxy
```
3. We can now use our validation function to check an existing record "in-place", rather than only outputting a list of wrong fields.
Compare this type to that of `validate`:
```haskell
hvalidate :: HDescriptor s -> s Identity -> s Maybe
hvalidate desc s = runIdentity $ desc $ \f _ _ check ->
  f s <&> (\a -> if check a then Just a else Nothing)
```

Here's something that we couldn't do at all before.
Imagine that we get a `HPerson (Const String)` from, say, a web form.
We can then use the `HDescriptor` to parse and check each field individually.
```haskell
hParseCheck :: HDescriptor s -> s (Const String) -> s Maybe
hParseCheck desc s = runIdentity $ desc $ \f _ parse check -> pure $
  case parse $ getConst (f s) of
    Just r | check r -> Just r
    _ -> Nothing
```

### HKD type classes

When you use HKD, you typically want to be able to `map`/`traverse`/`<*>` the fields of your record.
There are libraries like [`higgledy`](https://hackage.haskell.org/package/higgledy), [`barbies`](https://hackage.haskell.org/package/barbies), [`barbies-th`](https://hackage.haskell.org/package/barbies-th), or [`hkd`](https://hackage.haskell.org/package/hkd) that help you derive the required instances (and other nice things).
We can show that a descriptor gives you the same power:
```haskell
dmap :: HDescriptor s ->
  (forall a. f a -> g a) -> s f -> s g
dmap desc fn s = runIdentity $
  desc $ \f _ _ _ -> pure $ fn (f s)

dtraverse :: Applicative m => HDescriptor s ->
  (forall a. f a -> m (g a)) -> s f -> m (s g)
dtraverse desc fn s =
  desc $ \f _ _ _ -> fn (f s)

dpure :: HDescriptor s ->
  (forall a. f a) -> s f
dpure desc a = runIdentity $
  desc $ \_ _ _ _ -> pure a

dliftA2 :: HDescriptor s ->
  (forall x. f x -> g x -> h x) -> s f -> s g -> s h
dliftA2 desc fn sf sg = runIdentity $
  desc $ \f _ _ _ -> pure $ fn (f sf) (f sg)
```

This doesn't necessarily mean that descriptors compete with the libraries above.
The actual use cases are different, I think descriptors work best as an interface between a library and its users.

## Structs and FFI

Briefly, before we continue: a record field accessor of an HKD has type `forall f. s f -> f a`.
To avoid having to quantify the `f` every time, we're going to assign it a type signature:
```haskell
type Field s a = forall f. s f -> f a
```
For example, `hName :: Field HPerson String` and `hAge :: Field HPerson Int`.

### Updating a single field
One of the issues with normal `Storable`-based FFI is that, even if you define a `Storable` instance for a user-defined struct, you cannot perform any field-wise updates on it.
With HKD we can emulate it however, as follows:
```haskell
data MyStruct f = MyStruct
  { versionMajor        :: f Int
  , versionMinor        :: f Int
  , frictionCoefficient :: f Double
  , baconNumber         :: f Word8
  }

data SPtr struct = SPtr
  { sBase    :: Ptr ()
  , sOffsets :: struct (Const Int)
  }

setField :: Storable a => SPtr struct -> Field struct a -> a -> IO ()
setField (SPtr base offsets) field = poke ptr
  where
    ptr = plusPtr base . getConst . field $ offsets
```
As you can see, the trick is to use `MyStruct (Const Int)` to store the offset of every field.
We can then update a single field using
```haskell
setField ptr baconNumber 1

-- Or, if you want to get fancy,
let ($=) :: Storable a => Field struct a -> a -> ReaderT (SPtr struct) IO ()
    ($=) = ...
flip runReaderT ptr $ do
  versionMajor $= 2
  versionMinor $= 1
```
The field accessors of `MyStruct` now double as field accessors for our _foreign_ struct.
I'm leaving `getField` as an exercise, but it works the same way.

### Constructing the `SPtr`
Where does the `SPtr` actually come from?
As you might have guessed, we can make one with a descriptor.
```haskell
ptr <- newSPtr $ \field -> MyStruct
  <$> field versionMajor        1
  <*> field versionMinor        9
  <*> field frictionCoefficient 0.9
  <*> field baconNumber         0
```
The second argument to `f` is the initial value of each field.

`newSPtr` first traverses the constructor, creating a record of the sizes for each field.
It then `malloc`s the total size, and assigns each field its initial value:
```haskell
newSPtr :: SDescriptor s -> IO (SPtr s)
newSPtr desc = do
  base <- mallocBytes size
  desc $ \f a -> poke (plusPtr base . getConst . f $ offsets) a $> Proxy
  pure (SPtr base offsets)
  where
    (offsets, size) = flip runState 0 $
      desc $ \_ a -> state (\s -> (Const s, s + sizeOf a))
```
What makes `SDescriptor` different from our previous descriptors is that it has a type class constraint on the `field` function:
```haskell
type SDescriptor struct = forall m f. Applicative m
  => ( forall a. Storable a
              => Field struct a
              -> a
              -> m (f a)
     )
  -> m (struct f)
```
This means that, as soon as one of the fields of `MyStruct` is _not_ `Storable`, you cannot write a `SDescriptor` for it.
Conversely, the existence of the `SDescriptor MyStruct` automatically proves that every field of `MyStruct` is `Storable`.
For example, you could not add a `String` field to `MyStruct`.
That's not a fundamental issue, you might find some ideas for how to deal with that in the section on arrays below.

For additional exercises, you could try adding `setAll :: struct Identity -> IO ()`, `setSome :: struct Maybe -> IO ()`, and `getAll :: IO (struct Identity)` as fields to `SPtr`.

### Nested descriptors

The initial example already showed an example of how descriptors can be nested, but it might be a good idea to briefly revisit it.
The data definition is fairly straightforward, no different from how you would normally do it with HKD:
```haskell
data MySuperStruct f = MySuperStruct
  { someInt :: f Int
  , nestedData :: MySubStruct f
  }
```

As for the descriptor itself, you simply call the descriptor for the nested struct in the place it occurs, but you'll have to prepend the record field accessor as follows:
```haskell
descMySuperStruct :: SDescriptor MySuperStruct
descMySuperStruct field = MySuperStruct
    <$> field someInt 1
    <*> descMySubStruct (\f -> field (f . nestedData))
```

### Arrays
As a final thought, let's think about how to approach structs that contain arrays.
Like with everything here, there are multiple ways to tackle it, and my goal is just to show that it is possible.

The trick here is to give our records _two_[^arr] functor parameters:
```haskell
data Image fArr fPrim = Image
  { imgW    :: fPrim Int
  , imgH    :: fPrim Int
  , imgData :: fArr Word8
  }
```
Correspondingly, our descriptor now takes two function arguments, with the one for arrays taking an extra one indicating the size:
```haskell
myArrStructDescriptor :: ArrDescriptor MyStructWithArrays
myArrStructDescriptor array field = MyStructWithArrays
  <$> field imgW 99
  <*> field imgH 99
  <*> array imgData (99 * 99 * 3) 0
```
I'll give the type of `ArrDescriptor` below for completeness' sake, but even more than before it's not about the specifics of this approach, but the general idea; that you can add `f`s to correspond to different interfaces, and add a way to construct each in your descriptor.
In this case the difference is between primitive updates in `fPrim` and indexed updates in `fArr`, but you could, for example, also have read/write-only fields.
```haskell
type ArrDescriptor struct = forall m fArr fField. Applicative m
  => ( forall a. Storable a
              => (forall gArr gField. struct gArr gField -> gArr a)
              -> Int
              -> a
              -> m (fArr a)
     )
  -> ( forall a. Storable a
              => (forall gArr gField. struct gArr gField -> gField a)
              -> a
              -> m (fField a)
     )
  -> m (struct fArr fField)
```

[^arr]: Or more. You might want to make a special case for `String` types, or dynamically sized arrays...

## Conclusion

When you're in the trenches of a tutorial like this, it can be hard to see the forest for the trees.
Especially when working with nested structs and arrays, our types got pretty involved.
However, I hope I have also been able to convince you that when this approach works, it can work _really_ well.
The library author (person who defines the descriptor) gets a lot of power, and the user (person who implements the descriptor) only has to define a single generic traversal.
Furthermore, since we aren't using any existing abstractions, we get to completely tailor it to our own needs, as you saw in the array example.

Ultimately, I'm not sure if the ideas here are going to be useful for many people.
I have worked with libraries that horribly over-complicated their FFI that I know there are at least _some_ people who might find this useful, but that's not really the point of this post.
Most importantly, I think it's a neat example of how we can write wonderful abstract interfaces with _just_ `RankNTypes` and some polymorphism in the right places.

If you have any questions or criticism, feel free to contact me.