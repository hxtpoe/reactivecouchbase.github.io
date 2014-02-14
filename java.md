---
layout: home
title: ReactiveCouchbase Java wrapper
---

The Java wrapper 
==========================

There is a simple Java wrapper if you want to use ReactiveCouchbase from a Java app.

A simple example
-----------------

Here you'll see how to use the basic features of ReactiveCouchbase from a pure Java app.
 
### Prerequisites 

 
We assume that you got a running Couchbase instance. If not, get the latest Couchbase binaries and install it (<a href="http://www.couchbase.com/download">http://www.couchbase.com/download</a>). You can also populate your default bucket with the beer sample provided with Couchbase.
 
### Set up your project dependencies 

 
ReactiveCouchbase is available on a private Maven repository. If you use SBT, you just have to edit build.sbt and add the following

```scala
resolvers += "ReactiveCouchbase Snapshots" at "https://raw.github.com/ReactiveCouchbase/repository/master/snapshots/"
 
libraryDependencies ++= Seq(
  "org.reactivecouchbase" %% "java-wrapper" % "0.2-SNAPSHOT"
)
```

with maven

```xml

<dependency>
  <groupId>org.reactivecouchbase</groupId>
  <artifactId>java-wrapper</artifactId>
  <version>0.2-SNAPSHOT</version>
</dependency>

<repositories>
  <repository>
    <id>ReactiveCouchbase</id>
    <url>https://raw.github.com/ReactiveCouchbase/repository/master/snapshots</url>
  </repository>
</repositories>

```
                 
### Connect to the default bucket 

                 
You can get a connection to the default bucket like this

```java

import net.spy.memcached.ops.OperationStatus;
import org.reactivecouchbase.ReactiveCouchbaseDriver;
import org.reactivecouchbase.common.Functionnal;
import org.reactivecouchbase.json.JsObject;
import org.reactivecouchbase.json.Json;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.reactivecouchbase.json.Syntax.*;

public class Sample {

  public static void main(String... args) {
    final ReactiveCouchbaseDriver driver = 
          ReactiveCouchbaseDriver.apply();
    final CouchbaseBucket bucket = 
          new CouchbaseBucket(driver.bucket("default"));
    final ExecutorService ec = 
          Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
  
    ...

    driver.shutdown();
  }
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
                     
### Insert some documents

Using JSON blobs

```java

import net.spy.memcached.ops.OperationStatus;
import org.reactivecouchbase.ReactiveCouchbaseDriver;
import org.reactivecouchbase.common.Functionnal;
import org.reactivecouchbase.json.JsObject;
import org.reactivecouchbase.json.Json;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.reactivecouchbase.json.Syntax.*;

public class Sample {

  public static void main(String... args) {
    final ReactiveCouchbaseDriver driver = 
        ReactiveCouchbaseDriver.apply();
    final CouchbaseBucket bucket = 
        new CouchbaseBucket(driver.bucket("default"));
    final ExecutorService ec = 
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    JsObject document = Json.obj(
      $("name", "John"),
      $("surname", "Doe"),
      $("age", 42),
      $("address", Json.obj(
        $("number", 42),
        $("street", "Baker Street"),
        $("city", "London")
      ))
    );
    bucket.set("john-doe", document, FormatHelper.JS_OBJECT_FORMAT)
          .onSuccess(new Functionnal.Action<OperationStatus>() {
      @Override
      public void call(OperationStatus operationStatus) {
        System.out.println("Operation status : " + operationStatus.getMessage());
        driver.shutdown();
      }
    }, ec);
  }
}

```

Using typesafe objects

```java

import com.google.common.base.Function;
import net.spy.memcached.ops.OperationStatus;
import org.reactivecouchbase.ReactiveCouchbaseDriver;
import org.reactivecouchbase.common.Functionnal;
import org.reactivecouchbase.json.Format;
import org.reactivecouchbase.json.JsResult;
import org.reactivecouchbase.json.JsValue;
import org.reactivecouchbase.json.Json;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.reactivecouchbase.json.JsResult.combine;
import static org.reactivecouchbase.json.Syntax.$;

public class Sample {

  public static class Address {
    public final String number;
    public final String street;
    public final String city;
    public Address(String number, String street, String city) {
      this.number = number;
      this.street = street;
      this.city = city;
    }
    public static final Format<Address> FORMAT =  new Format<Address>() {
      @Override
      public JsResult<Address> read(JsValue value) {
        return combine(
          value.field("number").read(String.class),
          value.field("street").read(String.class),
          value.field("city").read(String.class)
        ).map(new Function<Functionnal.T3<String, String, String>, Address>() {
          @Override
          public Address apply(Functionnal.T3<String, String, String> input) {
            return new Address(input._1, input._2, input._3);
          }
        });
      }
      @Override
      public JsValue write(Address value) {
        return Json.obj(
          $("number", value.number),
          $("street", value.street),
          $("city", value.city)
        );
      }
    };
  }

  public static class Person {
    public final String name;
    public final String surname;
    public final Integer age;
    public final Address address;
    public Person(String name, String surname, Integer age, Address address) {
      this.name = name;
      this.surname = surname;
      this.age = age;
      this.address = address;
    }
    public static final Format<Person> FORMAT = new Format<Person>() {
      @Override
      public JsResult<Person> read(JsValue value) {
        return combine(
          value.field("name").read(String.class),
          value.field("surname").read(String.class),
          value.field("age").read(Integer.class),
          value.field("address").read(Address.FORMAT)
        ).map(new Function<Functionnal.T4<String, String, Integer, Address>, Person>() {
            @Override
            public Person apply(Functionnal.T4<String, String, Integer, Address> input) {
                return new Person(input._1, input._2, input._3, input._4);
            }
        });
      }
      @Override
      public JsValue write(Person value) {
        return Json.obj(
          $("name", value.name),
          $("surname", value.surname),
          $("age", value.age),
          $("address", Address.FORMAT.write(value.address))
        );
      }
    };
  }

  public static void main(String... args) {
    final ReactiveCouchbaseDriver driver = 
        ReactiveCouchbaseDriver.apply();
    final CouchbaseBucket bucket = 
        new CouchbaseBucket(driver.bucket("default"));
    final ExecutorService ec = 
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());

    Person document = new Person("John", "Doe", 42, 
            new Address("221b", "Baker Street", "London"));
    // persist the Person instance with the key 'john-doe', using implicit 'personFmt' for serialization
    bucket.set("john-doe", document, Person.FORMAT)
          .onSuccess(new Functionnal.Action<OperationStatus>() {
      @Override
      public void call(OperationStatus operationStatus) {
        System.out.println("Operation status : " + operationStatus.getMessage());
        driver.shutdown();
      }
    }, ec);
  }
}

```
### Fetch a document

```java


import com.google.common.base.Function;
import org.reactivecouchbase.ReactiveCouchbaseDriver;
import org.reactivecouchbase.common.Functionnal;
import org.reactivecouchbase.json.Format;
import org.reactivecouchbase.json.JsResult;
import org.reactivecouchbase.json.JsValue;
import org.reactivecouchbase.json.Json;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

import static org.reactivecouchbase.json.JsResult.combine;
import static org.reactivecouchbase.json.Syntax.$;

public class Sample {

  public static class Address {
    public final String number;
    public final String street;
    public final String city;
    public Address(String number, String street, String city) {
      this.number = number;
      this.street = street;
      this.city = city;
    }
    public static final Format<Address> FORMAT =  new Format<Address>() {
      @Override
      public JsResult<Address> read(JsValue value) {
        return combine(
          value.field("number").read(String.class),
          value.field("street").read(String.class),
          value.field("city").read(String.class)
        ).map(new Function<Functionnal.T3<String, String, String>, Address>() {
          @Override
          public Address apply(Functionnal.T3<String, String, String> input) {
            return new Address(input._1, input._2, input._3);
          }
        });
      }
      @Override
      public JsValue write(Address value) {
        return Json.obj(
          $("number", value.number),
          $("street", value.street),
          $("city", value.city)
        );
      }
    };
    @Override
    public String toString() {
      return "Address{" +
        "number='" + number + '\'' +
        ", street='" + street + '\'' +
        ", city='" + city + '\'' +
        '}';
    }
  }

  public static class Person {
    public final String name;
    public final String surname;
    public final Integer age;
    public final Address address;
    public Person(String name, String surname, Integer age, Address address) {
        this.name = name;
        this.surname = surname;
        this.age = age;
        this.address = address;
    }
    public static final Format<Person> FORMAT = new Format<Person>() {
        @Override
        public JsResult<Person> read(JsValue value) {
          return combine(
            value.field("name").read(String.class),
            value.field("surname").read(String.class),
            value.field("age").read(Integer.class),
            value.field("address").read(Address.FORMAT)
          ).map(new Function<Functionnal.T4<String, String, Integer, Address>, Person>() {
            @Override
            public Person apply(Functionnal.T4<String, String, Integer, Address> input) {
              return new Person(input._1, input._2, input._3, input._4);
            }
          });
        }
        @Override
        public JsValue write(Person value) {
          return Json.obj(
            $("name", value.name),
            $("surname", value.surname),
            $("age", value.age),
            $("address", Address.FORMAT.write(value.address))
          );
        }
    };
    @Override
    public String toString() {
      return "Person{" +
        "name='" + name + '\'' +
        ", surname='" + surname + '\'' +
        ", age=" + age +
        ", address=" + address +
        '}';
    }
  }

  public static void main(String... args) {
    final ReactiveCouchbaseDriver driver = 
        ReactiveCouchbaseDriver.apply();
    final CouchbaseBucket bucket = 
        new CouchbaseBucket(driver.bucket("default"));
    final ExecutorService ec = 
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    // get the Person instance with the key 'john-doe', 
    // using implicit 'personFmt' for deserialization
    bucket.get("john-doe", Person.FORMAT)
          .map(new Function<Functionnal.Option<Person>, Functionnal.Unit>() {
      @Override
      public Functionnal.Unit apply(Functionnal.Option<Person> input) {
        System.out.println(
          input.map(new Function<Person, String>() {
            @Override
            public String apply(Person input) {
              return "Found John : " + input;
            }
          }).getOrElse("Cannot find object with key 'john-doe'")
        );
        driver.shutdown();
        return Functionnal.Unit.unit();
      }
    }, ec);
  }
}

``` 

### Run a simple query 
                 
Assuming your default bucket is filled with the beer sample data, let's print all the beers 

```java


import com.couchbase.client.protocol.views.Query;
import com.google.common.base.Function;
import org.reactivecouchbase.ReactiveCouchbaseDriver;
import org.reactivecouchbase.client.Row;
import org.reactivecouchbase.common.Functionnal;
import org.reactivecouchbase.japi.concurrent.Future;
import org.reactivecouchbase.json.JsObject;
import org.reactivecouchbase.json.Json;

import java.util.List;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class Sample {

  public static void main(String... args) {
    final ReactiveCouchbaseDriver driver = 
        ReactiveCouchbaseDriver.apply();
    final CouchbaseBucket bucket = 
        new CouchbaseBucket(driver.bucket("default"));
    final ExecutorService ec = 
        Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
    // search the whole list of docs from the view 'brewery_beers'
    Future<List<Row<JsObject>>> futureList =
        bucket.search("beers", "brewery_beers", 
          new Query().setIncludeDocs(true), FormatHelper.JS_OBJECT_FORMAT);
    // when the query is done, run over all the docs and print them
    futureList.map(new Function<List<Row<JsObject>>, Functionnal.Unit>() {
      @Override
      public Functionnal.Unit apply(List<Row<JsObject>> input) {
        for (Row<JsObject> row : input) {
          System.out.println(Json.prettyPrint(row.docOpt().get()));
        }
        driver.shutdown();
        return Functionnal.Unit.unit();
      }
    }, ec);
  }
}

```
         
