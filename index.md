---
layout: home
title: ReactiveCouchbase
---

About Reactive Couchbase  
==========================

ReactiveCouchbase is a scala driver that provides non-blocking and asynchronous I/O operations on top of <a href="http://www.couchbase.com">Couchbase</a>. ReactiveCouchbase is designed to avoid blocking on each database operations. Every operation returns immediately, using the elegant <a href="http://www.scala-lang.org/api/current/#scala.concurrent.Future">Scala Future API</a> to resume execution when it's over. With this driver, accessing the database is not an issue for performance anymore. ReactiveCouchbase is also highly focused on streaming data in and out from your Couchbase servers using the very nice <a href="http://www.playframework.com/documentation/2.2.x/Iteratees">Play framework Iteratee API</a>.

A simple example
=======================
 
Here you'll see how to use the basic features of ReactiveCouchbase.
 
### Prerequisites 

 
We assume that you got a running Couchbase instance. If not, get the latest Couchbase binaries and install it (<a href="http://www.couchbase.com/download">http://www.couchbase.com/download</a>). You can also populate your default bucket with the beer sample provided with Couchbase.
 
### Set up your project dependencies 

 
ReactiveMongo is available on a private Maven repository. If you use SBT, you just have to edit build.sbt and add the following

```scala
resolvers += "ReactiveCouchbase Snapshots" at "https://raw.github.com/ReactiveCouchbase/repository/master/snapshots/"
 
libraryDependencies ++= Seq(
  "org.reactivecouchbase" %% "reactivecouchbase-core" % "0.1-SNAPSHOT"
)
```
                 
### Connect to the default bucket 

                 
You can get a connection to the default bucket like this

```scala
// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
 
object Application extends App {
 
  // get a driver instance driver    
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")
 
  // do something here with the default bucket  

  // shutdown the driver  
  driver.shutdown()
}
```

You'll need an application.conf file in the project resources 

```
couchbase {
  buckets = [{
    host="127.0.0.1"
    port="8091"
    base="pools"
    bucket="default"
    user=""
    pass=""
    timeout="0"
  }]
}
```
                 
                 
You can also provide your own ActorSystem, Logging system and configuration programmatically to the driver when you create it.
    
### Insert some documents

Using JSON blobs

```scala
// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
// import the implicit JsObject reader and writer 
import org.reactivecouchbase.CouchbaseRWImplicits.documentAsJsObjectReader
import org.reactivecouchbase.CouchbaseRWImplicits.jsObjectToDocumentWriter
import scala.concurrent.Future
import play.api.libs.json._

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // creates a JSON document
  val document = Json.obj(
    "name" -> "John",
    "surname" -> "Doe",
    "age" -> 42,
    "address" -> Json.obj(
      "number" -> "221b"
      "street" -> "Baker Street"
      "city" -> "London"
    )
  )

  bucket.set("john-doe", document).onSuccess {
    case status => println(s"Operation status : ${status.getMessage}")
  }

  // shutdown the driver  
  driver.shutdown()
}
```

Using typesafe objects

```scala

// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
import scala.concurrent.Future
import play.api.libs.json._

class Address(number: String, street: String, city: String)
class Person(name: String, surname: String, age: Int, address: Address)

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // provide implicit Json formatters
  implicit val addressFmt = Json.format[Address]
  implicit val personFmt = Json.format[Person]

  // creates a JSON document
  val document = Person("John", "Doe", 42, Address("221b", "Baker Street", "London"))

  bucket.set("john-doe", document).onSuccess {
    case status => println(s"Operation status : ${status.getMessage}")
  }

  // shutdown the driver  
  driver.shutdown()
}

```
### Fetch a document

```scala

// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
import scala.concurrent.Future
import play.api.libs.json._

class Address(number: String, street: String, city: String)
class Person(name: String, surname: String, age: Int, address: Address)

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // provide implicit Json formatters
  implicit val addressFmt = Json.format[Address]
  implicit val personFmt = Json.format[Person]

  bucket.get("john-doe").map { opt =>
    println(opt.map(person => s"Found John : ${person}").getOrElse("Cannot find object with key 'john-doe'"))
  }

  // shutdown the driver  
  driver.shutdown()
}

### Run a simple query 
                 
Assuming your default bucket is filled with the beer sample data, let's print all the beers 

```scala
// first import the implicit execution context  
import scala.concurrent.ExecutionContext.Implicits.global
import org.reactivecouchbase.ReactiveCouchbaseDriver
// import the implicit JsObject reader  
import org.reactivecouchbase.CouchbaseRWImplicits.documentAsJsObjectReader
import com.couchbase.client.protocol.views.Query
import play.api.libs.iteratee.Iteratee
import scala.concurrent.Future
import play.api.libs.json._

object Application extends App {

  // get a driver instance driver  
  val driver = ReactiveCouchbaseDriver()
  // get the default bucket  
  val bucket = driver.bucket("default")

  // search all docs from the view 'brewery_beers'   
  // then enumerate it and print each document on the fly  
  // streaming style
  bucket.searchValues[JsObject]("beers", "brewery_beers")
      (new Query().setIncludeDocs(true))
        .enumerated.apply(Iteratee.foreach { doc =>
    println(s"One more beer for the road  : ${Json.prettyPrint(doc)}")
  })

  // search the whole list of docs from the view 'brewery_beers'   
  val futureList: Future[List[JsObject]] = 
    bucket.searchValues[JsObject]("beers", "brewery_beers")
      (new Query().setIncludeDocs(true)).toList
  // when the query is done, run over all the docs and print them  
  futureList.map { list =>
    list.foreach { doc =>
      println(s"One more beer for the road : ${Json.prettyPrint(doc)}")
    }
  }
  // shutdown the driver  
  driver.shutdown()
}
```
                 

Samples
======================

You can find some application samples on <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play/tree/master/samples">ReactiveCouchbase's github</a>

You can also find starter kits to quickly bootstrap projects

* <a href="https://github.com/ReactiveCouchbase/reactivecouchbase-starter-kit">Pure Scala starter kit</a>
* <a href="https://github.com/ReactiveCouchbase/play-java-starter-kit">Play Java starter kit</a>
* <a href="https://github.com/ReactiveCouchbase/play-scala-starter-kit">Play Scala starter kit</a>
* <a href="https://github.com/ReactiveCouchbase/play-scrud-starter-kit">Pure Scala CRUD starter kit</a>

Components
======================

ReactiveCouchbase is composed of 3 components

### ReactiveCouchbase Core

The core of ReactiveCouchbase. Depends on Play Iteratees library, Play JSON library and Akka

### ReactiveCouchbase Event Store

A simple Event Store for eventsourcing application using Couchbase as event log

### ReactiveCouchbase Play 2 plugin

You can absolutely use ReactiveCouchbase from a Play 2 application. For more information about it, go to the <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play">ReactiveCouchbase-play's github project</a>

Community
======================

* ReactiveCouchbase on <a href="https://github.com/ReactiveCouchbase">GitHub</a>
* Tickets are on <a href="https://github.com/ReactiveCouchbase">GitHub</a>. Feel free to report any bug you find. Don’t hesitate to make pull requests.
* ReactiveCouchbase on <a>Google Groups</a>

                