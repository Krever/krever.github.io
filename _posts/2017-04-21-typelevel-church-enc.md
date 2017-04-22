---
title:  "Scala Typelevel Church Encodings"
date:   2017-04-20 21:37:00
categories: [software-development]
tags: [scala, typelevel propgramming]
---

You have probably heard about [Lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus), right? Basically everything you have there is lambdas. And to do anything with them it is nice to use [Church encoding](https://en.wikipedia.org/wiki/Church_encoding), which is a way to represent natural numbers, booleans, conditionals and simple data structures using only lambdas. 

Another fact: in Scala we have [Higher-kinded types](https://en.wikipedia.org/wiki/Kind_(type_theory)) which are basically typelevel functions(they are one from another). So the question is: can we implement Church encoding using only types, so all the computations are done during compilation? Spoiler: **YES, WE CAN!**. Let's see how.

## Type Lambda
To start doing anything we need to declare something that will help us abstract over type functions:

```scala
// equivalent to something like type TL = TL => TL
trait TL {
  type Apply[T <: TL] <: TL
}
```
In lambda calculus everything is a function, so this why `TL#Apply` argument has upper bound of `TL`. Now we can do fancy stuff like this:

```scala
type func1 <: TL = ... 
type arg1 <: TL = ... 
type result = func1#Apply[arg1] // equiv. to val result = func1(arg1)
```
How can we use *"instantiate"* such type lambda? Here are some examples: 

```scala
// equiv. to val identity = (x:TL) => x
// λx.x
type Identity = TL {
  type Apply[T <: TL] = T
}
// equiv. to def const(a: TL) = (_:TL) => a
// λa.λt.a
type Const[A <: TL] = TL {
  type Apply[T <: TL] = A
}
```
	
## Church encoding examples
Here are some examples of how to implement elements of Church encoding:

### Bools
```scala
// λa.λb.a
type True = TL {
  type Apply[A <: TL] = TL {
    type Apply[B <: TL] = A
  }
}

// λa.λb.b
type False = TL {
  type Apply[A <: TL] = TL {
    type Apply[B <: TL] = B
  }
}

// λp.λq.p q p
type And = TL {
  type Apply[P <: TL] = TL {
    type Apply[Q <: TL] = P#Apply[Q]#Apply[P]
  }
}
```

### Natural numbers

```scala
// λf.λx.x
type Zero = TL {
  type Apply[F <: TL] = TL {
    type Apply[X <: TL] = X
  }
}

// λn.λf.λx.f (n f x)
type Succ = TL {
  type Apply[N <: TL] = TL {
    type Apply[F <: TL] = TL {
      type Apply[X <: TL] = F#Apply[N#Apply[F]#Apply[X]]
    }
  }
}

// λm.λn.λf.λ
type Plus = TL {
  type Apply[M <: TL] = TL {
    type Apply[N <: TL] = TL {
      type Apply[F <: TL] = TL {
        type Apply[X <: TL] = M#Apply[F]#Apply[N#Apply[F]#Apply[X]]
      }
    }
  }
}
```

## BORING !!!

Yes, this is boring, because once we know the pattern, implementing next elements is just translating text from wikipedia. So I have done that for you and almost all encodings are available on github as [Krever/sturch-encodings](https://github.com/Krever/sturch-encodings). If you're interested in details feel free to go there(it consists of 5 files, so don't be scared). The library also contains very simple reflection-based parser for such encodings(so we can print it or convert to realt int/boolean). 

The most complicated type available there is `Predecessor` defined like this:

```scala
// λn.λf.λx.n (λg.λh.h (g f)) (λu.x) (λv.v)
type Pred = TL {
  type Apply[N <: TL] = TL {
    type Apply[F <: TL] = TL {
      type Apply[X <: TL] = N#Apply[TL {
        type Apply[G <: TL] = TL {
          type Apply[H <: TL] = H#Apply[G#Apply[F]]
        }
      }]#Apply[TL {
        type Apply[U <: TL] = X
      }]#Apply[TL {
        type Apply[V <: TL] = V
      }]
    }
  }
}
```
If you don't understand, don't worry, neither do I. But it works :)

## What can we do with it?
I have to admit, this almost completely useless. Lambda Calculus on value level doesn't have many applications, and when we go to the type level it is even more impractical.

But anyway, it is fun and creating this was nice exercise in type level programming . With some additional syntactic sugars we can write code like this:

```scala 
import sturch.nats._
import sturch.bools._

type x = Plus[`3`, `4`]
type y = Mult[`3`, `2`]
type `IsTypelevelProgrammingFun?` = Not[EQ[x, y]]
type result = If[`IsTypelevelProgrammingFun?`, Mult[x, y], Zero]
```
So we can solve almost any problem without running our program! We don't have loops, but who needs them anyway. And one more time we have proven that Scala type system is turing complete.

