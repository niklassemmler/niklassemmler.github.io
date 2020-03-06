---
layout: post
title: Json in Scala
date: 2018-01-17
categories: json scala
---

For a client-server application I needed to format to exchange messages. As the
format was really simple (type, key, value), I chose to use JSON. How hard can
it be, right?

My first approach was to try to map from case classes to JSON objects and back:

    {% highlight scala %}
    trait Metric
    case class Counter(key : String, value : Long) extends Metric
    case class Meter(key : String, value : Long) extends Metric
    {% endhighlight %}

The goal was to get functionality similar to the following:
    
    {% highlight scala %}
    val c = Counter("asd", 100)
    val json = Json.toJson(c)
    Json.parse(json) match {
        case c : Counter=> ...
        case m : Meter  => ...
    }
    {% endhighlight %}

This would make it easy for me to pass items across the network. I hoped that I
could find a framework, that would transform this type of class into something
similar to:

    {"___type":class_counter, "key":"asd", "value":100}

So I happily duckduckgo'd for a simple copy & paste code sample. Lucky enough
another blog poster has taken the effort to give samples for different JSON
libraries:

https://manuel.bernhardt.io/2015/11/06/a-quick-tour-of-json-libraries-in-scala/

Unfortunately most examples where going more in the direction of:

    {% highlight scala %}
    val c = Counter("asd", 100)
    val json = Json.toJson[Counter](c)
    Json.fromJson[Counter](json)
    {% endhighlight %}

This defeated the purpose of making the case classes easy to use in the rest of
my library.

First I tried the playframework. To me, the easiest way to play aroudn with
scala libraries is to create a custom build sbt and then start the console via
"sbt console". An example build.sbt for the JSON part of the Playframework:

    name := "Example"

    version := "1.0"

    scalaVersion := "2.12.4" // change this to your version

    libraryDependencies += "com.typesafe.play" % "play-json_2.12" % "2.6.8"

Following the logic defined [here](https://www.playframework.com/documentation/2.3.x/ScalaJsonCombinators), I separately defined implicit readers and writers.

    {% highlight scala %}
    import play.api.libs.json._
    import play.api.libs.json.Reads._ 
    import play.api.libs.functional.syntax._

    implicit val countWrites = new Writes[Counter] {
        def writes(c: Counter) = Json.obj(
            "key" -> c.key,
            "value" -> c.value
        )
    }
    implicit val meterWrites = new Writes[Meter] {
        def writes(m: Meter) = Json.obj(
            "key" -> m.key,
            "value" -> m.value
        )
    }

    implicit val counterReads: Reads[Counter] = (
        (JsPath \ "key").read[String] and
        (JsPath \ "value").read[Long]
    )(Counter.apply _)
    implicit val meterReads: Reads[Meter] = (
        (JsPath \ "key").read[String] and
        (JsPath \ "value").read[Long]
    )(Meter.apply _)

    val c = Counter("asd", 100)
    val json = Json.toJson(c) // {"key":"asd","value":100}
    Json.fromJson(json)
    {% endhighlight %}

This would allow me to avoid the type when creating the json string, but I would
not be able to read the json object, due to the redundant readers:

    error: ambiguous implicit values:
    both value counterReads of type => play.api.libs.json.Reads[Counter]
    and value meterReads of type => play.api.libs.json.Reads[Meter]
    match expected type play.api.libs.json.Reads[T]
                Json.fromJson(json)

This is in itself did not surprise me much as the json string itself does not
hold any additional information. A simple solution would be to add a constant to
the json object myself and for the write this actually works.

    {% highlight scala %}
    implicit val countWrites = new Writes[Counter] {
        def writes(c: Counter) = Json.obj(
            "types" -> 0,
            "key" -> c.key,
            "value" -> c.value
        )
    }
    implicit val meterWrites = new Writes[Meter] {
        def writes(m: Meter) = Json.obj(
            "types" -> 1,
            "key" -> m.key,
            "value" -> m.value
        )
    }
    implicit val counterReads = new Reads[Counter] {
        def reads(js: JsValue): JsResult[Counter] = {
            val typ = (js \ "type").as[Int]
            if (typ != 0) JsError("Invalid type") else {
                val key = (js \ "key").as[String]
                val value = (js \ "value").as[Long]
                JsSuccess(Counter(key, value))
            }
        }
    }
    implicit val meterReads = new Reads[Meter] {
        def reads(js: JsValue): JsResult[Meter] = {
            val typ = (js \ "type").as[Int]
            if (typ != 1) JsError("Invalid type") else {
                val key = (js \ "key").as[String]
                val value = (js \ "value").as[Long]
                JsSuccess(Meter(key, value))
            }
        }
    }
    {% endhighlight %}

Yet again for a read we arrive at an error:

     Json.fromJson(json)
    error: ambiguous implicit values:
    both value counterReads of type => play.api.libs.json.Reads[Counter]
    and value meterReads of type => play.api.libs.json.Reads[Meter]
    match expected type play.api.libs.json.Reads[T]
            Json.fromJson(json) 

As a last try, I try to define a common reader for the Metric trait. To this end
I transform the trait into a regular class:

    {% highlight scala %}
    class Metric(key : String, value : Long)
    case class Counter(key : String, value : Long) extends Metric(key, value)
    case class Meter(key : String, value : Long) extends Metric(key, value)

    implicit val metricReads = new Reads[Metric] {
        def reads(js: JsValue): JsResult[Metric] = {
            val typ = (js \ "type").as[Int]
            val key = (js \ "key").as[String]
            val value = (js \ "value").as[Long]
            if (typ == 0)  {
                JsSuccess(Counter(key, value))
            } else if (typ == 0) {
                JsSuccess(Meter(key, value))
            } else JsError("Invalid input")
        }
    }
    val c = Counter("asd", 100)
    val json = Json.toJson(c) // {"key":"asd","value":100}
    Json.fromJson[Metric](json)
    {% endhighlight %}

This does not seem to work. Apparently the read implementation is missing
something.

    play.api.libs.json.JsResultException: JsResultException(errors:List((,List(JsonValidationError(List('type' is undefined on object: {"types":0,"key":"asd","value":100}),WrappedArray())))))
        at play.api.libs.json.JsReadable.$anonfun$as$2(JsReadable.scala:25)
        at play.api.libs.json.JsError.fold(JsResult.scala:56)
        at play.api.libs.json.JsReadable.as(JsReadable.scala:24)
        at play.api.libs.json.JsReadable.as$(JsReadable.scala:23)
        at play.api.libs.json.JsUndefined.as(JsLookup.scala:181)
        at $anon$1.reads(<console>:29)
        at play.api.libs.json.Json$.fromJson(Json.scala:193)
        ... 36 elided

So far my attempt. With more careful reading of the earlier blog post, I
realized that what I was looking for is the automated parsing of case
hierarchies and the author notes himself, that he is using **sphere-json** for this
process. Ah, well a bit of more careful reading would have helped.

The final answer is more concise than anything written before. (It does,
however, not work in the scala console.)

    {% highlight scala %}
    import cats.data.Validated.{Invalid, Valid}
    import io.sphere.json.generic._
    import io.sphere.json._

    sealed trait Metric
    case class Counter(key : String, value : Long) extends Metric
    case class Meter(key : String, value : Long) extends Metric

    object Main {

        implicit val allYourJson = deriveJSON[Metric]
        def main(args: Array[String]): Unit = {
            val c = Counter("asd", 100)
            val json = toJSON[Metric](c)
            fromJSON[Metric](json) match {
                case Valid(x) => println(x)
                case Invalid(_) => println("Could not parse JSON")
            }
        }
    }
    {% endhighlight %}
