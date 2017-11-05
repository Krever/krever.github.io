---
title:  "MapReduce, Cats, collection-strawman and Object Algebras"
date:   2017-11-03 21:37:00
categories: [software-development]
tags: [scala, bigdata]
---

5 months ago John DeGoes wrote nice tweet about expressing map-reduce model as haskell function signature.
In this article we will see how to translate it to scala and how far we can push it's polimorphism.

Sadly I wasn't able to find original tweet today, but f I remember correctly that's the essence i:

```haskell
mapReduce :: (A -> [(B, C)]) -> (B -> [C] -> D) -> [A] -> [D]  
```

Although this is completely correct, it's not completely readable for humans. So I wrote a short response:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">For this of us who don&#39;t speak mathish:<br><br>mapReduce :: (record -&gt; [(key, mapped)]) -&gt; (key -&gt; [mapped] -&gt; reduced) -&gt; [record] -&gt; [reduced]</p>&mdash; Wojtek Pituła (@Krever01) <a href="https://twitter.com/Krever01/status/872377965547573248?ref_src=twsrc%5Etfw">June 7, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Haskell explained

Here is the relevant content with highlighting:

```haskell
mapReduce :: (Record -> [(Key, Mapped)]) -> (Key -> [Mapped] -> Reduced) -> [Record] -> [Reduced]
```

So what is really happening here is that we take the mapping function (mapper) `(Record -> [(Key, Mapped)])`
 and reducing function (reducer) `(Key -> [Mapped] -> Reduced)` and return the function that will take the set of
 input records and transform them to the set of output records `[Record] -> [Reduced]`
 
## Let's do it in scala

While haskell version is extremely elegant I'm a Scala fan, so lets translate it.

```scala
  def mapReduce[mK, mV, rK, rV, oK, oV](
    map: (mK, mV) => Seq[(rK, rV)],
    reduce: (rK, Seq[rV]) => Seq[(oK, oV)]
  ): Seq[(mK, mV)] => Seq[(oK, oV)]
```
I'm cheating a bit, because I modified the semantics so the input is a key-value pair instead of simple type, this gets us closer 
to the Hadoop implementation. We got three key-value pairs, mapper input `(mK, mV)`, reducer input `(rK, rV)` and output `(oK, oV)`
Now lets see how can we implement and run this method.

```scala
def mapReduce[mK, mV, rK, rV, oK, oV](
  map: (mK, mV) => Seq[(rK, rV)],
  reduce: (rK, Seq[rV]) => Seq[(oK, oV)]
): Seq[(mK, mV)] => Seq[(oK, oV)] =
  _
    .flatMap(map.tupled)
    .groupBy(_._1)
    .flatMap { case (key, values) => reduce(key, values.map(_._2)) }
    .toSeq 


val wordCount = mapReduce(
  (key: Any, value: String) => value.split(" ").map(x => (x, 1)),
  (key: String, values: Seq[Int]) => Seq((key, values.sum))
)
// wordCount: Seq[(Any, String)] => Seq[(String, Int)] = $$Lambda$1502/1086741722@1176f3cd

wordCount(Seq((null, "x y z"), (null, "x y")))
// res1: Seq[(String, Int)] = Vector((z,1), (y,2), (x,2))
```

So we could stop at this point, but lets go a little further and see what else can we do with this signature.

## Generalization

Let's try to make our implementation agnostic about the type of collection we are using and make it able to work with
`List`s, `Set`s and other.

### Typeclasses
The first approach uses `cats` typeclasses to make sure chosen collection delivers all required operations.

```scala
import cats._
import cats.implicits._

class MapReduce[Coll[_]](implicit
  val collMonad: Monad[Coll],
  val collFoldable: Foldable[Coll],
  val collMonoid: MonoidK[Coll]
) {

  override def mapReduce[mK, mV, rK, rV, oK, oV](
    mapF: (mK, mV) => Coll[(rK, rV)],
    reduceF: (rK, Seq[rV]) => Coll[(oK, oV)]
  ): Coll[(mK, mV)] => Coll[(oK, oV)] = {
    input => {
      val mappped: Coll[(rK, rV)] = collMonad.flatMap(input)(mapF.tupled)
      val grouped: Map[rK, Seq[rV]] = mappped.foldLeft(Map[rK, Seq[rV]]().withDefaultValue(Seq())) {
        case (state, (key, value)) => state.updated(key, state(key) ++ Seq(value))
      }
      val reduced: Coll[(oK, oV)] = grouped.foldLeft(collMonoid.empty[(oK, oV)]) {
        case (state, (key, values)) => collMonoid.combineK(state, reduceF(key, values))
      }
      reduced
    }
  }
}


val wordCount = new MapReduce[List]().mapReduce(
  (key: Any, value: String) => value.split(" ").map(x => (x, 1)).toList,
  (key: String, values: List[Int]) => List((key, values.sum))
)
wordCount(List((null, "x y z"), (null, "x y")))
```

This implementation is by no means perfect, but works well enough. It got a bit more complicated though.

### Weaker types

Second approach uses least specific type possible from standard library, which turns out to be `Iterable`. To make it more interesting we 
will use scala 2.13 and upcoming collection-strawman.

```scala
def mapReduce[mK, mV, rK, rV, oK, oV](
    map: (mK, mV) => Iterable[(rK, rV)],
    reduce: (rK, Iterable[rV]) => Iterable[(oK, oV)]
  ): Iterable[(mK, mV)] => Iterable[(oK, oV)] =
    _
      .flatMap(map.tupled)
      .groupMap(_._1)(_._2)
      .toSeq
      .flatMap(reduce.tupled)

  val wordCount = mapReduce(
    (key: Any, value: String) => new ArrayOps(value.split(" ").map(x => (x, 1))).toSeq,
    (key: String, values: Iterable[Int]) => Seq((key, values.sum))
  )

  wordCount(Seq((null, "x y z"), (null, "x y")))
  wordCount(Set((null, "x y z"), (null, "x y")))
``` 

Here we have gave up the control over the context in which we perform the computations, so the result type is always `Iterable`. 
Thanks to that we gained bigger space of usages but have to convert the result to know for sure what collection we get.
This solution is not equivalent to typeclass-based but is simpler and may be good enough for some cases.

### Hadoop MapReduce

Now lets see how can we reuse the original signature to create something usable from Hadoop MR point of view.
Instead of returning function we will return a pair of hadoop `Mapper` and `Reducer`.

```scala
import java.lang
import org.apache.hadoop.mapreduce.{Mapper, Reducer}

def mapReduce[mK, mV, rK, rV, oK, oV](
  mapF: (mK, mV) => Seq[(rK, rV)],
  reduceF: (rK, Seq[rV]) => Seq[(oK, oV)]
): (Mapper[mK, mV, rK, rV], Reducer[rK, rV, oK, oV]) = {
  val mapper = new Mapper[mK, mV, rK, rV] {
    override def map(key: mK, value: mV, context: Mapper[mK, mV, rK, rV]#Context): Unit = {
      mapF(key, value).foreach((context.write _).tupled)
    }
  }
  val reducer = new Reducer[rK, rV, oK, oV] {
    override def reduce(key: rK, values: lang.Iterable[rV], context: Reducer[rK, rV, oK, oV]#Context): Unit = {
      import scala.collection.JavaConverters._
      reduceF(key, values.asScala.toSeq).foreach((context.write _).tupled)
    }
  }
  (mapper, reducer)
}
```

There are few problems with this implementation:

* MR API doesn't allow passing mapper and reducer as values, they have to have static class names.
* The types being used as input and output have to implement `Writable` interface, but this is not enforced on signatures level.

Lets forget about these problems for a moment, because there is one more interesting thing we can do here!

## Unifying

We have seen following implementations of the map-reduce model:

* simple, `Seq`-based
* collection agnostic, typeclass-based
* generic but weak, `Iterable`-based
* hadoop-specific

They had slightly different signatures but maybe we can express them as one abstraction which will allow us to specify 
a word-count, and then leave the implementation details to the user? Let's try that!

We will start with defining common interface for all the implementations. This interface will define our vocabulary
(methods and types) which can also be called _algebra_.

```scala
trait MapReduceAlg {

  type Coll[T]

  type Output[mK, mV, rK, rV, oK, oV]

  def mapReduce[mK, mV, rK, rV, oK, oV](
    mapF: (mK, mV) => Coll[(rK, rV)],
    reduceF: (rK, Seq[rV]) => Coll[(oK, oV)]
  ): Output[mK, mV, rK, rV, oK, oV]

  def fromIterable[A](it: Iterable[A]): Coll[A]
}
```

Next, lets define our application, which will be a simple wordcount as seen before. We will inherit from the algebra trait
and use the exposed vocabulary to define one more member.

```scala
trait WordCount extends MapReduceAlg {

  val wordCount: Output[Any, String, String, Int, String, Int] = mapReduce(
    (key: Any, value: String) => fromIterable(value.split(" ").map(x => (x, 1))),
    (key: String, values: Seq[Int]) => fromIterable(Seq((key, values.sum)))
  )

}
```

We have the wordcount but we don't know what it really is. To find out we will need _interpreters_.


```scala
trait SimpleImpl extends MapReduceAlg {

  type Coll[T] = Seq[T]
  type Output[mK, mV, rK, rV, oK, oV] = Seq[(mK, mV)] => Seq[(oK, oV)]

  override def mapReduce[mK, mV, rK, rV, oK, oV](
    mapF: (mK, mV) => Seq[(rK, rV)],
    reduceF: (rK, Seq[rV]) => Seq[(oK, oV)]
  ): (Seq[(mK, mV)]) => Seq[(oK, oV)] = {
    _
      .flatMap(mapF.tupled)
      .groupBy(_._1)
      .flatMap { case (key, values) => reduceF(key, values.map(_._2)) }
      .toSeq

  }

  override def fromIterable[A](it: Iterable[A]): Seq[A] = it.toSeq
}
```

```scala
import cats._
import cats.implicits._
  
class TypeclassImpl[CC[_]](implicit
  val collMonad: Monad[CC],
  val collFoldable: Foldable[CC],
  val collMonoid: MonoidK[CC]
) extends MapReduce {

  type Coll[T] = CC[T]
  type Output[mK, mV, rK, rV, oK, oV] = CC[(mK, mV)] => CC[(oK, oV)]

  override def mapReduce[mK, mV, rK, rV, oK, oV](
    mapF: (mK, mV) => Coll[(rK, rV)],
    reduceF: (rK, Seq[rV]) => Coll[(oK, oV)]
  ): Coll[(mK, mV)] => Coll[(oK, oV)] = {
    input => {
      val mappped: Coll[(rK, rV)] = collMonad.flatMap(input)(mapF.tupled)
      val grouped: Map[rK, Seq[rV]] = mappped.foldLeft(Map[rK, Seq[rV]]().withDefaultValue(Seq())) {
        case (state, (key, value)) => state.updated(key, state(key) ++ Seq(value))
      }
      val reduced: Coll[(oK, oV)] = grouped.foldLeft(collMonoid.empty[(oK, oV)]) {
        case (state, (key, values)) => collMonoid.combineK(state, reduceF(key, values))
      }
      reduced
    }
  }

  override def fromIterable[A](it: Iterable[A]): Coll[A] = it.foldLeft(collMonoid.empty[A]){
    case (coll, x) => collMonoid.combineK(coll, collMonad.pure(x))
  }
}
```

```scala
trait HadoopImpl extends MapReduceAlg {
  import java.lang
  import org.apache.hadoop.mapreduce.{Mapper, Reducer}

  type Coll[T] = Seq[T]

  type Output[mK, mV, rK, rV, oK, oV] = (Mapper[mK, mV, rK, rV], Reducer[rK, rV, oK, oV])

  override def mapReduce[mK, mV, rK, rV, oK, oV](
    mapF: (mK, mV) => Seq[(rK, rV)],
    reduceF: (rK, Seq[rV]) => Seq[(oK, oV)]
  ): (Mapper[mK, mV, rK, rV], Reducer[rK, rV, oK, oV]) = {
    val mapper = new Mapper[mK, mV, rK, rV] {
      override def map(key: mK, value: mV, context: Mapper[mK, mV, rK, rV]#Context): Unit = {
        mapF(key, value).foreach((context.write _).tupled)
      }
    }
    val reducer = new Reducer[rK, rV, oK, oV] {
      override def reduce(key: rK, values: lang.Iterable[rV], context: Reducer[rK, rV, oK, oV]#Context): Unit = {
        import scala.collection.JavaConverters._
        reduceF(key, values.asScala.toSeq).foreach((context.write _).tupled)
      }
    }
    (mapper, reducer)
  }

  override def fromIterable[A](it: Iterable[A]): Seq[A] = it.toSeq
}
```

That was quite easy, we just implemented the missing members in some concrete ways. 
With interpreters in place we can finally use our wordcount. 
It's enough to just mix in the interpreter to the application definition to get some valuable piece of code.

```scala
object SimpleImplTest {

  def main(args: Array[String]): Unit = {
    val impl = new WordCount with SimpleImpl
    val wc = impl.wordCount(Seq((null, "a b c"), (null, "a b")))
    println(wc)
  }
}
``` 

```scala
object TypeclassImplTest {

  def main(args: Array[String]): Unit = {
    import cats.implicits._
    val impl = new TypeclassImpl[List] with WordCount
    val wc = impl.wordCount(List((null, "a b c"), (null, "a b")))
    println(wc)
  }

}
```

```scala
object HadoopImplTest {

  def main(args: Array[String]): Unit = {
    val impl = new WordCount with HadoopImpl
    val wc: (Mapper[Any, String, String, Int], Reducer[String, Int, String, Int]) = impl.wordCount
    println(wc)
  }
}
```

The technique used here for unifying different implementations of the same problem is called `Object Algebras`.
It is used as a ground concept in [julienrf/endpoints](https://github.com/julienrf/endpoints/) library. 
This article is just an example of using this technique and if you are curious what are the benefits and problems with it I encourage 
to study it further. 
Maybe I will have chance to present this concept more deeply on one of the scala conferences soon.
