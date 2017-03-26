---
title:  "Typeclass: when inheritance is not enough"
date:   2017-03-25 21:37:00
categories: [software-development]
tags: [scala]
---

If you ask Wikipedia about polymorphism, it will tell you that it is about *"provisioning a single interface to entities of different types"*. And this is true(remember that interface here and in whole post means API, not some element that is part of the language, like scala's `trait`). In the following post we will go through a series of cases where traditional way of implementing polymorphism(by which I mean inheritance) is not enough . 

### Basic example
Just to remind what is the typeclass, let's look at the same interface encoded with inheritance and typeclass:

```scala
//inheritance
trait Hello {
    def hello: String
}
class World(name: String) extends Hello {
    def hello: String = "Hello "+name
}
new World("Earth").hello
```

```scala
//typeclass
trait Hello[T] {
    def hello(t: T): String
}
class World(val name: String)
val helloWorld = new Hello[World] {
    def hello(t: World): String = "Hello "+t.name
}
helloWorld.hello(new World("Earth"))    
```

As we can see inheritance variant is less verbose and nicer to use. So why would anybody want to use typeclasses anyway? Here are some reasons...
                                
### Problem 1: Derive interface for new types automatically 
Imagine you have following code already there. Nothing sophisticated, one interface that says *"you can convert something to `String`"*, and few methods depending on it.

```scala
trait Serializable {
    def serialize: String
}

def writeToFile(data: Serializable,  file: File)
def writeToSocket(data: Serializable,  socket: Socket)
def print(data: Serializable)
```

But after some time you add a json support:

```scala
trait Json {
    def write: String
}
trait Jsonable {
    def toJson: Json
}
case class User(name:String, age: Int) extends Jsonable { ... }
```

And then you would like to use the old api without modyfing it. Just like this:

```scala
print(User("John", 18))
```

Sadly, it's not possible, because `User` doesn't implement `Serializable`, although it can be easily converted to string through `Json` abstraction. Ther are three ways to go: 

- create 3 new overloads taking `Jsonable` interface instead of `Serializable` - this is obviously wrong, as it doesn't scale
- make `User` implement `Serializable` - this is not a good solution, because you don't always have access to particular type, and even if you do, you have to do this for every type that is `Jsonable` and not `Serializable` 
- create conversion method `jsonableToSerializable` - this is better approach, but you loose you original type during such conversion and also(at least with explicit conversion) you have to call this method explicitly with every use of original API. If you go with implicit conversion, it is almost good,  but it has may limitations(e.g. cannot be chained, like `Jsonable` -> `Serializable` -> `SomethingElse`)
    
Lets see how such scenario can look with typeclass encoding:
        
```scala
trait Serializable[T]{
    def serialize(t: T): String
}
def writeToFile[T: Serializable](data: T,  file: File)
def writeToSocket[T: Serializable](data: T,  socket: Socket)
def print[T: Serializable](data: T)
                                 
trait Json {
    def write: String
}
trait Jsonable[T] {
    def toJson(v: T): Json
}
case class User(name:String, age: Int)
implicit val userJsonable: Jsonable[User] = ???
                                         
implicit def jsonableToSerializable[T: Jsonable]: Serializable[T] = new Serializable[T] {
    def serialize(t: T): String = implicitly[Jsonable[T]].toJson(t).write
}
                                             
print(User("John", 18))
```

### Problem 2: Provide multiple implementations
You want to compare objects. But you want to do this in multiple ways in different contexts. This cannot be done with inheritance, so we will go with typeclass encoding right away.

```scala
trait Equatable[T] {
    def equals(a: T, b: T): Boolean
}
case class User(name:String, age: Int)
val eq1: Equatable[User] = ???
val eq2: Equatable[User] = ???
```
                                                 
### Problem 3: Define external interface for external type 
This is very common(at least for me) situation. We have a type defined in module/library A 

```scala
case class User(name:String, age: Int)
```

and an interface defined in module/library B:

```scala
trait Equatable {
    def equals(a: Any): Boolean
}
```

now we would like to define `User` as `Equatable`. This cannot be done with inheritance and is straight forward with typeclasses.

```scala
// defined in module A
trait Equatable[T] {
    def equals(a: T, b: T): Boolean
}
// defined in module B
case class User(name:String, age: Int)
// defined in module C
implicit val userEq: Equatable[User] = ???
```
	
#### Problem 3.5: Define interface for simple types
There is no way to inherit from simple types like `Int` or `Float` so there is no way to make them conform to any interface by inheritance. Happily typeclasses doesn't have this problem:

```scala
implicit val intEq: Equatable[Int] = ???
```

### Problem 4: Get exact type that implements the interface
Let's create `Sortable` interface, that will tell us that given collection can be sorted. The best we can do with pure inheritance is:

```scala
trait Sortable {
    def sort(): Sortable //or Any
}
```

Not very useful because we loose all type information after sorting. It gets a little better if we add some generics:

```scala
trait Sortable[Coll] {
    def sort(): Coll 
}
```

But still, there is no way to be sure that output collection is exactly the same type as output one(we can create `class List extends Sortable[Set]`). To solve this we have to use typeclass that looks almost exactly the same:

```scala
trait Sortable[Coll] {
    def sort(coll: Coll): Coll 
}
```

### Problem 5: Implement multiple interfaces with dependencies
Imagine, that to make something `Jsonable`, we have to provide some dependency object, and the same goes with `Persistent`

```scala
abstract class Jsonable(jsonFactory: JsonFactory) { ... }
abstract class Persistent(persistenceCtx: PersistenceContext) { ... }
																	 
case class User(username: String, age:Int) // no way to inherit both, at least until dotty is here
```

So again, we fallback to typeclasses:

```scala
abstract class Jsonable[T](jsonFactory: JsonFactory) { ... }
abstract class Persistent[T](persistenceCtx: PersistenceContext) { ... }

case class User(username: String, age:Int) 
implicit val userJsonable = new Jsonable[User](???)
implicit val userPersistent = new Persistent[User](???)
```
																	 
### Problem 6: Define static function
We would like to define an interface that allows to create an instance of given type from `String` and, as you probably have guessed by now, it cannot be done with inheritance, because to inherit some behavior you need and instance already. With typeclass it is straight forward:

```scala
trait FromString[T] {
    def create(str: String): T 
}
```

## Summary 
We have seen some examples, where typeclasses show their expressive power. They are great tools, but I would like to advise you to use them carefully. There are some problems with typeclasses, and two greatest ones are:
																		 
- they are not as widely understood as inheritance, so if you are working with some Java developers, make sure they understand the idea
- implicits can become a head ache very easily and you can't use typeclasses efficiently without implicits
																		 
So remember good old Uncle Ben:
																		 
> With great power comes great responsibility
																		 
