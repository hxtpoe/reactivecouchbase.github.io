---
layout: home
title: ReactiveCouchbase
---

About Reactive Couchbase  
==========================

more soon ...

Step By Step Example
=======================
 
Let's show a simple use case: print the documents of a bucket.
 
### Prerequisites 

 
We assume that you got a running Couchbase instance. If not, get the latest Couchbase binaries and install it (<a href="http://www.couchbase.com/download">http://www.couchbase.com/download</a>)
 
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
  bucket.searchValues[JsObject]("beers", "brewery_beers")
      (new Query().setIncludeDocs(true))
        .enumerated.apply(Iteratee.foreach { doc =>
    println(s"found beer : ${Json.prettyPrint(doc)}")
  })

  // search the whole list of docs from the view 'brewery_beers'   
  val futureList: Future[List[JsObject]] = 
    bucket.searchValues[JsObject]("beers", "brewery_beers")
      (new Query().setIncludeDocs(true)).toList
  // when the query is done, run over all the docs and print them  
  futureList.map { list =>
    list.foreach { doc =>
      println(s"found beer : ${Json.prettyPrint(doc)}")
    }
  }
  // shutdown the driver  
  driver.shutdown()
}
```
                 

Samples
======================

You can find some application samples on <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play/tree/master/samples">ReactiveCouchbase's github</a>

Play 2 plugin
======================

You can absolutely use ReactiveCouchbase from a Play 2 application. For more information about it, go to the <a href="https://github.com/ReactiveCouchbase/ReactiveCouchbase-play">ReactiveCouchbase-play's github project</a>
                