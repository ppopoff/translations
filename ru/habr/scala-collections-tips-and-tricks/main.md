Scala Collections Tips and Tricks
=================================

Library article presents a list of simplifications and optimizations
of typical [Scala Collections API](https://www.scala-lang.org/docu/files/collections-api/collections.html) usages.

Some of the tips rest upon subtle implementation details,
though most of the recipes are just common sense transformations that,
in practice, are often overlooked.

The list is inspired by my efforts to devise practical
[Scala Collections inspections](https://youtrack.jetbrains.com/oauth?state=%2Fissues%2FSCL%3Fq%3Dby%253A%2BPavel.Fatin%2Bcollection%2Border%2Bby%253A%2Bcreated)
for the [IntelliJ Scala plugin](https://confluence.jetbrains.com/display/SCA/Scala+Plugin+for+IntelliJ+IDEA).
We’re now in process of implementing those inspections, so if you use the
plugin in IDEA, you’ll automatically benefit from static code analysis.

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
*Prefer `length` to `size` for arrays*

    // Before
    array.size

    // After
    array.length

While `size` and `length` are basically synonyms, in Scala 2.11
`Array.size` calls are still implemented via implicit conversion,
so that intermediate wrapper objects are created for every method call. Unless
you enable [escape analysis](https://en.wikipedia.org/wiki/Escape_analysis)
in JVM , those temporary objects will burden GC and can potentially
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


### 4.5 Existence
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


### 4.6 Filtering
*Don’t negate filter predicate*

    // Before
    seq.filter(!p)

    // After
    seq.filterNot(p)

The latter expression is syntactically simpler
(while semantically they are equal).

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


*Don’t resort to filtering to count elements*

    // Before
    seq.filter(p).length

    // After
    seq.count(p)

The call to `filter` creates an intermediate collection
(which is not really needed) that takes heap space and loads GC.

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


*Don’t use filtering to find first occurrence*

    // Before
    seq.filter(p).headOption

    // After
    seq.find(p)

Unless `seq` is a lazy collection (like `Stream`),
filtering will find all occurrences (and create a temporary collection),
while only the first one is needed.

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


### 4.7 Sorting

*Don’t sort by a property manually*

    // Before
    seq.sortWith(_.property <  _.property)

    // After
    seq.sortBy(_.property)

We have a special method for that, which is more clear and concise.


*Don’t sort by identity manually*

    // Before
    seq.sortBy(identity)
    seq.sortWith(_ < _)

    // After
    seq.sorted

There’s a special method for that, more clear and concise.


*Perform reverse sorting in one step*

    // Before
    seq.sorted.reverse
    seq.sortBy(_.property).reverse
    seq.sortWith(f(_, _)).reverse

    // After
    seq.sorted(Ordering[T].reverse)
    seq.sortBy(_.property)(Ordering[T].reverse)
    seq.sortWith(!f(_, _))

That’s how we can avoid a temporary collection allocation and
exclude the additional transformation step
(to save both heap space and CPU cycles).


### 4.8 Reduction
*Don’t calculate sum manually*

    // Before
    seq.reduce(_ + _)
    seq.fold(z)(_ + _)

    // After
    seq.sum
    seq.sum + z

The advantage is clarity and conciseness.

Other possible methods are:
`reduceLeft`, `reduceRight`, `foldLeft`, `foldRight`.

The second transformation might be replaced with the first when z is 0.

Also applicable to: `Set`, `Iterator`.


*Don’t calculate product manually*

    // Before
    seq.reduce(_ * _)
    seq.fold(z)(_ * _)

    // After
    seq.product
    seq.product * z

Rationale is the same as in the previous tip.

The second transformation might be replaced with the first when z is 1.

Also applicable to: `Set`, `Iterator`.


*Don’t search for the smallest element manually*

    // Before
    seq.reduce(_ min _)
    seq.fold(z)(_ min _)

    // After
    seq.min
    z min seq.min

Rationale is the same as in the previous tips.

Also applicable to: `Set`, `Iterator`.


*Don’t search for the largest element manually*

    // Before
    seq.reduce(_ max _)
    seq.fold(z)(_ max _)

    // After
    seq.max
    z max seq.max

Rationale is the same as in the previous tips.

Also applicable to: `Set`, `Iterator`.


*Don’t emulate forall*

    // Before
    seq.foldLeft(true)((x, y) => x && p(y))
    !seq.map(p).contains(false)

    // After
    seq.forall(p)

The goal of the simplification is clarity and conciseness.

The predicate `p` must be pure.

Also applicable to: `Set`, `Option` (for the second line), `Iterator`.


*Don’t emulate `exists`*

    // Before
    seq.foldLeft(false)((x, y) => x || p(y))
    seq.map(p).contains(true)

    // After
    seq.exists(p)

Besides clarity and conciseness,
the latter expression might be faster
(because it stops element processing when a single occurrence is found)
and it can work with infinite sequences.

The predicate `p` must be pure.

Also applicable to: `Set`, `Option` (for the second line), `Iterator`.


*Don’t emulate `map`*

    // Before
    seq.foldLeft(Seq.empty)((acc, x) => acc :+ f(x))
    seq.foldRight(Seq.empty)((x, acc) => f(x) +: acc)

    // After
    seq.map(f)

That’s a “classical” implementation of map via folding
in functional programming.
While it’s surely didactic, there’s no need to resort to such a
formulation when we have the concise built-in method
(which is also faster, because it uses a simple `while` loop under the hood).

Also applicable to: `Set`, `Option`, `Iterator`.


*Don’t emulate filter*

    // Before
    seq.foldLeft(Seq.empty)((acc, x) => if (p(x)) acc :+ x else acc)
    seq.foldRight(Seq.empty)((x, acc) => if (p(x)) x +: acc else acc)

    // After
    seq.filter(p)

Rationale is the same as in the previous tip.

Also applicable to: `Set`, `Option`, `Iterator`.


*Don’t emulate reverse*

    // Before
    seq.foldLeft(Seq.empty)((acc, x) => x +: acc)
    seq.foldRight(Seq.empty)((x, acc) => acc :+ x)

    // After
    seq.reverse

Again, the built-in method is cleaner and faster.

Also applicable to: `Set`, `Option`, `Iterator`.


### 4.9 Matching

Here are several dedicated tips that are related to Scala’s
[pattern matching](http://docs.scala-lang.org/tutorials/tour/pattern-matching.html)
and [partial functions](https://www.scala-lang.org/api/current/index.html#scala.PartialFunction).


*Use partial function instead of function with pattern matching*

    // Before
    seq.map {
      _ match {
        case P => ??? // x N
      }
    }

    // After
    seq.map {
      case P => ??? // x N
    }

The updated expression produces the same result, yet looks simpler.

It’s possible to apply this transformation to any functions,
not only to `map` arguments.
This tip is not actually collection-related at all. However, because
[higher-order functions](https://en.wikipedia.org/wiki/Higher-order_function)
are so ubiquitous in Scala collection API, the tip is especially handy.


*Convert `flatMap` with partial function to collect*

    // Before
    seq.flatMap {
      case P => Seq(???) // x N
      case _ => Seq.empty
    }

    // After
    seq.collect {
      case P => ??? // x N
    }

The updated expression produces the same result, but looks much simpler.

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


*Convert `match` to `collect` when the result is a collection*

    // Before
    v match {
      case P => Seq(???) // x N
      case _ => Seq.empty
    }

    // After
    Seq(v) collect {
      case P => ??? // x N
    }

When all the case statements produce collections,
it’s possible to simplify the expression by converting the `match`
statement to a `collect` call.
In that way we can write collection creation only once and we can omit
the explicit default `case` branch.

Personally, I often use this trick with `Option` rather than sequences per se.

Also applicable to: `Set`, `Option`, `Iterator`.


*Don’t emulate collectFirst*

    // Before
    seq.collect{case P => ???}.headOption

    // After
    seq.collectFirst{case P => ???}

We have a special method for such a use case,
which also works faster on non-lazy collections.

The partial function must be pure.

Also applicable to: `Set`, `Map`, `Iterator`.


### 4.10 Rewriting
Merge consecutive `filter` calls

    // Before
    seq.filter(p1).filter(p2)

    // After
    seq.filter(x => p1(x) && p2(x))

That’s how we can avoid creation of an intermediate collection
(after the first filter call), and thus relieve GC’s burden.

We may also apply a generalized approach that relies on views (see below),
so the result will be `seq.view.filter(p1).filter(p2).force`.

The predicates `p1` and `p2` must be pure.

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


*Merge consecutive `map` calls*

    // Before
    seq.map(f).map(g)

    // After
    seq.map(f.andThen(g))

Again, we’re creating the resulting collection directly,
without an intermediate one.

We may also apply a generalized approach that relies on views (see below),
so the result will be `seq.view.map(f).map(g).force`.

The functions `f` and `g` must be pure.

Also applicable to: `Set`, `Option`, `Map`, `Iterator`.


*Do sorting after filtering*

    // Before
    seq.sorted.filter(p)

    // After
    seq.filter(p).sorted

Sorting is computationally expensive procedure.
There’s no need to sort elements that will be potentially
filtered out in the next step.

The same applies to the other possible sorting methods,
like `sortWith` and `sortBy`.

The predicate `p` must be pure.


*Don’t reverse collection explicitly before calling `map`*

    // Before
    seq.reverse.map(f)

    // After
    seq.reverseMap(f)

The first expression creates an intermediate (reversed) collection before
transforming elements, which is quite reasonable sometimes (e.g. for `List`).
Other times it’s possible to perform the required transformation directly,
without creating the intermediate collection, which is more efficient.


*Don’t reverse collection explicitly to acquire reverse iterator*

    // Before
    seq.reverse.iterator

    // After
    seq.reverseIterator

Again, the latter expression is simpler and might be more efficient.


*Don’t convert collection to `Set` to find distinct elements*

    // Before
    seq.toSet.toSeq

    // After
    seq.distinct

There’s no need to create a temporary set (at least explicitly) to
find distinct elements.


*Don’t emulate slice*

    // Before
    seq.drop(x).take(y)

    // After
    seq.slice(x, x + y)

For linear sequences all we gain is clear intent and conciseness.
For indexed sequences, however,
we may expect a potential performance improvements.

Also applicable to: `Set`, `Map`, `Iterator`.


*Don’t emulate `splitAt`*

    // Before
    val seq1 = seq.take(n)
    val seq2 = seq.drop(n)

    // After
    val (seq1, seq2) = seq.splitAt(n)

The optimized expression performs faster on linear sequences
(like `List` or `Stream`), because it computes the results in a single pass.

Also applicable to: `Set`, `Map`.


*Don’t emulate `span`*

    // Before
    val seq1 = seq.takeWhile(p)
    val seq2 = seq.dropWhile(p)

    // After
    val (seq1, seq2) = seq.span(p)

That’s how we can traverse the sequence and check the predicate only once,
instead of twice.

The predicate `p` must be pure.

Also applicable to: `Set`, `Map`, `Iterator`.


*Don’t emulate `partition`*

    // Before
    val seq1 = seq.filter(p)
    val seq2 = seq.filterNot(p)

    // After
    val (seq1, seq2) = seq.partition(p)

Again, the benefit is a single-pass computation.

The predicate `p` must be pure.

Also applicable to: `Set`, `Map`, `Iterator`.


*Don’t emulate `takeRight`*

    // Before
    seq.reverse.take(n).reverse

    // After
    seq.takeRight(n)

The latter expression is more concise and potentially more efficient
(for both indexed- and linear sequences).


*Don’t emulate `flatten`*

    // Before (seq: Seq[Seq[T]])
    seq.reduce(_ ++ _)
    seq.fold(Seq.empty)(_ ++ _)
    seq.flatMap(identity)

    // After
    seq.flatten

There’s no need to flatten collections manually when we have a dedicated
built-in method for doing exactly that.

Also applicable to: `Set`, `Option`, `Iterator`.


*Don’t emulate `flatMap`*

    // Before (f: A => Seq[B])
    seq.map(f).flatten

    // After
    seq.flatMap(f)

Again, there’s no need to emulate built-in method manually.
Besides improved clarity, we can skip the creation of an intermediate collection.

Also applicable to: Set, Option, Iterator.


*Don’t use `map` when result is ignored*

    // Before
    seq.map(???) // the result is ignored

    // After
    seq.foreach(???)

When side effect is all that is needed,
there’s no justification for calling `map`.
Such a call is misleading (and less efficient).

Also applicable to: Set, Option, Map, Iterator.


*Don’t use `unzip` to extract a single component*

    // Before (seq: Seq[(A, B]])
    seq.unzip._1

    // After
    seq.map(_._1)

There’s no need to create additional collection(s)
when only a single component is required.

Other possible method: `unzip3`.

Also applicable to: Set, Option, Map, Iterator.


*Don’t create temporary collections*

This recipe is subdivided into three parts
(depending on transformation final result).

1. Transformation reduces collection to a single value.

    // Before
    seq.map(f).flatMap(g).filter(p).reduce(???)

    // After
    seq.view.map(f).flatMap(g).filter(p).reduce(???)

In place of `reduce` might be any method that reduces collection to
a single value, for example:
`reduceLeft, `reduceRight`, `fold`, `foldLeft`, `foldRight`, `sum`, `product`, `min`, `max`,
`head`, `headOption`, `last`, `lastOption`, `indexOf`, `lastIndexOf`, `find`, `contains`,
`exists`, `count`, `length`, `mkString`, etc.

The exact order of transformations is not relevant — what matters
is that we’re creating one or more temporary, intermediate collections,
that are not needed by themselves, yet they will take heap space and burden GC.
That happens because, by default, all collection transformers
(`map`, `flatMap`, `filter`, `++,` etc.) are “strict” (except for `Stream`)
and construct a new collection with all its elements as a result of
transformation.

That’s where views come to the rescue — we may think of view as some
kind of `Iterator`, that allows re-iteration:

  * Views are “lazy” — elements constructed only when they are needed.
  * Views don’t hold created elements in memory (which even `Stream` does).

To go from a collection to its view, we use the `view` method.

2. Transformation produces a collection of the same class.

It’s possible to use views when the final result of transformation is still a
collection — the `force` method will build a collection of the original class
(while creation of all the intermediate collections is still avoided):

    // Before
    seq.map(f).flatMap(g).filter(p)

    // After
    seq.view.map(f).flatMap(g).filter(p).force

If the only intermediate transformation is filtering, you may consider using
`withFilter` method as an alternative:

    seq.withFilter(p).map(f)

The method is originally [intended](https://www.scala-lang.org/old/node/3698.html#comment-14546)
to be used in for comprehensions. It works much like a view — it creates a
temporary object that restricts the domain of subsequent collection
transformations (so, it reorders possible side effects).
However, there’s no need to explicitly convert collection to / from temporary
representation (by calling `view` and `force`).

While view-based approach is more universal, in this particular case you may
prefer `withFilter` method because of the conciseness.

3. Transformation creates a collection of different class.

    // Before
    seq.map(f).flatMap(g).filter(p).toList

    // After
    seq.view.map(f).flatMap(g).filter(p).toList

This time we use a suitable converter method
(instead of the generic `force` call),
so the result will be a collection of different class.

There also exists an alternative way of handling “transformation + conversion”
case that relies on `breakOut`:

    seq.map(f)(collection.breakOut): List[T]

Such an expression is functionally equivalent to using a view,
however this approach:

  * needs expected type to be explicit
  (which often requires an additional type annotation),
  * is limited to a single transformation
  (like `map`, `flatMap`, `filter`, `fold`, etc.),
  * looks rather tricky
  (because implicit builders are usually [omitted](https://www.scala-lang.org/api/current/index.html#scala.collection.Seq)
  from the Scala Collections API docs).

Thus, it’s usually better to substitute `breakOut` for a view, which is more
clear, more concise and more flexible.

Views are especially efficient when collection size is relatively large.

All the functions (like `f` and `g`) and predicates (`p`) must be pure
(because view might delay, skip or reorder computations).

Also applicable to: `Set`, `Map`.


*Use assignment operators to reassign a sequence*

    // Before
    seq = seq :+ x
    seq = x +: seq
    seq1 = seq1 ++ seq2
    seq1 = seq2 ++ seq1

    // After
    seq :+= x
    seq +:= x
    seq1 ++= seq2
    seq1 ++:= seq2

Scala offers a syntactic sugar known as “assignment operators” —
it automatically converts `x <op>= y` statements into `x = x <op> y`
where `<op>` is some character operator (like `+,` `-,` etc.).
Note, that if `<op>` ends with `:` it is considered right-associative
(i. e. invoked on the right expression, instead of the left).

There’s also a special syntax for lists and streams:

    // Before
    list = x :: list
    list1 = list2 ::: list

    stream = x #:: list
    stream1 = stream2 #::: stream

    // After
    list ::= x
    list1 :::= list2

    stream #::= x
    stream1 #:::= stream2

The optimized statements are more concise.

Also applicable to: `Set`, `Map`, `Iterator` (with operator specifics in mind).


*Don’t convert collection type manually*

    // Before
    seq.foldLeft(Set.empty)(_ + _)
    seq.foldRight(List.empty)(_ :: _)

    // After
    seq.toSet
    seq.toList

We have corresponding build-it methods for doing that, which are
cleaner and faster.

If you need to transform or filter values in the course of the conversion,
consider using views or similar techniques, as described above.

Also applicable to: `Set`, `Option`, `Iterator`.


*Be careful with `toSeq` on non-strict collections*

    // Before (seq: TraversableOnce[T])
    seq.toSeq

    // After
    seq.toStream
    seq.toVector

Because `Seq(...)` creates a strict collection (namely, [Vector](https://www.scala-lang.org/api/current/index.html#scala.collection.immutable.Vector)),
we may be tempted to use toSeq to convert a non-strict entity
(like `Stream`, `Iterator`, or “view”) to a strict collection. However,
`TraversableOnce.toSeq` returns a `Stream` under the hood, which is a lazy
collection, and that might result in hard-to-find bugs or performance problems.
Even if a stream is exactly what you expect,
such an expression might confuse other people who read your code.

Here’s a typical example of the pitfall:

    val source = Source.fromFile("lines.txt")
    val lines = source.getLines.toSeq
    source.close()
    lines.foreach(println)

Such a code will throw an `IOException` complaining that
the stream is already closed.

To clarify the intent, it’s better to write `toStream` explicitly, or,
if we need a strict collection, after all, to use `toVector` instead of `toSeq`.

*Don’t convert to String manually*

    // Before (seq: Seq[String])
    seq.reduce(_ + _)
    seq.reduce(_ + separator + _)
    seq.fold(prefix)(_ + _)
    seq.map(_.toString).reduce(_ + _) // seq: Seq[T]
    seq.foldLeft(new StringBuilder())(_ append _)

    // After
    seq.mkString
    seq.mkString(prefix, separator, "")

The latter approach is cleaner and potentially faster,
because it uses a single `StringBuilder` under the hood.

Other possible methods are:
`reduceLeft`, `reduceRight`, `foldLeft`, `foldRight`.

Also applicable to: `Set`, `Option`, `Iterator`.


## 5. Sets
Most of the tips for sequences are applicable to sets as well.
Additionally, there are several set-specific tips available.
Don’t use sameElements to compare unordered collections

    // Before
    set1.sameElements(set2)

    // After
    set1 == set2

This rule was introduced earlier (for sequences), yet for sets the
rationale is even more sound.

The `sameElements` method might return indeterministic results for
unordered collections, because this method respects order of elements,
while we cannot rely on the order in a set.

Classes that explicitly guarantee predictable iteration order
(like `LinkedHashSet`) are exceptions from this rule.

Also applicable to: `Map`.


*Don’t compute set intersection manually*

    // Before
    set1.filter(set2.contains)
    set1.filter(set2)

    // After
    set1.intersect(set2) // or set1 & set2

The latter expression is more clear and concise
(while performance is the same).

This transformation can be performed on sequences,
however we should keep in mind, that in such a case,
duplicate elements will be handled differently.


*Don’t compute set difference manually*

    // Before
    set1.filterNot(set2.contains)
    set1.filterNot(set2)

    // After
    set1.diff(set2) // or set1 &~ set2

Again, the updated expression is more clear and concise
(while performance is the same).

The transformation is potentially applicable to sequences,
though we should consider duplicate elements.


## 6. Options
Technically, `Option` is not a part of the Scala Collections,
however it provides similar interface (with monadic methods, etc.)
and behaves like a special kind of collection that may or may not
hold some value.

Many of the tips for sequences are also applicable to options. Besides,
here are dedicated tips, that are based on `Option` API.

### 6.1 Value
*Don’t compare option values with None*

    // Before
    option == None
    option != None

    // After
    option.isEmpty
    option.isDefined

While such a comparison is fully legitimate, we have a more straightforward
way to check whether option is defined.

Another advantage of this simplification, is that if you ever decide to
change the type from `Option[T]` to just `T`, scalac will compile the former
expression (yielding only a warning), while compilation of the latter one
will result in a righteous error.


*Don’t compare option values with Some*

    // Before
    option == Some(v)
    option != Some(v)

    // After
    option.contains(v)
    !option.contains(v)

This tip is a complement to the previous one.


*Don’t rely on instance type to check value existence*

    // Before
    option.isInstanceOf[Some[_]]

    // After
    option.isDefined

There’s simply no need for such trickery.


*Don’t resort to pattern matching to check value existence*

    // Before
    option match {
        case Some(_) => true
        case None => false
    }

    option match {
        case Some(_) => false
        case None => true
    }

    // After
    option.isDefined
    option.isEmpty

Again, while the former expression is technically correct, there’s no
justification for such an extravagance. Besides,
the simplified expression will work faster.

Also applicable to: `Seq`, `Set`.


*Don’t negate value existence-related properties*

    // Before
    !option.isEmpty
    !option.isDefined
    !option.nonEmpty

    // After
    seq.isDefined
    seq.isEmpty
    seq.isEmpty

Rationale is the same as the one for sequences — simple property adds less
visual clutter than a compound expression.

Note that we have the synonyms: `isDefined` (option-specific) and `nonEmpty`
(sequence-specific). It might be reasonable to prefer the former one,
to distinguish options from sequences.


### 6.2 Null
*Don’t compare value with `null` explicitly to construct an `Option`*

    // Before
    if (v != null) Some(v) else None

    // After
    Option(v)

We have a special, concise syntax for that.


*Don’t provide `null` as an explicit alternative*

    // Before
    option.getOrElse(null)

    // After
    option.orNull

We can rely on the predefine method in such a case,
so the expression will become shorter.


### 6.3 Processing

It’s possible to single out a groups of tips that are related to how `Option`
value is processed.

As `Option`‘s API [documentation](https://www.scala-lang.org/api/current/index.html#scala.Option)
says that “the most idiomatic way to use an
`Option` instance is to treat it as a collection or monad and use
map, flatMap, filter, or foreach”, the basic principle here is to
avoid “check & get” chains, implemented either via
if statement or via pattern matching.

The goal is conciseness and robustness, the “monadic” code is:

  * more concise and intelligible,
  * safeguarded against `NoSuchElementException` nor `MatchError`
  exceptions at runtime.

This rationale is common for all the following cases.


*Don’t emulate `getOrElse`*

    // Before
    if (option.isDefined) option.get else z

    option match {
      case Some(it) => it
      case None => z
    }

    // After
    option.getOrElse(z)


*Don’t emulate `orElse`*

    if (option1.isDefined) option1 else option2

    option1 match {
      case Some(it) => Some(it)
      case None => option2
    }

    // After
    option1.orElse(option2)


*Don’t emulate `exists`*

    // Before
    option.isDefined && p(option.get)

    if (option.isDefined) p(option.get) else false

    option match {
      case Some(it) => p(it)
      case None => false
    }

    // After
    option.exists(p)


*Don’t emulate `forall`*

    // Before
    option.isEmpty || (option.isDefined && p(option.get))

    if (option.isDefined) p(option.get) else true

    option match {
      case Some(it) => p(it)
      case None => true
    }

    // After
    option.forall(p)


*Don’t emulate `contains`*

    // Before
    option.isDefined && option.get == x

    if (option.isDefined) option.get == x else false

    option match {
      case Some(it) => it == x
      case None => false
    }

    // After
    option.contains(x)


*Don’t emulate `foreach`*

    // Before
    if (option.isDefined) f(option.get)

    option match {
      case Some(it) => f(it)
      case None =>
    }

    // After
    option.foreach(f)


*Don’t emulate `filter`*

    // Before
    if (option.isDefined && p(option.get)) option else None

    option match {
      case Some(it) && p(it) => Some(it)
      case _ => None
    }

    // After
    option.filter(p)


*Don’t emulate `map`*

    // Before
    if (option.isDefined) Some(f(option.get)) else None

    option match {
      case Some(it) => Some(f(it))
      case None => None
    }

    // After
    option.map(f)


*Don’t emulate `flatMap`*

    // Before (f: A => Option[B])
    if (option.isDefined) f(option.get) else None

    option match {
      case Some(it) => f(it)
      case None => None
    }

    // After
    option.flatMap(f)


### 6.4 Rewriting
*Convert `map` with `getOrElse` to fold*

    // Before
    option.map(f).getOrElse(z)

    // After
    option.fold(z)(f)

Those expressions are semantically equivalent (in both cases `z` is computed
lazily, on demand), yet the latter expression is more concise.
The transformation may sometimes require an additional type annotation
(because of the way how Scala’s type inference works), and, in such cases,
the former expression might be preferable.

Keep in mind that this particular simplification is somehow
[controversial](https://www.reddit.com/r/scala/comments/2z411u/scala_collections_tips_and_tricks/cqiip08/),
because the latter expression looks more obscure, especially if you
not accustomed to it.


*Don’t emulate `exists`*

    // Before
    option.map(p).getOrElse(false)

    // After
    option.exists(p)

We presented a rather similar rule for sequences
(which is also directly applicable to options).
This transformation is a special case with `getOrElse` call.


*Don’t emulate `flatten`*

    // Before (option: Option[Option[T]])
    option.map(_.get)
    option.getOrElse(None)

    // After
    option.flatten

The latter expression looks cleaner.


*Don’t convert option to sequence manually*

    // Before
    option.map(Seq(_)).getOrElse(Seq.empty)
    option.getOrElse(Seq.empty) // option: Option[Seq[T]]

    // After
    option.toSeq

There’s a special method for doing that, which is more concise and,
less computationally-expensive.


## 7. Maps

Just like with other collection classes, many tips for sequences are
applicable to maps as well, so we’ll enumerate only map-specific ones.


*Don’t search for a value manually*

    // Before
    map.find(_._1 == k).map(_._2)

    // After
    map.get(k)

While in principe the former code will work, the performance is sub-optimal,
because `Map` is not a simple collection of (key, value) pairs — it can
perform lookups in a much more efficient way.

Moreover, the latter expression is simple and easier to undestand.


*Don’t use `get` when a raw value is needed*

    // Before
    map.get(k).get

    // After
    map(k)

There’s no need to produce an intermediate `Option` value when a
raw value is needed.


*Don’t use lift instead of get*

    // Before
    map.lift(k)

    // After
    map.get(k)

There’s no need to treat map value as a partial function to acquire an
optional result (which is useful for sequences), because we have a
built-in method with the same functionality. Though `lift` works fine, it
performs additional transformation (from `Map` to `PartialFunction`) and it
might look confusing.


*Don’t call `get` with `getOrElse` separately*

    // Before
    map.get(k).getOrElse(z)

    // After
    map.getOrElse(k, z)

The single method call is simpler, both syntax- and performance-wise.
In both cases `z` is computed lazily, on demand.


*Don’t extract keys manually*

    // Before
    map.map(_._1)
    map.map(_._1).toSet
    map.map(_._1).toIterator

    // After
    map.keys
    map.keySet
    map.keysIterator

The optimized expressions are more intelligible (and might be faster).


*Don’t extract values manually*

    // Before
    map.map(_._2)
    map.map(_._2).toIterator

    // After
    map.values
    map.valuesIterator

The simplified expressions are more comprehensible (and potentially faster).


*Be careful with `filterKeys`*

    // Before
    map.filterKeys(p)

    // After
    map.filter(p(_._1))

The `filterKeys` methods wraps the original map without copying any elements.
There’s nothing wrong with that per se, however such a behaviour is hardly
expected from `filterKeys` method.
Because it unexpectedly behaves like a view, code performance might be
substantially degraded in some cases (e. g. for `filterKeys(p).groupBy(???)`).

Another possible pitfall is unexpected “laziness” (collection transformers
are expected to be strict by default) – the predicated is not evaluated at
all withing the method call itself,
so possible side effects might be reordered.

The `filterKeys` method should probably be deprecated, because it’s now
impossible to [make it strict](https://issues.scala-lang.org/browse/SI-4776)
without breaking backward compatibility. A more suitable name for the current
implementation is `withKeyFilter` (by analogy with the withFilter method).

All in all, it seems reasonable to follow the
[Rule of Least Surprise](https://en.wikipedia.org/wiki/Principle_of_least_astonishment)
and to filter keys manually.

Nevertheless, as the view-like functionality of `filterKeys` is potentially
useful (when only a few entries will be accessed while the map is relatively
large), we may still consider using the method. To keep other people who read
(or modify) our code from confusion, it’s better clarify our intent by defining
an appropriate method synonym:

    type MapView[A, +B] = Map[A, B]

    implicit class MapExt[A, +B](val map: Map[A, B]) extends AnyVal {
      def withKeyFilter(p: A => Boolean): MapView[A, B] =
        map.filterKeys(p)
    }

We’re using the type alias `MapView` to indicate that the result
map is view-like.

Another option is to define a simple helper method, like:

    def get(k: T) = if (p(k)) map.get(k) else None


*Be careful with mapValues*

    // Before
    map.mapValues(f)

    // After
    map.map(f(_._2))

Rationale is the same as in the previous tip. In a similar way,
we may define an unambiguous synonym:

    type MapView[A, +B] = Map[A, B]

    implicit class MapExt[A, +B](val map: Map[A, B]) extends AnyVal {
      def withValueMapper[C](f: B => C): MapView[A, C] =
        map.mapValues(f)
    }

A simpler option would be to define a helper method, like:

    def get(k: T) = map.get(k).map(f)


*Don’t filter out keys manually*

    // Before
    map.filterKeys(!seq.contains(_))

    // After
    map -- seq

We can rely on the simpler syntax to filter out keys.


*Use assignment operators to reassign a map*

    // Before
    map = map + x -> y
    map1 = map1 ++ map2
    map = map - x
    map = map -- seq

    // After
    map += x -> y
    map1 ++= map2
    map -= x
    map --= seq

As with sequences, we may rely on syntactic sugar to simplify such statements.


## 8. Supplement
In addition to the listed recipes, I recommend you to take a look at the
[Scala Collections documentation](http://docs.scala-lang.org/overviews/collections/introduction.html),
which is surprisingly good an easy to read.

See also:

  * [Scala for Project Euler](https://pavelfatin.com/scala-for-project-euler/) —
    Concise functional Scala solutions to the Project Euler problems.
  * [Ninety-nine](https://pavelfatin.com/ninety-nine/) — Ninety-nine problems
    in Scala, Java, Clojure and Haskell (with multiple solutions)

Those puzzles really helped me to shape and deepen my understanding of
Scala collections.

While the list of tips is rather comprehensive,
it’s probably far from complete. Moreover, because applicability
of the recipes varies greatly, some fine-tuning is definitely needed.
Your suggestions welcome.

