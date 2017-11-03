+++
title = "Whats Next After Java 9"
date = 2017-11-02T15:48:44+01:00
draft = false
#notonhomepage = true

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = ["Future of Java"]
categories = ["Java"]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
[header]
image = "headers/whats-next-after-java9.jpg"
caption = "[looking into the Palantir](# 'Copyright New Line Cinema')"


# Useful shortcodes:
# {{% toc%}}
# {{% alert note %}} ... {{% /alert %}}
# {{% alert warning %}} ... {{% /alert %}}
# {{< figure src="/img/foo.png" title="Foo!" >}}
# {{< tweet 666616452582129664 >}}
# {{< gist USERNAME GIST-ID  >}}
# {{< speakerdeck 4e8126e72d853c0060001f97 >}}

# Footnotes: [^1] and [^1]: Footnote example.
+++

With Java 9 out the door, you might wonder what the future holds for Java. I do
too, so I had a quick look at current JEPs and Projects candidates to see what
is looming ahead (you'll get this joke in a second :grin:).

In this article I will glance at the ones that stood out to me. Keep in mind
most of these are not finalized, some are even just exploratory...

Without further ceremony, let's dive in!

<!--more-->

{{% toc %}}

# :slot_machine: Value Types
{{% alert note %}}
This is part of `Project Valhalla`, under [JEP-169](http://openjdk.java.net/jeps/169).
The initial design document can be seen [here](http://cr.openjdk.java.net/~jrose/values/values-0.html).
{{% /alert %}}

Project Valhalla has been announced some time ago, in July 2014. One of its focus
is to bring _Value Types_ to the JVM: types that are neither primitive nor `Object`,
but something in the middle:

> Codes like a class, works like an int!

Such types, like primitive types, would have no **identity** and would be **immutable**.
This in turns allow efficient memory layout where all the data backing the type
is colocated in memory and doesn't need things like _boxing_. The typical
example used in the JEP is that of a `Point`, which is just an aggregate type
around two `int`s (the `x` and `y` coordinates). A `Point` doesn't need mutability,
and equality is defined by that of its coordinates, not its reference.

With value types, another improvement would occur with arrays of `Point`s: there
would be no dereference of pointers needed with a `Point[]` like there is eg. for
a `String[]`:

> Similarly, an array of Points would be laid out as a packed array of
alternating x and y values, rather than a series of references.

Other than that, Value Types have all the features of regular classes (except
those that rely on the concept of identity, eg. cloning, locking...).

Foreseen usage range from custom numeric types (like _complex_ numbers) to
flatter data structures (`Map` with less levels of indirection anyone?), Tuples
and algebraic data types like `Either` or `Optional`.

# :black_joker: Generics Over Primitive Types
{{% alert note %}}
This is part of `Project Valhalla`, under [JEP-218](http://openjdk.java.net/jeps/218).
{{% /alert %}}

In this proposal, also part of Project Valhalla, generic types are improved by
allowing generic classes to be generified over non-`Object` types.

So far, having a generic class work with a primitive type would involve the
overhead of **boxing** that primitive type (eg. from an `int` to `Integer`).
Alternatively, one would need to introduce artificial specialized types (like
the `IntStream` in Java 8).

> more memory, more more indirection, allocation, and more garbage collection

With the upcoming introduction of value types, this become even more problematic,
as there would be a proliferation of types that are "hostile to generics".

So the JEP aims at overcoming these limitations of generics in Java and making
them compatible with both primitive types and value types :+1:.

# :package: Data Classes
{{% alert note %}}
This is explored in `Project Amber` by Brian Goetz [here](http://cr.openjdk.java.net/~briangoetz/amber/datum.html)
(not a proposal yet).
{{% /alert %}}

In this very nascent proposal (early design document dates from October 2017),
Brian Goetz explores the possibility of having `Data Classes` in Java.

Data classes are readily available in other JVM like Kotlin and Scala (where
they're called `case` classes). The document notes that there are different
flavors of data classes, listed in a light and funny way as personas:

 - `Algebraic Annie` ("a data class is just an algebraic product type", fits well with pattern
matching)
 - `Boilerplate Billy` ("a data class is just an ordinary class with better syntax")
 - `Tuple Tommy` ("a data class is just a nominal tuple")
 - `Values Victor` ("a data class is really just a more transparent value type")

But in essence, the document falls on the definition where a data class is a
class where the state can be described directly in the class header and is a
"plain data carrier": it is the data, the whole data, and nothing but the data.

This implies a strong relationship between the class and its _state vector_:
a data class can be constructed by simply providing its state, and can be
deconstructed down to its state components. Additionally, deconstructing an
instance to its state vector then reconstructing another instance from that is
always going to yield an equivalent instance.

The constraints listed allow to safely remove some boilerplate (although Brian
seems to think of it as a bonus rather than the end goal: I guess he's no
Boilerplate Billy :smile:).

Namely, Data Classes would not require the developer to write constructors,
accessors, `Object` methods (`equals()`, `hashcode()` and `toString()`)...
They would also be a very good fit for _externalization_, eg. marshaling to
JSON.

This doesn't mean that it would not be possible to have custom constructors, on
the contrary! The only limitation would be that such constructors **must** call
the default one (potentially via a `default()` keyword rather than `super()`).
As the typical use case for that sort of constructor would be to do state vector
checks (e.g. to do invariant checks, like preventing the construction of a
_rational_ number with a `0` denominator), the limitations of `super()`
constructor calls would be partially lifted:

```java
// Explicit principal constructor
public Range(int lo, int hi) {
    // validation logic can happen before call to default constructor
    if (lo > hi)
        throw new IllegalArgumentException(...);

    // delegate to default constructor
    default(lo, hi);
}
```

Furthermore, data classes would be like regular classes in many regards (much
like enums): nothing would prevent them from being generic, having methods,
implementing interfaces, etc...

Things like mutability of fields are still up in the air however.

An interesting section of the design document covers the overlap between data
classes and value types, concluding with the following:

> the notion of a "value data class" is perfectly sensible for things like
extended numerics or tuples.

Value types sacrifice the notion of **identity** for a gain in terms of memory
(a "flat and dense layout of objects in memory"). Data classes have different
restrictions (most likely, at least, no read access encapsulation) and associated
gains (including less boilerplate).

Overall, this design document is a very informative and entertaining read, and
it will be interesting to see the final direction the JEP takes.

# :lipstick: Better Enums
{{% alert note %}}
This is discussed in the Candidate [JEP-301](http://openjdk.java.net/jeps/301).
{{% /alert %}}

This one is all about improving enums, most notably by enabling generics on
enums:

```java
public enum Key<X> {

  NAME<String>("foo"),
  AGE<Integer>(0);

  private final X defaultValue;

  Key(X def) {
    this.defaultValue = def;
  }

  public X defaultValue() {
    return defaultValue;
  }
}
```

Additionally, the generic type would have better retention on enums in order for
the compiler to perform sharper type checking for enum constants. That would
have two direct applications:

 - `NAME` and `AGE` above would be of distinct types, preventing e.g. to pass a
 `NAME` to a method that takes a `Key<Integer>` as a parameter.
 - The type specialization would allow to access methods declared at the levels
 of a specific enum constant.

Here is an example of such a case:

```java
public enum Key<X> {

  NAME<String>("foo"),
  TELEPHONE<String>("") {
    boolean validatePhone(String phone) { return phone != null && phone.startsWith("+"); }
  },
  AGE<Integer>(0);

  //...
}

String phone;
if (Key.TELEPHONE.validatePhone(phone)) {
  map.put(Key.TELEPHONE, phone);
}
```

# :fallen_leaf: Lambdas Leftovers
{{% alert note %}}
This is discussed in the Candidate [JEP-302](http://openjdk.java.net/jeps/302).
{{% /alert %}}

There are three main areas where lambdas can still be improved in Java 10 and
beyond, which are the focus of this proposal:

 - Treatment of underscores
 - Shadowing of lambda parameters
 - Better disambiguation for functional expressions

## Treatment of Underscores
The first one has been prepared for in both Java 8 and Java 9: a simple
underscore identifier `_`:

 - was A-OK in Java 7 and below
 - would produce a compiler warning discouraging in in methods in Java 8. It was
 also forbidden by the compiler _for lambda parameters_.
 - became forbidden by the compiler in Java 9.

Thus the goal of this proposal is to build on that preparation to rehabilitate
`_` as a reserved identifier denoting unused parameters (be it in methods, lambdas
or catch clauses). In the future, one will be able to use `_` to tell the compiler
this is a parameter one doesn't intend to use, allowing stronger static checks
in that regard:

```java
//Given Map's computeIfPresent(K key, BiFunction<K, V, V> remapping):
someMap.computeIfPresent("foo", (_, currentValue) -> currentValue + 1);
```

## Shadowing of Lambda Parameters
In Java 8 and 9, lambda parameters cannot have the same name as a variable in
the lambda's enclosing scope:

```java
Map<String, Integer> msi;//...
//...
String key = computeSomeKey();
msi.computeIfAbsent(key, key -> key.length()); //error
```

The error occurs because the lambda declares a parameter `key` that has the
same name as a variable in the scope.

This proposal offers to allow such shadowing, as it has become a recurring pain
point of lambdas.

## Better Disambiguation of Functional Expressions
This one is considered optional in the proposal, but sounds appealing to me.
The typical problem it proposes to solve is the case where a class has two or
more method overloads that take functional interfaces as parameter, and the
compiler sees an ambiguity when calling these methods with a lambda expression:

```java
public class Value {
  Value operate(Predicate<String> filter);
  Value operate(Function<String, String> transform);
}

Value v = new Value();
v.operate(v -> false);
```

Although it seems rather obvious that the `Function<String, String>` variant
doesn't apply here, the compiler doesn't see it that way. This is because it
considers an implicit lambda as _not pertinent to applicability_, and then
performs a relaxed check where only the arity of parameter is compared to that
of the target function type.

In order to eliminate the accidental ambiguity there, the compiler could still
look at the lambda's return type and detect that it doesn't match the candidate
function type, eliminating the `Function<String, String>` variant from its list
of candidates.

# :zap: Project Loom, Lightweight Threads on the JVM
> Aka Fibers, Coroutines

{{% alert note %}}
[`Project Loom`](http://cr.openjdk.java.net/~rpressler/loom/Loom-Proposal.html)
is currently in the Call For Votes phase, until November 10th 2017. With only
"yes" votes so far, things are looking good :)
{{% /alert %}}

Since [I work on a library](http://projectreactor.io) that aims, among other
things, at making concurrent programming easier in Java, this one is of the most
interest to me :sunglasses: :sparkles:

Project `Loom` aims at adding a new concurrency tool to the Java toolbelt:
**fibers**. Fibers are defined as lightweight `Thread`s, eliminating classic
threads limitations such as their overhead, switching cost and instance count
threshold (it would be perfectly fine to spawn millions of fibers on a single JVM).

Project Loom's end goal is described as:

> to eliminate the tradeoff between simplicity and efficiency in writing
concurrent programs.

A light thread can run code written in imperative style, issuing blocking calls.
It is made up of a **scheduler** and of what is called **continuation**.
Project Loom assesses that Java already has a plenty capable scheduler in the
form of the `ForkJoinPool`, so it will focus on bringing _continuations_ on the
JVM. Since continuations have other applications, it will probably be a
user-facing concept as well, embedded in the Java language itself.

But what's a continuation? A _thread_ (in the general sense) is a sequence of
instructions that run sequentially on a CPU core. It has the ability to suspend
itself and automatically resume at a later point in time (eg. while waiting on
IO). The pausable sequence of instructions is what we called the _continuation_.
The proposed flavor is called a `delimited continuation` or `coroutine`. In a
way, this is akin to Go and Kotlin's _coroutines_.

The other piece of the puzzle is the scheduler: it is its responsibility to
assign the continuation to a particular CPU core and ensure a ready-to-resume
continuation will eventually be assigned a core.

Current `Thread` in Java are kernel-mode threads: the scheduler is the OS, so
it runs in kernel space and has no particular knowledge of the work that is being
done by the continuation (in user space). Furthermore, user/kernel mode switches
have a cost.

Project Loom's lightweight threads, or _fibers_, are user-mode threads: the
scheduling will also happens within the JVM. This allows the scheduler to make
more pinpointed assumptions about the usage pattern, and optimize scheduling
accordingly.

Fibers would be very similar in terms of API to `Thread` (so much so that it
might be possible to implement them by just modifying the `Thread` class), but
would additionally

 1. have a pluggable scheduler (allowing switching schedulers
when a fiber is paused).
 2. be serializable.

Once again, the design document is a very informative and accessible read. It
exposes the concepts and challenges around bringing fibers to the JVM, as well
as some additional applications of continuations. Definitely a recommended read!

# Conclusion
As you can see, plenty of interesting stuff is on the table for future versions
of Java :tada:

With the switch to a release cycle with higher frequency, some of
these features might not be that far ahead of us, although they haven't been
associated to an exact roadmap yet. Time will tell :calendar: :crystal_ball: :eyes:


