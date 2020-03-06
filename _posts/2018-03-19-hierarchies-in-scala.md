---
layout: post
title:  Hierarchies in Scala
date:   2018-03-19 
categories: scala types generics
---

# Hierarchies in Scala

In the last couple of weeks I have tried to come to grips with _Scala_ and I
have to come to love it. One thing I have been struggling with is the
combination of types and generics and for the accidental visitor I will record
my findings in the following.

## Preliminaries

First some initial concepts, that we need to use later:


`class`  
- Typical Object Oriented structure, they contain methods, values and include
    one or multiple constructors.

`abstract class`  
- Similar to classes, but can contain abstract methods (`abstract def`), which have to be defined in classes that inherit from this one using `override def`.

`case class`  
- Can be used in Scala's pattern matching.
- Variables in the constructor are public.
- State inside of the class should depend solely on its constructor arguments.

`trait`  
- Supports polymorphism using the `with` keyword: `class X extends A with B`,
    where _A_ can be a trait or class, but _B_ has to be a class.
- Can define methods, but no constructor.
- Only interoperable with Java if it does not include method implementations.
    (In this case it is similar to interfaces in Java.)

`object`
- Objects are singletons.
- Objects can inherit from traits.
- When an object has the same name as a class, it becomes the _companion object_
    of the class and can access its private members.

## Inheritance

Similar to other high-level languages Scala supports inheritance:

    {% highlight scala %}
    class Vehicle
    class Car extends Vehicle
    class Bike extends Vehicle
    class MotorBike extends Vehicle
    class Porsche extends Car
    class Jeep extends Car
    {% endhighlight %}

## Generics

Scala also supports generics meaning that a class can take a type parameter, to
define what type of value it stores.

    {% highlight scala %}
    class Container[T](val value : T)

    val bikeContainer : Container[Bike] = new Container(new Bike)
    val carContainer : Container[Car] = new Container(new Car)
    {% endhighlight %}

The type of a type paramter can bounded similar to `extends`. 

    {% highlight scala %}
    class CarContainer[T <: Car](val value : T)
    {% endhighlight %}


The `<:` here is a short hand for `extends`, which is specifically used for type
parameters.
Similarly we can define an _is-parent_ relationship with `>:`. So if we would
want to define an abstract container, we could define:

    {% highlight scala %}
    class AbstractContainer[T >: Porsche]
    {% endhighlight %}

## Inheritance + Generics Covariance = Variance

When we play around with Containers, we discover that we cannot easily convert
between a container of type car and a container of type vehicle, even though
car is a subtype of vehicle:

    {% highlight scala %}
    scala> val c : Container[Porsche] = new Container(new Porsche)
    c: Container[Porsche] = Container@1da748f8

    scala> val c2 : Container[Car] = c
    <console>:18: error: type mismatch;
    found   : Container[Porsche]
    required: Container[Car]
    Note: Porsche <: Car, but class Container is invariant in type T.
    You may wish to define T as +T instead. (SLS 4.5)
        val c2 : Container[Car] = c
                                  ^
    {% endhighlight %}

The compiler gives us the right hint. Basically we want to tell the compiler
that if `Car <: Vehicle` then `Container[Car] <: Vehicle[Car]`. This concept is
called Covariance. For this we need to define:


    {% highlight scala %}
    class Container[+T](val value : T)
    {% endhighlight %}

And then it will compile fine:

    {% highlight scala %}
    scala> val c : Container[Porsche] = new Container(new Porsche)
    c: Container[Porsche] = Container@41a9793e

    scala> val c2 : Container[Car] = c
    c2: Container[Car] = Container@41a9793e
    {% endhighlight %}

Going further we might want to convert between `CarContainer` and `Container`. To do
this, we make `CarContainer` a subtype of `Car`:

    {% highlight scala %}
    class CarContainer[T <: Car](override val value : T) extends Container[Car](value)
    {% endhighlight %}

And then this too works:

    {% highlight scala %}
    scala> val c3 = new CarContainer(new Porsche)
    c3: CarContainer[Porsche] = CarContainer@2758386b

    scala> val c4 : Container[Car] = c3
    c4: Container[Car] = CarContainer@2758386b
    {% endhighlight %}

Seems easy right? Well, what happens when we want to be able to replace the
vehicle stored in the container? We could simply change the constructor variable
from immutable to a mutable one:


    {% highlight scala %}
    class Container[+T](var value : T)

    error: covariant type T occurs in contravariant position in type T of value value_=
           class Container[+T](var value : T) { }
                                   ^
    {% endhighlight %}

Explain:
- Liskov substitution principle
- Its effect on functions
- How we in principle can resolve this issue
- How in this case we cannot
- How maybe this is for the better

## TODO

- CoVariance: Think of reasons why it should not be possible (Liskov-substitution
        principles)
- Is it possible to store multiple instances of classes parametrized by
    different numeric types?
- Do type classes or implicits provide an alternative solution




## Small problems

### Summing a list of arbitrary numbers

Now all of my inquiries started by the wall I hit when trying to accomplish a
few simple tasks:

I wanted to sum a list of numbers, which are of different type. My first
attempt did not compile:

    {% highlight scala %}
    scala> val l : List[Number] = List(1, 2.5)
    scala> l.sum
    <console>:13: error: could not find implicit value for parameter num: Numeric[Number]
        l.sum
    {% endhighlight %}
            ^
The reason for this is that Scala's `Int` and `Double` types do not inherit from a
common numeric supertype. `Number` is in fact a type from Java and while it is
easy to convert Scala numbers to `Number`, it does not do you any good, because
you cannot process the numbers.

Instead from looking at Scala's type hierachy, that while a numeric super type
does not exist, numbers are implicitly converted from any other number format to `Double`.

So we can write: 

    {% highlight scala %}
    scala> val l = List(1, 2.4, 3)
    l: List[Double] = List(1.0, 2.4, 3.0)
    {% endhighlight %}

So far so good. We can however not pass a List of _Int_ to a List of _Double_:

    {% highlight scala %}
    scala> val l1 = List(1, 2, 3)
    l1: List[Int] = List(1, 2, 3)

    scala> val l2: Double = l1
    <console>:12: error: type mismatch;
    found   : List[Int]
    required: List[Double]
        val l2 : List[Double] = l1
                                ^
    {% endhighlight %}

To make this work we either have to explicitly map each _Int_ to a _Double_ or
use implicits. (I used the code sample provided [here](https://stackoverflow.com/questions/14061134/implicit-conversion-from-listint-to-listdouble-fails#14061166))

    {% highlight scala %}
    scala> val l1 = List(1, 2, 3)
    l1: List[Int] = List(1, 2, 3)

    scala> val l2 : List[Double] = l1
    <console>:12: error: type mismatch;
    found   : List[Int]
    required: List[Double]
        val l2 : List[Double] = l1
                                ^

    scala> val l1 = List(1, 2, 3)
    l1: List[Int] = List(1, 2, 3)

    scala> val l2 : List[Double] = l1.map(_.toDouble)
    l2: List[Double] = List(1.0, 2.0, 3.0)

    scala> import scala.language.implicitConversions
    import scala.language.implicitConversions

    scala> implicit def il2dl(il: List[Int]): List[Double] = il.map(_.toDouble)
    il2dl: (il: List[Int])List[Double]

    scala> val l2 : List[Double] = l1
    l2: List[Double] = List(1.0, 2.0, 3.0)
    {% endhighlight %}

This does of course not mean that we can expect the same to work for the
concatenation of _List[Int]_ to _List[Double]_

    {% highlight scala %}
    scala> val l1 = List(1, 2, 3)
    l1: List[Int] = List(1, 2, 3)

    scala> val l2 = List(4.5, 5.5, 6.5)
    l2: List[Double] = List(4.5, 5.5, 6.5)

    scala> l1 ++ l2
    res0: List[AnyVal] = List(1, 2, 3, 4.5, 5.5, 6.5)
    {% endhighlight %}

Here we get an AnyVal, that we cannot perform any numeric operations over.
Making this work requires a whole other deal of work.


### Dealing with type erasure

I am trying to extract values from the following Java class hierarchy in Scala:

    {% highlight java %}
    public interface Metric {}
    public interface Gauge<T> extends Metric {
        T getValue();
    }
    public interface Counter extends Metric {
        long getCount();
    }
    {% endhighlight %}

Take note that _T_ can be of any type. However matching type with Scala's
pattern matching fails, due to type erasure.

    {% highlight scala %}
    var counters : Map[String,Counter] = Map()
    var gauges : Map[String,Gauge[Number]] = Map()

    override def notifyOfAddedMetric(metric: Metric, metricName: String, group: MetricGroup): Unit = {
        metric match {
            // WARNING: non-variable type argument Number in type pattern org.apache.flink.metrics.Gauge[Number] is unchecked since it is eliminated by erasure
            case x: Gauge[Number] => gauges += (fullName -> x)
            case x: Counter => counters += (fullName -> x)
            case _ => LOG.warn("...")
        }
    }
    {% endhighlight %}

It will also throw an exception at runtime, when Gauge[String]. Can we instead
match on the value inside of Gauge?

    {% highlight scala %}
    var counters : Map[String,Counter] = Map()
    var gauges : Map[String,Gauge[Number]] = Map()

    override def notifyOfAddedMetric(metric: Metric, metricName: String, group: MetricGroup): Unit = {
        metric match {
            case x: Gauge[Number] =>
                x.getValue match {
                    case _: Number => gauges += (fullName -> x)
                    case _ => LOG.warn("Did not register metric " + fullName)
                }
            case x: Counter => counters += (fullName -> x)
            case _ => LOG.warn("...")
        }
    }
    {% endhighlight %}

No, this does not work either. Scala throws an _java.lang.ClassCastException_.
After all getValue should return a number and a String is not a number and even
though this information should be discarded at runtime, this seems to be a
problem.

Well what if we use Any instead of Number, a common supertype of both Number and
String?

    [error]   Expression does not convert to assignment because:
    [error]     type mismatch;
    [error]      found   : org.apache.flink.metrics.Gauge[Any]
    [error]      required: org.apache.flink.metrics.Gauge[Number]
    [error]     Note: Any >: Number, but Java-defined trait Gauge is invariant in type T.
    [error]     You may wish to investigate a wildcard type such as `_ >: Number`. (SLS 3.2.10)
    [error]     expansion: FlinkMetricManager.this.gauges = FlinkMetricManager.this.gauges.+(fullName.$minus$greater(x))

So this tells us that yes, indeed Number is an Any, but no, Scala is having none
of this. An alternative would be to use [type tags](https://stackoverflow.com/questions/16056645/how-to-pattern-match-on-generic-type-in-scala)
I am not experienced with this method, but I believe it requires the inclusion
of generics inside the function header. Unfortunately this is not possible for
my solution as the function is part of an interface, which I have no control
over.

The only other way around this was to explicitly catch _java.lang.ClassCastException_ and silently dismiss them, which does not seem like a good way to handle this.

### Mapping a list of tuples

Another thing that tripped me up is mapping through a list of tuples:

    {% highlight scala %}
    scala> val x : List[(String, Int)] = List(("Hello", 100), ("World", 1001))
    x: List[(String, Int)] = List((Hello,100), (World,1001))
    {% endhighlight %}

Why is it that the first of the following examples does not work, but the latter
do?

    {% highlight scala %}
    scala> x.map((a, b) => f"$a-$b")
    <console>:15: error: missing parameter type
    Note: The expected type requires a one-argument function accepting a 2-Tuple.
        Consider a pattern matching anonymous function, `{ case (a, b) =>  ... }`
        x.map((a, b) => f"$a-$b")
                ^
    <console>:15: error: missing parameter type
        x.map((a, b) => f"$a-$b")
                    ^

    scala> x.map(a => f"${a._1}-${a._2}")
    res4: List[String] = List(Hello-100, World-1001)

    scala> x.map({ case (a, b) => f"$a-$b" })
    res5: List[String] = List(Hello-100, World-1001)
    {% endhighlight %}

The tuples are immutable strongly typed data structures, so it cannot be that
the type inference is failing here. Then of course a List can contain either a
_Cons_ or a _Nil_. However this should be the same for single elements. My best
guess is that Scala has some syntax sugar to better deal with single value lists
than with a list of tuples. 

### Casting with generics

Another try at generics. Casting a value dynamically:

    class Receiver[T](setter: T=>Unit) {
        def recv() = ???
        def set(input: String) = {
            val x = input.asInstanceOf[T]    
            setter(x)
        }
    }

    object Receiver extends App {
        val myRecv = new Receiver[Int]((x: Int) => println(x)) 
        myRecv.set("123")
    }

Doesn't work. But why? Let's go into detail in the repl:

    scala> val y = "123"
    y: String = 123

    scala> y.asInstanceOf[Int]
    java.lang.ClassCastException

Ok, so this doesn't work. Would a similar method work for the Any type?

    scala> val z: Any = 123
    z: Any = 123

    scala> z.asInstanceOf[Int]
    res2: Int = 123

This works well! So can we go via the Any type?

    scala> y.asInstanceOf[Any]
    res4: Any = 123

    scala> val a : Any = y.asInstanceOf[Any]
    a: Any = 123

    scala> a.asInstanceOf[Int]
    java.lang.ClassCastException: java.lang.String cannot be cast to java.lang.Integer
    at scala.runtime.BoxesRunTime.unboxToInt(BoxesRunTime.java:101)
    ... 28 elided

Nope. Interestingly, the exception refers to the String, but the variable should
have the type Any. So it seems, that the String is kept somewhere as a hidden
type.

Well maybe it is simply impossible to cast from String to Int.

    scala> val y: String = "123"
    y: String = 123

    scala> y.toInt
    res13: Int = 123

Oh come on...

So it seems that scala Strings are either WrappedString or StringOps and both
implement StringLike, which has the following line:

    def toInt: Int         = java.lang.Integer.parseInt(toString)

The implementation of asInstanceOf is likely more complex and I could not find
it directly in Scala's source code.

Others with the same problem: http://www.scala-archive.org/problem-with-asInstanceOf-td1936317.html
