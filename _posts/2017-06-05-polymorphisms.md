---
title:  Polymorphism and higher kinds
description: TBD
date:   2017-06-05 18:05:00
category: scala
tags: scala learning
comments: false
published: true
---

We'll talk briefly about parametric, subtype and ad-hoc polymorphism, then focus on _ad-hoc polymorphism_ and its relationship with the _type class pattern_ and higher-kinded types like **Functor**, then move on to discuss higher-kinded types shipped with the _Cats_ library.



The following polymorphism discussion is based in part on this [Herding Cats](http://eed3si9n.com/herding-cats/polymorphism.html) article.

## Polymorphism
### 1. Parametric polymorphism

Using parametric polymorphism, a function or a data type can be written generically so that it can handle values identically without depending on their type.[[1]][1] When the type of a value contains one or more unconstrained type variables, the value may adopt any type that results from substituting those variables with concrete types.

{% highlight scala %}
  def fun[A](a:A): Unit = println(a)
{% endhighlight %}

### 2. Subtype polymorphism

We can constrain the type parameter by an **upper bound**: 

{% highlight scala %}
trait PlusInt[A] {
  def plus(i2: A): A
}
def plusBySubtype[A <: PlusIntf[A]](a1: A, a2: A): A =
  a1.plus(a2)
{% endhighlight %}

This requires the types used in `plusBySubtype` to _extend_ the trait. This means, it could not _be_ an `Int`.

### 3. Ad-hoc polymorphism

We can constrain the type parameter by a **context bound**: 

{% highlight scala %}
trait CanPlus[A] {
  def plus(a1: A, a2: A): A
}
def plus[A: CanPlus](a1: A, a2: A): A =
  implicitly[CanPlus[A]].plus(a1, a2)
{% endhighlight %}

This places a special kind of constraint on the type parameter. The actual type of `A` must be from a specific class of types (type class).

A context bound like the above will be de-sugared to:

{% highlight scala %}
def plus[A](a1: A, a2: A)(implicit cpa: CanPlus[A]): A =
  cpa.plus(a1, a2)
{% endhighlight %}

This means that in order to call the `plus` method, the compiler will need to be able to find evidence that behaviour/properties described by `CanPlus` have been implemented for the actual type of `A` (or an explicit instance must be provided by the programmer).

## Type classes

What determines membership of a type `T` in a type class `TC` is the ability to provide an implementation of the trait `TC` for the specific type `T` (i.e. an instance of `TC[T]`). For example, the `String` type could be a member of the `Party` type class, if the compiler could find an implicit instance of `Party[String]` in scope. This would serve as _evidence_ that the type `String` indeed belongs to the class of types that __know how to `Party`__.

([Read Eugene's article](http://eed3si9n.com/revisiting-implicits-without-import-tax) to understand how implicits are resolved.)

## Values, Types and Kinds

**Kinds are to types what types are to values.** Just like different values can be of the same type, Different types can be of the same kind. For example `String`, `Int`, `Boolean` and `Banana` are of the _same kind_ - namely, they are all **proper** types. The **kind** of a _proper type_ is usually denoted as `*`.

-----

Some types are parameterised with other _proper_ types: e.g. `List`, `Option`, `Try` and `Future`. Also known as _type constructors_, they require type argument(s) in order to construct a _proper_ type. These would be classified as **first-order types**. There are various **kinds** of first-order types: 
 - `* -> *` for a type constructor like `List` or `Option` or `Function0`
 - `* -> * -> *` for a type constructor that takes two type arguments like `Either` or `Map` or `Function1`
 - `* -> * -> * -> * -> * -> *` for something as highly parameterised as `Function4`

-----

**Higher-order types** declare a first-order (or higher order) type parameter(s). Higher-order types declare type constructors as their type parameters. They are of a **higher kind**. Many higher-kinded types are provided by the [cats library][2] (or other libraries like [ScalaZ][3]):
 - `(* -> *) -> *` as is `Functor[F[_]]` or `Monad[F[_]]`
 - `(* -> *) -> * -> *` as is for example the `OptionT[F[_], A]` monad transformer
 - `(* -> *) -> * -> * -> *` like the `EitherT[F[_], A, B]` monad transformer
 - `(* -> *) -> (* -> *) -> * -> *` as in `EitherK[F[_], G[_], A]`

## Cats

There's a good discussion of [Type classes][4] on the Typelevel _Cats_ site, from which we'll "lift" this line:

> Functional programming tends towards a combination of parametric polymorphism [...] and ad-hoc polymorphism.

The cats library revolves around a rich collection of type classes that encode a variety of **higher-level abstractions**. `List` is an abstraction, `Option` is another, and _both_ have something in common: both could be viewed as mappable containers - functors. Thus `Functor` is a higher-level abstraction - an abstraction over abstractions.

### Laws
The cats library provides a number of **lawful** type class _instances_. All type classes come with a set of implicit laws - laws that are not explicitly laid out in the type class itself. For example a `Semigroup` implementation must provide an associative binary `combine` operation. The name, number and type of parameters and type of return value of the operation is known from its signature:

```scala
def combine(a1:T, a2:T):T
```

What is not obvious from the above is the requirement that the combine operation be **associative**. You, as the library client could attempt to hand-craft an instance of `Semigroup[Int]` for the integer subtraction operation, perhaps something like this:

```scala
def combine(a1:Int, a2:Int):Int = a1 - a2
```

However the integer subtraction operation is not associative, hence a lawful `Semigroup` instance for it does not exist. This thing we created above is _not_ a semigroup, and it will not work as expected. Code may use an instance of `Semigroup[Int]` to `combine` while _folding_ **left** or **right**, assuming that the `Semigroup[Int]` instance adheres to the **associativity law**. 

[1]: https://en.wikipedia.org/wiki/Parametric_polymorphism#CITEREFPierce2002
[2]: http://typelevel.org/cats/
[3]: http://scalaz.github.io/scalaz/
[4]: http://typelevel.org/cats/typeclasses.html

