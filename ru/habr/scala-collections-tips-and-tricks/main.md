Scala Collections Tips and Tricks
=================================

Library article presents a list of simplifications and optimizations
of typical Scala Collections API usages.

Some of the tips rest upon subtle implementation details,
though most of the recipes are just common sense transformations that,
in practice, are often overlooked.

The list is inspired by my efforts to devise practical Scala Collections
inspections for the IntelliJ Scala plugin. We’re now in process of
implementing those inspections, so if you use the plugin in IDEA,
you’ll automatically benefit from static code analysis.

Nevertheless, the recipes are valuable by themselves and can help
you to deepen your understanding of Scala Collections
and to make your code faster and cleaner.


**Contents:**
  1. Legend
  2. Composition
  3. Side effects
  4. Sequences
    4.1. Creation
    4.2. Length
    4.3. Equality
    4.4. Indexing
    4.5. Existence
    4.6. Filtering
    4.7. Sorting
    4.8. Reduction
    4.9. Matching
    4.10. Rewriting
  5. Sets
  6. Options
    6.1. Value
    6.2. Null
    6.3. Processing
    6.4. Rewriting
  7. Maps
  8. Supplement

Все примеры кода доступны в
[репозитории на GitHub](https://github.com/pavelfatin/scala-collections-tips-and-tricks).


## 1. Legend
To make code examples more comprehensible, I
adhered to the following notation conventions:

  * `seq` — an instance of `Seq`-based collection, like `Seq(1, 2, 3)`
  * `set` — an instance of `Set`, for example `Set(1, 2, 3)`
  * `array` — an array, e. g. `Array(1, 2, 3)`
  * `option` — an instance of `Option`, for example, `Some(1)`
  * `map` — an instance of `Map`, like `Map(1 -> "foo", 2 -> "bar")`
  * `???` — an arbitrary expression
  * `p` — a predicate function with type `T => Boolean`, like `_ > 2`
  * `n` — an integer value
  * `i` — an integer index
  * `f`, `g` — simple functions, `A => B`
  * `x`, `y` — some arbitrary values
  * `z` — initial or default value
  * `P` — a pattern

## 2. Composition
Keep in mind that although all the recipes are isolated and
self-contained, we can compose them to iteratively transform
more advanced expressions, for example:

    seq.filter(_ == x).headOption != None

    // Via seq.filter(p).headOption -> seq.find(p)

    seq.find(_ == x) != None

    // Via option != None -> option.isDefined

    seq.find(_ == x).isDefined

    // Via seq.find(p).isDefined -> seq.exists(p)

    seq.exists(_ == x)

    // Via seq.exists(_ == x) -> seq.contains(x)

    seq.contains(x)

So, we can rely on “substitution model of recipe application” (by analogy
with [SICP](https://mitpress.mit.edu/sicp/full-text/sicp/book/node10.html))
to simplify complex expressions.

## 3. Side Effects
“Side effect” is an essential concept that should be reviewed
before enumerating the actual transformations.

Basically, side effect is any action besides returning a value,
that is observable outside of a function or expression scope, like:

  * input/output operation,
  * variable modification (that is accessible outside of the scope),
  * object state modification (that is observable outside of the scope),
  * throwing an exception (that is not caught within the scope).

When a function or expression contain any of these actions,
they are said to have side effects, otherwise they are said to be “pure”.

Why side effects are such a big deal?
Because in the presence of side effects, the order of evaluation matters.
For example, here are two “pure” expressions
(assigned to the corresponding values):

    val x = 1 + 2
    val y = 2 + 3

Because they contain no side effects
(i. e. effects that are observable outside of the expressions),
we may actually evaluate those expressions in any order — `x` then `y`,
or `y` then `x` — it doesn’t disrupt evaluation correctness
(we may even cache the result values, if we want to).
Now let’s consider the following modification:

    val x = { print("foo"); 1 + 2 }
    val y = { print("bar"); 2 + 3 }

That’s another story — we cannot reverse the order of evaluation,
because, that way, “barfoo” instead of “foobar” will be printed to
console (and that is not what we expect).

So, the presence of side effects **reduces the number of
possible transformations** (including simplifications and optimizations)
that we may apply to code.

The same reasoning can be applied to collection-related expressions.
Let’s imagine that we have some out-of-scope `builder`
(with side-effecting method `append`):

    seq.filter{ x => builder.append(x); x > 3 }.headOption

In principle, `seq.filter(p).headOption` construct is
reducible to `seq.find(p)` call,
however the side effect presence forbids us to do that.
Here’s an attempt:

    seq find { x => builder.append(x); x > 3 }

While those expressions are equivalent in terms of result value,
they are not equivalent in regard to side effects.
The former expression will append all the elements, while the latter
one will omit elements after a first predicate match.
So, such a simplification cannot be performed.

What can we do to make the automatic simplification possible?
The answer is a **golden rule** that should be applied to all side effects
in our code (including code with no collections at all):

  * avoid side effects, whenever possible,
  * isolate side effect from pure code, otherwise.

Thus, we need either to get rid of the `builder`
(with its side-effecting API),
or separate the call to `builder` from the pure expression.
Let’s assume that this `builder` is some third-party object
that we cannot get rid of, so we have to isolate the call:

    seq.foreach(builder.append)
    seq.filter(_ > 3).headOption

Now we can safely apply the transformation:

    seq.foreach(builder.append)
    seq.find(x > 3)

Nice and clean! The isolation of the side effect made automatic
transformation possible. An additional benefit is that,
because of the clean separation, the resulted code is
easier to comprehend by humans.

Less obvious, yet very important benefit of side effect isolation, is that
our code becomes more robust, irrelatively of the possible simplifications.
As applied to the example,
here’s why: the original expression might produce different side effects
depending on the actual `Seq` implementation — with `Vector`,
for example, it will append all the elements, yet with `Stream` it
will omit elements after a first predicate match
(because streams are “lazy” — elements are only evaluated when they are needed).
The side effect isolation allowed us to avoid this indeterminate behavior.


## 4. Sequences
While tips in the current section are intended for classes
that are descendants of `Seq`, some transformations are applicable
to other collection (and non-collection) classes,
like `Set`, `Option`, `Map` and even `Iterator`
(because all of them provide similar interfaces with monadic methods).

### 4.1 Creation
*Create empty collections explicitly*

    // Before
    Seq[T]()

    // After
    Seq.empty[T]

Some immutable collection classes provide singleton “empty” implementations,
however not all of the factory methods check length of the created collections.
Thus, by making collection emptiness apparent at compile time,
we could save either heap space (by reusing empty collection instances)
or CPU cycles (otherwise wasted on runtime length checks).

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.

### 4.2 Length
*Prefer `length` to size for arrays*
    // Before
    array.size

    // After
    array.length

While `size` and `length` are basically synonyms, in Scala 2.11
`Array.size` calls are still implemented via implicit conversion,
so that intermediate wrapper objects are created for every method call.
Unless you enable escape analysis in JVM ,
those temporary objects will burden GC and can potentially
degrade code performance (especially, within loops).


*Don’t negate emptiness-related properties*
    // Before
    !seq.isEmpty
    !seq.nonEmpty

    // After
    seq.nonEmpty
    seq.isEmpty

Simple property adds less visual clutter than a compound expression.
Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


*Don’t compute length for emptiness check*
    // Before
    seq.length > 0
    seq.length != 0
    seq.length == 0

    // After
    seq.nonEmpty
    seq.nonEmpty
    seq.isEmpty

On the one hand, simple property is more easy to perceive than
compound expression. On the other hand — collections that are decedents
of `LinearSeq` (like `List`) can take `O(n)` time to compute length
(instead of `O(1)` time for `IndexedSeq`),
so we can speedup our code by avoiding length computations when
exact value is not really required.

Additionally, keep in mind that `.length` call on an infinite stream will
never complete, yet we can always verify stream emptiness directly.

Also applicable to: `Set`, `Map`.


*Don’t compute full length for length matching*

    // Before
    seq.length > n
    seq.length < n
    seq.length == n
    seq.length != n

    // After
    seq.lengthCompare(n) > 0
    seq.lengthCompare(n) < 0
    seq.lengthCompare(n) == 0
    seq.lengthCompare(n) != 0

Because length calculation might be “expensive” computation for
some collection classes, we can reduce comparison time from `O(length)`
to `O(length min n)` for decedents of `LinearSeq`
(which might be hidden behind `Seq`-typed values).

Besides, such approach is indispensable when
we’re dealing with infinite streams.


### 4.3 Equality
*Don’t rely on == to compare array contents*

    // Before
    array1 == array2

    // After
    array1.sameElements(array2)

Equality check always produces `false` for different array instances.
Also applicable to: `Iterator`.


*Don’t check equality between collections in different categories*

    // Before
    seq == set

    // After
    seq.toSet == set

Equality checks cannot be used to compare collections in
different categories (i. e. `List` vs. `Set`).

Please note that you should think twice about exact meaning of such a check
(in reference to the example — how to treat duplicates in the sequence).


*Don’t use `sameElements` to compare ordinary collections*

    // Before
    seq1.sameElements(seq2)

    // After
    seq1 == seq2

Equality check is the way to go when we’re comparing collections
in the same category. In theory, that might improve performance
because of possible underlying instance check
(`eq`, which is usually much faster).


*Don’t check equality correspondence manually*

    // Before
    seq1.corresponds(seq2)(_ == _)

    // After
    seq1 == seq2

We have a built-in method that does the same.
Both expressions respect order of elements.
Again, we might benefit from performance improvement.

## 4.4 Indexing
*Don’t retrieve first element by index*

    // Before
    seq(0)

    // After
    seq.head

The updated approach may be slightly faster for some collection classes
(check `List.apply` code, for example).
Additionally, property access is simpler (both syntactically and semantically)
than method call with an argument.


*Don’t retrieve last element by index*

    // Before
    seq(seq.length - 1)

    // After
    seq.last

The latter expression is more obvious and allows to avoid redundant
collection length computation (that might be slow for linear sequences).
Besides, some collection classes can retrieve last element more efficiently
comparing to by-index access.


*Don’t check index bounds explicitly*

    // Before
    if (i < seq.length) Some(seq(i)) else None

    // After
    seq.lift(i)

The second expression is semantically equivalent, yet more concise.


*Don’t emulate headOption*

    // Before
    if (seq.nonEmpty) Some(seq.head) else None
    seq.lift(0)

    // After
    seq.headOption

The optimized expression is more concise.


*Don’t emulate `lastOption`*

    // Before
    if (seq.nonEmpty) Some(seq.last) else None
    seq.lift(seq.length - 1)

    // After
    seq.lastOption

The optimized expression is more concise (and potentially faster).


*Be careful with `indexOf` and `lastIndexOf` argument types*

    // Before
    Seq(1, 2, 3).indexOf("1") // compilable
    Seq(1, 2, 3).lastIndexOf("2") // compilable

    //  After
    Seq(1, 2, 3).indexOf(1)
    Seq(1, 2, 3).lastIndexOf(2)

Because of [how variance works](http://stackoverflow.com/questions/2078246/why-does-seq-contains-accept-type-any-rather-than-the-type-parameter-a/2078619#2078619),
`indexOf` and `lastIndexOf` methods accept arguments of `Any` type.
In practice, that might lead to hard-to-find bugs, which are not
discoverable at compile time. That’s where auxiliary IDE
inspections come in handy.


*Don’t construct indices range manually*

    // Before
    Range(0, seq.length)

    // After
    seq.indices

There’s a built-in method that returns the range of all indices of a sequence.


*Don’t zip collection with its indices manually*

    // Before
    seq.zip(seq.indices)

    // After
    seq.zipWithIndex

For one thing, the latter expression is more concise. Besides, we may
expect some performance gain, because we avoid hidden
length calculation (which might be expensive for linear sequences).

Additional benefit of the latter expression is that it works well with
possibly infinite collections (like `Stream`).


## 4.5 Existence
*Don’t use equality predicate to check element presence*

    // Before
    seq.exists(_ == x)

    //  After
    seq.contains(x)

The second expression is semantically equivalent, yet more concise.

When those expressions are applied to `Set` classes,
performance might be different as night and day,
because sets offer close to `O(1)` lookups
(due to internal indexing, which is left unused within `exists` calls).

Also applicable to: `Set`, `Option`, `Iterator`.

*Be careful with `contains` argument type*

    // Before
    Seq(1, 2, 3).contains("1") // compilable

    //  After
    Seq(1, 2, 3).contains(1)

Just like `indexOf` and `lastIndexOf` methods,
contains accepts arguments of `Any` type, what might lead
to hard-to-find bugs, which are not discoverable at compile time.
Be careful with the method arguments


*Don’t use inequality predicate to check element absence*

    // Before
    seq.forall(_ != x)

    // After
    !seq.contains(x)

Again, the latter expression is cleaner and
possibly faster (especially, for sets).
Also applicable to: `Set`, `Option`, `Iterator`.


*Don’t count occurrences to check existence*

    // Before
    seq.count(p) > 0
    seq.count(p) != 0
    seq.count(p) == 0

    //  After
    seq.exists(p)
    seq.exists(p)
    !seq.exists(p)

Obviously, when we need to know whether a predicate holds for some
elements of the collection, counting the number of elements which
satisfy the predicate is redundant.
The optimized expressions looks cleaner and performs better.
The predicate `p` must be pure.
Also applicable to: `Set`, `Map`, `Iterator`.


*Don’t resort to filtering to check existence*

    // Before
    seq.filter(p).nonEmpty
    seq.filter(p).isEmpty

    // After
    seq.exists(p)
    !seq.exists(p)

The call to `filter` creates an intermediate collection which takes heap
space and loads GC. Besides, the former expressions find all occurrences,
when only the first one is needed
(which might slowdown code, depending on likely collection contents).

The potential performance gain is less significant for lazy collections
(like `Stream` and, especially, Iterator).

The predicate `p` must be pure.

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.




