---
layout: post
title:  "Using web tier validation to enable strongly typed, big data pipelines"
date:   2014-08-07 11:50:13
categories: scala validation play spark
---

Do you think of yourself as a web developer?  Or maybe you're more into building backend, low-latency systems.  Map/reduce much?  As engineers, we often overspecialize and lose out on opportunities to apply creative solutions from other domains to the problem at hand.

Recently I had the opportunity to revisit some work we had done to a web-tier API using the [Play web framework](http://www.playframework.com/) and apply it to solving the problem of creating type safe data pipelines that are resilient in the face of invalid records.

The secret sauce?  [Play's combinator-based approach to data validation](https://www.playframework.com/documentation/2.3.x/ScalaJson).


Whether your data is big or small, garbage in is garbage out
-------------------------------------------------------------

MediaMath processes TBs of online user behavior and advertising data every day. It's inevitable that with hundreds of machines spread across multiple datacenters, legacy systems and partner provided APIs we receive bad data or invalid records from time to time. Systems built around file formats like CSV or TSV are especially susceptible to encoding errors that can cause headaches for downstream processing systems.

So what are your options?

Quite often the first step for most data processing pipelines (be they single node scripts or massive Hadoop jobs) is translating some kind of encoded wire format `T` into a record of type `D` for partitioning, joining, filtering or aggregating.  In mathematical terms, you need a function `translate: (input: T) => D` where `input` could be a parsed JSON object, a snippet of XML, an array of bytes or an array of stings in the case of tab or comma delimited files.

But what if the translation fails?

Think about the scenario of processing a CSV file line by line.  Each line has columns of different types (strings, integers, floating point numbers).    What if someone puts "#@$?" where you were expecting a number?  Or leaves a required field blank?  In other words, our function is only defined *for some* values of `T` (it's a partial function).  At MediaMath we use Scala, so the natural choice would be to model this by throwing an exception or returning an `Option[D]`.

That said, a richer validation API would really open up the possibility of sharing parsing logic between projects, easily swapping out the underlying wire format, using richer types like Joda Time's `DateTime` instead of `String`s or `Int`s and much more...

That's where [Play](http://www.playframework.com/) comes in.


Validation with Play's combinator-based API
-------------------------------------------

The Play framework has a very robust API for doing just these sorts of validations against user submitted HTML forms or JSON documents, in essence providing a function equivalent to something like `validate[D]: (input: Json) => JsResult[D]` where `JsResult` is a more robust version of `Option` that can remember failures for each path or index of the JSON document.  

Let's take a look at an example service for creating new online advertising campaigns.  First we define our domain model, taking advantage of Scala's strong typing and features such as `Option` to safely process values that may be missing.

```scala
case class Campaign(name: String, startDate: DateTime, endDate: DateTime, budget: Double, meritPixelId: Option[Long])
```

Next we create a sort of recipe or formula for mapping a JSON document into our domain model called a `Reads`.  Note that the parsing could fail for any number of reasons such as users uploading a document with missing fields, or fields having the wrong data type.  We use combinator syntax to `and` together each field's `Reads` and apply the resulting objects as the arguments to the `Campaign` constructor.

```scala
implicit val reads = (
  (__ \ "name").read[String] and
  (__ \ "start_date").read[DateTime] and
  (__ \ "end_date").read[DateTime] and
  (__ \ "budget").read[Double] and
  (__ \ "merit_pixel_id").read[Option[Long]]
)(Campaign)
```

Let's use our new validator to parse some JSON:

```scala
val json = Json.parse("""
{
  "name" : "Summer 2014 Blockblusters",
  "start_date" : "2014-08-01",
  "end_date" : "2014-08-09",
  "budget" : 10000.00,
  "merit_pixel_id" : 100213
}
""")

json.validate[Campaign] match {
  case JsSuccess(campaign, _) ⇒ println(s"Created campaign: ${campaign}")
  case JsError(errors)        ⇒ println(s"Invalid JSON document: ${errors}")
}
// prints Campaign(Summer 2014 Blockblusters,2014-08-01T00:00:00.000-04:00,2014-08-09T00:00:00.000-04:00,10000.0,Some(100213))
```

This approach to data validation is fully type safe and very powerful. We've found multiple ways of using it at MediaMath.

Extending validation to work with AWS DynamoDB
----------------------------------------------

MediaMath uses a variety of technologies in our analytics stack, including [AWS DynamoDB](http://aws.amazon.com/dynamodb/).  DynamoDB is a distributed, fault-tolerant key value store as a service that makes it easy to store/query massive datasets.  We use it power a few internal troubleshooting tools which have front-end/API layers written in Play.

That said, a downside of using the [AWS Java SDK](http://aws.amazon.com/sdk-for-java/) from Scala is that it feels quite verbose and unidiomatic.  We really liked the succinct JSON API from Play and wanted to see if it could be extended to create a data binding layer for DynamoDB's `Item` instead of JSON docs.  Turns out this was quite easy to do and the results are now open sourced as [Play DynamoDB](https://github.com/MediaMath/play-dynamodb).

Working with Play DynamoDB is very similar to working with the Play JSON API.

```scala
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

As you can see, the code is almost the same:

- Create your domain object of type `D`
- Create your blueprint for parsing it from some type `T`, in this case a DynamoDB `Item`
- Use Play's functional/combinator constructs to map from `Item => DdbResult[D]`


Let's take this further!
----------------------------------------

OK, how does that relate to processing big data pipelines?

There has been a lot of discussion around splitting out Play's JSON API into its own project, as can be seen [in this pull request](https://github.com/playframework/playframework/pull/1904).  It makes a lot of sense because it nicely generalizes the issue of translating data to and from a potentially unsafe wire format in a fully type safe way.  

Development work on the new [Validation API](http://jto.github.io/articles/play_new_validation_ap) happens in GitHub at https://github.com/jto/validation.  It already unifies parsing of JSON and HTML forms and recently MediaMath [submitted a patch](https://github.com/jto/validation/pull/14) to extend it to work with CSV/TSV delimited files like so:

```Scala
case class Contact(name: String, email: String, birthday: Option[LocalDate])

val contactReads = From[Delimited] { __ ⇒ (
  (__ \ 0).read[String] and
  (__ \ 1).read(email) and
  (__ \ 2).read(optionR[LocalDate](Rules.equalTo("N/A")))
)(Contact)}

val csv1 = "John Doe,joe@example.com,1981-07-24"
val csv2 = "Jane Doe,jane@example.com,N/A"

contactReads.validate(csv1.split(",")) mustEqual Success(Contact("John Doe", "joe@example.com", Some(new LocalDate(1981, 7, 24))))
contactReads.validate(csv2.split(",")) mustEqual Success(Contact("Jane Doe", "jane@example.com", None))
```

With the new API you create and combine `Rules` that can bind and validate records from TSV files. Add [Apache Spark](https://spark.apache.org/) to the mix and you get a very compelling development environment for doing fast, reliable and type safe data processing over enormous data sets (which are often stored as lines of JSON or CSV/TSV).


Processing S3 Access Logs in Spark
----------------------------------

We are heavy users of S3 at MediaMath and have enabled [S3 Access Logging](http://docs.aws.amazon.com/AmazonS3/latest/dev/ServerLogs.html) on many of the buckets we use for production datasets.  An S3 Access Log looks like this:

```
2a520ba2eac7794c5663c17db741f376756214b1bbb423214f979d1ba95bac73 bidder-data-bucket [16/Jul/2014:15:47:42 +0000] 74.121.142.208 arn:aws:iam::000000000000:user/test@mediamath.com XXXXXXXXXXXXXXXX REST.PUT.OBJECT 2014-07-16-15/bd.ewr-bidder-x36.1405524542.log.lzo "PUT /2014-07-16-15/bd.ewr-bidder-x36.1405524542.log.lzo HTTP/1.1" 200 - - 18091170 931 66 "-" "aws-sdk-java/1.7.5 Linux/3.2.0-4-amd64 OpenJDK_64-Bit_Server_VM/20.0-b12/1.6.0_27 com.amazonaws.services.s3.transfer.TransferManager/1.7.5" -
```

What if we want to do some data analysis on these access logs, using Spark?

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

The S3 access logs need a little processing before we can simply treat them as a "space-delimited" file.  For example, empty values are represented by the string `-`. Fortunately we can account for all that by chaining multiple `Rule`s together to create a new `Rule` which maps from an `Array[String]` (aliased as `Delimited`) to our `S3AccessLog` domain object.

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

Now we're ready to start crunching these logs with Spark:

```scala
def main(args: Array[String]) = {
  val sc = new SparkContext("local", "")

  lazy val scheme = new S3AccessLogScheme()

  sc.textFile("s3://mm-prod-cloud-trail-logs/S3AccessLogs/mm-prod-raw-logs-bidder/2014-07-16-16-32-16-EC08CB17F0F10717")
    .map(scheme.translate) // Type of the stream at this point is Validation[Seq[ValidationError], S3AccessLog]
    .foreach(println)
}
```

The data pipeline above is fully typed, resilient to bad records and can be joined, grouped and aggregated with compile time type checks and IDE-based code completion.  Much nicer than hardcoding column names all throughout your job!


More to come
------------

This post just scratched the surface of what's possible with the strong, combinator-based approach to data translation and validation offered by the new Play Validation API.  I really hope the project catches on and can stand on its own 2 feet (without from Play).  In the future we'd like to merge our [Play DynamoDB](https://github.com/MediaMath/play-dynamodb) library into it as well.  As we've shown, the enhanced type safety and reusable parsing logic can be used in many ways outside of a traditional web app.


If you'd like more info, here are some links:

- [An article announcing the new Play validation API](http://jto.github.io/articles/play_new_validation_api)
- [Play DynamoDB](https://github.com/MediaMath/play-dynamodb)
- [The new Play validation API project](https://github.com/jto/validation)

Oh, and if you like working with Big Data and AWS?  Checkout the [MediaMath careers page](http://www.mediamath.com/about/careers/)!
