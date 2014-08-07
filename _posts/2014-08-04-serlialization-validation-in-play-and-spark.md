---
layout: post
title:  "Using web tier validation to enable strongly typed, big data pipelines"
date:   2014-08-07 11:50:13
categories: scala validation play spark
---



Do you think of yourself as a web developer?  Or maybe you're more into building backend, distributed systems.  MapReduce much?  As engineers, we often overspecialize and lose out on opportunities to apply creative solutions from other domains to the problem at hand.

Recently I had the opportunity to revisit some work we had done on a front-end API using the Play web framework and apply it to solving the problem of creating type safe data pipelines that are resilliant in the face of invalid records.


Whether your data is big or small, garbage in is garbage out!
-------------------------------------------------------------

MediaMath processes TBs of data every day streaming from our data centers distributed across XXX global locations.  We strive to create strong guarantees for data integrity but it's invetiable that with these many machines and this much data some bad records slip in from time to time.

Often this happens when you try to process user-submitted data such as User-agnet strings or metadata entered via autoamted systesm.  Unfortunately this is the real world and datasets often contain unformatted columns or strings, such as user-agent, that are all too easy to bust up a naive format like TSV.  It's often the case that changing to a more robust data transfer format like Avro or Protocol Buffers is not feasible so you're stuck with TSV.

What are your options?

Functional thinking
-------------------

If you consider any sort of Map/Reduce paradigm (realtime or batch) you often need to translate some kind of wire format `T` into an object of type `D` with a set of fields in order to decide how to partition, join, filter or aggregate the records as they stream through your processor.  In mathematical terms, you need a function `translate: (input: T) => D` where `input` could be a parsed JSON object, a snippet of XML, an array of bytes or an array of stings in the case of tab or comma delimited files.

The problem is that the function above can fail.  Think about the scenario of processing a CSV file line by line.  Each line has columns of different types (strings, integers, floating point numbers).    What if someone puts "#@$?" where you were expecting a number?  Or leaves a required field blank?  In other words, our function is only defined *for some* values of `T` (it's a partial function).  We can model this in Scala a couple of ways, most notably we could throw an exception or return an `Option[D]`.

Play's JSON API
-----------------------------

The Play framework has a very robust API for doing just these sorts of validations against user submitted HTML forms or JSON documents.  Let's take a look at some examples.

First we create our domain model, taking advantage of Scala's strong typing and features such as `Option` to safely process values that may be missing.

```scala
case class Impression(...)
```

Next we create a sort of recipe or formula for mapping a JSON document into our domain object.  Note that this step could fail for any number of reasons such as missing fields, or fields having the wrong type.

The type signature in this case is `T => Success[E, A]`


The real power comes from being able to combine pure functions to process these fields in new, interesting ways in a fully type safe manner.

```scala
Example using map, flatmap, composition, etc...
```

This approach to data validation is very powerful, and we've found multiple ways of using it at MediaMath.

Play DynamoDB
-------------

MediaMath uses a variety of technologies in our analytics stack, including DynamoDB.  DynamoDB is a NoSQL service that makes it easy to prototype and scale data services.  It powers a few internal troubleshooting tools which have front-end/API layers written in Play.

We really liked the JSON API and wanted to see if it could be extended.  The Java API is quite verbose, but essentially API calls return an `Item` which is a sort of `HashMap`.  What if we could have functions from `Item => Success[E, A]` and get data binding like Play for free?


We build a system called Play DynamoDB to do this.

- Easily convert data stored in DynamoDB into case classes
- Supports primitive types, iterables, byte arrays and Joda-Time classes


First, make sure you understand the [JSON combinators API](http://www.playframework.com/documentation/2.2.x/ScalaJsonCombinators).
Then you just need to create an implicit reads for your case class and call 'validate'.  You will get back either
a `DdbSuccess` or a `DdbError` depending on whether or not the validation/parsing succeeded.

```scala
import play.api.libs.functional.syntax._
import play.api.libs.dynamodb._
import org.joda.time.DateTime

/**
 * Example item as returned from DynamoDB API
 */
val item = Map(
  "username"       → new AttributeValue().withS("themodernlife"),
  "favoriteQuotes" → new AttributeValue().withSS("Audentes fortuna iuvat", "Festina lente"),
  "githubUrl"      → new AttributeValue().withS("https://github.com/themodernlife"),
  "commits"        → new AttributeValue().withN("25"),
  "createdAt"      → new AttributeValue().withS("2014-05-19 11:26:00")
)

/**
 * Case class for objects stored in DynamoDB
 */
case class User(username: String, favoriteQuotes: Set[String], githubUrl: Option[String], commits: Int, createdAt: DateTime)

/**
 * Override default date parsing
 */
implicit val jodaDateTimeReads = Reads.dateTimeReads("yyyy-MM-dd HH:mm:ss")

/**
 * Formula for validating a User case class
 */
implicit val userReads = (
  DdbKey("username").read[String] and
  DdbKey("favoriteQuotes").read[Set[String]] and
  DdbKey("githubUrl").read[Option[String]] and
  DdbKey("commits").read[Int] and
  DdbKey("createdAt").read[DateTime]
)(User)

/**
 * Perform the validation and convert to case class
 */
val user = Item.parse(item).validate[User]
```

Let's take this further!
-------------------------

Play's JSON API has been split out into a separate project with a lot of promise.  Currently dev work is happening at jto/validation.  Can we extend it even more?

Recall our mathematical definiton of valdation... it's sort of a serialziation frameowrk.  For example:

How would we look at serialization mathematically?  fn: Array[Bytes] => Option[T]
What about JSON?  JsonObject => Option[T]
What about CSV file?  Array[Byte] => Option[T]
What about a result set?

In the new Validation API you get a slightly different syntax, but it's still cool.

```Scala
Example!
```


Processing S3 Access Logs in Spark
-----------------------------------

We are heavy users of S3 at MediaMath.  We've enabled S3 Access Logging on many of the buckets we use for production datasets.  An S3 access log looks like this:

What if we want to do some data analysis on this, using Spark for instance?

First, let's create our domain object

```scala
case class S3AccessLog(
  bucketOwner: String,
  bucket: String,
  time: DateTime,
  remoteIp: String,
  requester: String,
  requesterId: String,
  operation: String,
  key: String,
  requestUri: String,
  httpStatus: Int,
  errorCode: Option[String],
  bytesSent: Option[Long],
  objectSize: Option[Long],
  totalTime: Long,
  turnAroundTime: Long,
  referrer: String,
  userAgent: String,
  versionId: String
)
```
The S3 access logs need a little processing before we can simply treat them as a "delimited" file.  But once we've done that, we just need to create a `Rule` which maps from an `Array[String]` (aliased as `Delimited`) to our `S3AccessLog` domain object and we're good!

```scala
class S3AccessLogScheme(csvParser: CSVParser = new CSVParser(' ')) extends Scheme[S3AccessLog] {
  private val dateTimePattern = """(.*)\[(\d\d/\w\w\w/\d\d\d\d:\d\d:\d\d:\d\d (\+|-)?\d\d\d\d)\](.*)""".r

  implicit val reader = From[Delimited] { __ ⇒ (
    (__ \ 0).read[String] ~
    (__ \ 1).read[String] ~
    (__ \ 2).read(jodaDateRule("dd/MMM/yyyy:HH:mm:ss Z")) ~
    (__ \ 3).read[String] ~
    (__ \ 4).read[String] ~
    (__ \ 5).read[String] ~
    (__ \ 6).read[String] ~
    (__ \ 7).read[String] ~
    (__ \ 8).read[String] ~
    (__ \ 9).read[Int] ~
    (__ \ 10).read(optionR[String](equalTo("-"))) ~
    (__ \ 11).read(optionR[Long](equalTo("-"))) ~
    (__ \ 12).read(optionR[Long](equalTo("-"))) ~
    (__ \ 13).read[Long] ~
    (__ \ 14).read[Long] ~
    (__ \ 15).read[String] ~
    (__ \ 16).read[String] ~
    (__ \ 17).read[String]
  )(S3AccessLog.apply _)}

  private def fixDate(s: String): String = dateTimePattern.replaceAllIn(s, "$1\"$2\"$4")

  def translate(record: String): VA[S3AccessLog] = {
    val sanitized = fixDate(record)
    reader.validate(csvParser.parseLine(sanitized))
  }
}
```

Let's process

```scala
def main(args: Array[String]) = {
  val sc = new SparkContext("local", "")

  lazy val scheme = new S3AccessLogScheme()

  sc.hadoopConfiguration.setClass("mapred.input.pathFilter.class", classOf[DateFilter], classOf[PathFilter])

  sc.textFile("s3://mm-prod-cloud-trail-logs/S3AccessLogs/mm-prod-raw-logs-bidder/2014-07-16-16-32-16-EC08CB17F0F10717")
    .map(scheme.translate)
    // At this point we have a stream of S3AccessLog objects
    .foreach(println)
}
```

That right there is a fully typed safe data pipeline that can be joined, grouped and aggregated with compile time type checks and is resilient to bad records.

More to come
------------

This post just scratched the surface of what's possible with the strong, combinator-based approach to data translation and validation offered by Play.  I really hope the Validation project catches on and can stand on its own 2 feet (without Play).  As we've shown the applications to Big Data pipelines could really be a huge boon!

If you'd like more info, please check out:

- Play DynamoDB for more info on using DynamoDB from Play
- The PR against Validation to add Delimited (CSV) support

Oh, and if you like working with Big Data on AWS?  Checkout MediaMath for more big data fun!
