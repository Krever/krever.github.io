---
title:  "The Proof - Monad as a Monoid in Category of Endofunctors"
date:   2016-12-10 21:37:00
categories: [software-development]
tags: [scala, category-theory]
---

You have probably seen folllowing _Definition_ somewhere on your FP path.

> A monad is just a monoid in the category of endofunctors

If you're curious, as I am, you've probably even googled for some explaination of it. And you probably have found  some answer on SO or Quora... the math answer. So now you know the reasoning but probably your level of understanding have stayed on the exact same level, zero.

In the following post I will try to express this _Definition_ in simple Scala code. Little disclaimer: all of the following code could have been written using scalaz or cats libraries, but for clarity I will stick to the pure Scala.

### The Monoid
I assume you already have some knowledge from category theory, but we will make short remainder here. First, lets define a `Monoid`:

```scala
trait Monoid[T] {
  def zero: T
  def combine(a1: T, a2: T): T
}
```
As we can see, it allows us to combine two instances of type `T` and gives us zero element, also of type `T`. An example may be an `Int` with `0` and `+` operation.

```scala
object IntAddMonoid extends Monoid[Int] {
  def zero: Int = 0
  def combine(a1: Int, a2: Int): Int = a1 + a2
}
```

## The Category
Ok, that was the easy part. To go further we need to realize, that _Definition_ speaks about _monoid in category of ..._, so we need to know what it means. In fact what we have defined above is a _monoid in category of sets_ where every type is a set. Lets make it more abstract and start with defining a generic `MonoidalCategory`:

```scala
trait MonoidalCategory {
  type Morphism[F, G]
  type MonoidalProduct[F,G]
  type IdentityObject
}
```
I won't go deep into this, but I hope everything will get much clearer when we define our `CategoryOfSets`

```scala
trait CategoryOfSets extends MonoidalCategory {
  type Morphism[F, G] = F => G
  type MonoidalProduct[F,G] = (F, G)
  type IdentityObject = Unit
}
```
As we can see, `Morphism` is just traditional `Function`, that translates object of type `F` into object of type `G`. `MonoidalPoruct` is just a way to combine two elements of category, so in this case we are just making a `Tuple2` out of them. For now, we don't care much for the `IdentityObject`, but we can say, that `Unit` is an `IndentityObject` because `(F, Unit)` is equal to `F` in terms of possible values it can take.

### The Monoid in Category
Now we've got all the necesseary pieces to define  `MonoidInCategory` and `MonoidInCategoryOfSets`:

```scala
trait MonoidInCategory[T] {
  type Category <: MonoidalCategory
  def zero: Category#Morphism[Category#IdentityObject, T]
  def combine: Category#Morphism[Category#MonoidalProduct[T,T], T]
}

trait MonoidInCategoryOfSets[T] extends MonoidInCategory[T] {
  type Category = CategoryOfSets
  def zero: Unit => T
  def combine: ((T, T)) => T
}
```
So, our `zero` is function from `IdentityObject` to `T` and our `combine` is function from product of `T` and `T` to another instance of `T`.
Can you see that `MonoidInCategoryOfSets` is almost the same our firs `Monoid`? If not, here is and example for `Int` type with add operation:

```scala
object IntAddMonoidInCategoryOfSets extends MonoidInCategoryOfSets[Int] {
    def zero: Unit => Int = (a: Unit) => 0 
    def combine: ((Int, Int)) => Int = (t: (Int, Int)) => t._1 + t._2
}
```
Not so scary, is it?

### The Functor
Now we should define an endofunctor. We will simplify here and use the name `Functor` where we should use `EndofunctorInCategoryOfSets`:

```scala
trait Functor[F[_]] {
  def fmap[A, B](f: A => B): F[A] => F[B]
}
```
So `F[_]` is a `Functor` because it is a type constructor(kind `* -> *`) and it allows as to lift any function from `A` to `B` into function from `F[A]` to `F[B]`.

### The Category of Endofunctors
Now we would probably like to define a `CategoryOfEndofunctors` but for this we need few additional steps.That's because Scala doesn't provide kind polymorphism, so we cannot put type constructor `F[_]`(kind `* -> *`) when we require type `F`(kind `*`). 

```scala
trait Product[F[_], G[_]] {
  type Out[T]
}

trait MonoidalCategoryK2 {
  type Morphism[F[_], G[_]]
  type MonoidalProduct[F[_],G[_]] <: Product[F, G]
  type IndentityObject[T]
}
```
Okey, so we migrated our `MonoidalCategory` from kind `*` into kind `* -> *`. One thing that may be not obious here is the `Product`. When we were acting in kind `*` the product was just a type function(type constructor) from two types into new type, so it was `* -> * -> *`, and when aplied with two types(`*`), it resulted in type(`*`). But now we are acting in world of type constructors(`* -> *`) so we need to change `* -> * -> *` into `(* -> *) -> (* -> *) -> (* -> *)`, so when applied with two type constructors(`* -> *`) it will result with type constructor(`* -> *`). If it is not clear yet, we'll have an example soon.
Now lets go back to our mission of defining ` CategoryOfEndofunctors`:

```scala
case class Id[T](value: T)

trait NaturalTransformation[-F[_], +G[_]] { self =>
  def apply[A](fa: F[A]): G[A]
}
type ~~>[-F[_], +G[_]] = NaturalTransformation[F, G]

trait Compose[F[_], G[_]] extends Product[F,G]{
  type Out[T] = F[G[T]]
}
```
Okey, so we've defined three things here. At first we have simple case class that we will use in a moment. Second goes the `NaturalTransformation` and it is just a function from `F[T]` to `G[T]` for any type `T`(we also introduce a type alias `~~>` that we will use to make things shorter). Then we have an example of a product type that just composes the type constructors. Here is a small example to make it more real:

```scala 
import scala.util.Try
object TryToOption extends NaturalTransformation[Try, Option] {
  def apply[T](t: Try[T]): Option[T] = t.toOption
}
def example[T](v: T): Compose[Option, Try]#Out[T] = {
  val result: Option[Try[T]] = Option(Try(v))
  result
}
```
And finally, here comes our `CategoryOfEndofunctors`!

```scala
trait CategoryOfEndofunctors extends MonoidalCategoryK2{
  type Morphism[F[_], G[_]] = NaturalTransformation[F, G]
  type MonoidalProduct[F[_],G[_]] = Compose[F,G]
  type IndentityObject[T] = Id[T]
}
```
As we can see, it's just assembled with little elements we've already definded.

### The Monoid in Category of Endofunctors
As you may feel, we are coming with big steps to the `MonoidInCategoryOfEndofunctors` but there is this one thing on our way: we need to migrate `MonoidInCategory` from `MonoidalCategory` into `MonoidalCategoryK2` to make scala compiler happy.

```scala
trait MonoidInCategoryK2[T[_]] {
  type Category <: MonoidalCategoryK2
  def zero: Category#Morphism[Category#IndentityObject, T]
  def combine: Category#Morphism[Category#MonoidalProduct[T, T]#Out, T]
}
```
And here it comes, THE BIG PLAYER:

```scala
trait MonoidInCategoryOfEndofunctors[F[_]] extends MonoidInCategoryK2[F] {
  type Category = CategoryOfEndofunctors
  def zero: Id ~~> F
  def combine: Compose[F,F]#Out ~~> F
  def functor: Functor[F]
}
```
Pretty, isn't it? And not so complicated. Our `zero` is now a `NaturalTransformation`(instead of a `Function`) from `Id`(instead of an `Unit`) to `F[_]`. Our combine is also a `NaturalTransformation` from `F[F[_]]` to `F[_]`. And one more thing, we need an evidence, that `F[_]` is a `Functor` so we have this little function called `functor`.
Maybe a little example of defining such a magnificent beeing?

```scala
object OptionMonoid extends MonoidInCategoryOfEndofunctors[Option] {
  override def zero: Id ~~> Option = new NaturalTransformation[Id, Option]{
    def apply[T](id: Id[T]): Option[T] = Some(id.value)
  }

  override def combine: Compose[Option,Option]#Out ~~> Option = 
    new NaturalTransformation[Compose[Option,Option]#Out, Option]{
      def apply[T](opt: Option[Option[T]]): Option[T] = opt.flatten
    }

  override def functor: Functor[Option] = new Functor[Option] {
    override def fmap[A, B](f: (A) => B): (Option[A]) => Option[B] = {
      case Some(x) => Some(f(x))
      case None => None
    }
  }
}
```

### The Monad
Now, when we have dealt with right side of the equasion, lets got back to the left side. Here is a definition of `Monad`:

```scala
trait Monad[M[_]] {
    def unit[T](v: T): M[T]
    def bind[T, U](m: M[T], f: T => M[U]): M[U]
}
```
And some example:

```scala
object OptionMonad extends Monad[Option] {
  override def unit[T](v: T): Option[T] = Some(v)
  override def bind[T, U](m: Option[T], f: (T) => Option[U]): Option[U] = m.flatMap(f)
}
```

## The Proof
If the _Definition_ is true, as the math guys say, we should be able to define following functions:

```scala
def fromMonoidToMonad[M[_]](monoid: MonoidInCategoryOfEndofunctors[M]): Monad[M]
def fromMonadToMonoid[M[_]](monad: Monad[M[_]]): MonoidInCategoryOfEndofunctors[M]
```
and in fact we are! Here is the first one:

```scala 
def fromMonoidToMonad[M[_]](monoid: MonoidInCategoryOfEndofunctors[M]): Monad[M] = {
  new Monad[M]{
    override def unit[A](v: A): M[A] = monoid.zero(Id(v))
    override def bind[A, B](m: M[A], f: (A) => M[B]): M[B] = monoid.combine(monoid.functor.fmap(f)(m))
  }
}
```
Pretty short and in fact rather simple. To put value of type `A` inside `M[A]` it's enough to put it inside `Id` and tranform it with `monoid.zero`. To bind value `m` of type `M[A]` with function `f` from `A` to `M[B]` we can lift `f` with `monoid.functor.fmap` so now it is `M[A] => M[M[B]]`. Then we apply it with `m` and flatten with `monoid.combine`. No magic attached.
And the second one, a bit longer I've to admit:

```scala
def fromMonadToMonoid[M[_]](monad: Monad[M]): MonoidInCategoryOfEndofunctors[M] = {
  new MonoidInCategoryOfEndofunctors[M] {
    override def zero: Id ~~> M = new (Id ~~> M) {
      override def apply[A](a: Id[A]): M[A] = monad.unit(a.value)
    }
    
    override def combine: Compose[M, M]#Out ~~> M = new (Compose[M, M]#Out ~~> M) {
      override def apply[A](m: M[M[A]]): M[A] = monad.bind(m, identity[M[A]])
    }
    
    override def functor: Functor[M] = new Functor[M] {
      override def fmap[A, B](f: (A) => B): (M[A]) => M[B] = (m: M[A]) => {
        monad.bind(m, f.andThen(monad.unit))
      }
    }
  }
}
```
Few words of explaination. To transform `a`, instance of `Id[A]`, into instance of `M[A]` we just lift the content of `Id` with `monad.unit`. To flatten(combine) `M[M[A]]` it is enough to bind it with identity functions. To create a `Functor` instance, we define `fmap` as function that binds it argument with function that is composition of `f` and `monad.unit`.

### The summary
I hope this article was clear enough, and know you understand what this incomprehensible _Definition_ means. Sadly, I don't have any formal education in field of category theory, but I hope there are no obvious errors. Because all the code attached compiles, I may be sure that the proof is correct as long as defined types are correct.

One thing I have to confess is that I knowingly ignored all aspects of laws.For example `Monad` has to obey these three monadic laws, and `Monoid` has some laws of his own(just as a `Functor` does). You may treat this as a homework, to prove if they hold :)
