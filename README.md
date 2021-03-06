[![Build Status](https://travis-ci.org/1123/StackExchangeImporter.png)](https://travis-ci.org/1123/bitemporaldb)

Bitemporal
----------

This is a bitemporal, object oriented bitemporal database layer on top of 
Mongodb written in Scala. Also an in-memory bitemporal database based on Java 
hashmaps is included that can be used without a database. 

MongoDB has been chosen as the first
persistent implementation, since it allows for document oriented storage and
therefore helps to simplify the data model of your application, reducing the
number of joins, which must be thought out carefully in a bitemporal context.

For an explanation of the concept of bitemporality see http://en.wikipedia.org/wiki/Temporal_database

The concept of bitemporal data management is important for many industries
ranging from banking over insurances to the public sector. Ordinary databases
per se only store our current knowledge about the current state of reality.
Bitemporal databases instead store the entire history of our knowledge about
the different states of reality at different points in time. Therefore bitemporal data
management allows to answer WHY-questions about the past, allowing to simulate
queries for any points in the past.

Bitemporal databases distinguish two time dimensions: the valid time and the
transaction time.  Valid time concerns the state of objects in reality, which
changes over time.  Transaction time concerns our assumptions or knowledge
about the objects in reality as they are saved in our bitemporal database.

Requirements
------------

* scala 
* sbt
* java 1.6 or higher

Usage
-----

Using this library is most easily understood by looking at the test cases in the example directory. Here is an example that stores information about a student and updates that information. Subsequently we would like to query the database state as it was 10 seconds earlier. We get the original value. This example only demonstrates the use of transaction time. There are other examples in src/test/scala that also demonstrate the use of valid time.

```scala
/*
 * Store two temporal versions of the same object to the database.
 * Then find one version by date and update its logical properties.
 *
 * This should result in a new technical version of the updated temporal version.
 *
 * Use Case:
 * + A person has the name "Allen Doe" from Date d1 to Date d2
 * + The person marries on d3 and changes his name to "Allen Dot"
 * + We find out that the original name should have been "John Doe", and record this to the database.
 */

class UpdateTemporalTest extends FlatSpec with Matchers {

  val d1 = new DateTime(2013,6,7,0,0,0).toDate // 2013-06-07
  val d2 = new DateTime(2013,6,8,0,0,0).toDate // 2013-06-08
  val d3 = new DateTime(2013,6,9,0,0,0).toDate // 2013-06-09
  val d4 = new DateTime(2013,6,10,0,0,0).toDate // 2013-06-10

  behavior of "an InMemoryBitemporalDatabase"
  
  it should "allow updates for existing objects and retrieval with a bitemporal context" in {
    
    InMemoryBitemporalDatabase.clearDatabase()

    val s  = new Student("Allen", "Doe")
    val t  = new Student("Allen", "Dot")

    val sPeriod = new Period(d1, d2)
    val tPeriod = new Period(d3, d4)

    val sId = InMemoryBitemporalDatabase.store(s, sPeriod)

    // save the other temporal version
    InMemoryBitemporalDatabase.updateLogical(sId, t, tPeriod)
    val context1 =  new BitemporalContext(new Date(), d1)
    val retrieved1 : Temporal[Student, Int] = InMemoryBitemporalDatabase.findLogical(new Student(), sId, context1).get
    Thread.sleep(10)
    
    InMemoryBitemporalDatabase.updateLogical(sId, new Student("John", "Doe"), sPeriod)

    // since we are searching with the old context (the transaction time before we did the update),
    // we are expecting the old version of the Student.
    val retrieved2 = InMemoryBitemporalDatabase.findLogical(new Student(), sId, context1).get
    retrieved2.element should equal (retrieved1.element)

    // search with the new transaction time. Thus we expect to find the correct/updated name.
    val retrieved3 : Temporal[Student, Int] = InMemoryBitemporalDatabase.findLogical(new Student(), sId, new BitemporalContext(new Date(), d1)).get;
    retrieved3.element.firstName should be ("John")
  }
}
```

Installing to a local Maven repository
--------------------------------------

The command `sbt publishM2` in the root directory of the checked out sources will build a jar from this project and install this to your local maven repository under `~/.m2/...` together with a pom.xml describing its dependencies. Subsequently you can use this library from other projects.

Usage from Java
---------------
Once the maven jar and pom is installed to your local repository, you can use this library from other projects, also from Java projects. The subdirectory example-java in this repository includes an example of how to use the bitemporal mongodb layer from java. Here is an excerpt of the relevant code. 

BitemporalTest.java:
```Java
public class BitemporalTest {

    private static final Date JAN2015 = new DateTime(2015,1,1,0,0,0).toDate();
    private static final Date BETWEEN = new DateTime(2015, 1, 15, 0, 0, 0).toDate();
    private static final Date FEB2015 = new DateTime(2015,2,1,0,0,0).toDate();
    private static final Date AFTER = new DateTime(2015,2,15,0,0,0).toDate();

    @Test
    public void test() {
        BitemporalMongoDb bitemporalMongoDb = new BitemporalMongoDb();
        bitemporalMongoDb.clearCollection(new Room());
        String id1 = bitemporalMongoDb.store(new Room(25.0f, 2.0f), new Period());
        bitemporalMongoDb.update(new Room(24.0f, 2.0f), id1, new Period(JAN2015, FEB2015));
        Room roomInJanuary = bitemporalMongoDb.find(Room.class, id1, new BitemporalContext(new Date(), BETWEEN)).get();
        assertEquals(roomInJanuary.getSize(), 24.0f, 0.1f);
        Room roomInFebruary = bitemporalMongoDb.find(Room.class, id1, new BitemporalContext(new Date(), AFTER)).get();
        assertEquals(roomInFebruary.getSize(), 25.0f, 0.1f);
    }

}
```

Room.java:
```java
public class Room {

    private float size;
    private float height;

    public Room() {
        // for clear collection
    }

    public Room(float size, float height) {
        this.size = size;
        this.height = height;
    }

    public float getSize() {
        return size;
    }

    public void setSize(float size) {
        this.size = size;
    }

    public float getHeight() {
        return height;
    }

    public void setHeight(float height) {
        this.height = height;
    }
}
```



