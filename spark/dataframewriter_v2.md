# Spark DataFrameWriterV2 example using Sqlite (Scala)

This article explains on an example how we can use DataFrameWriterV2 API 
introduced in Spark 3.0.0.

[DataFrameWriterV2](https://issues.apache.org/jira/browse/SPARK-23521) is improvement over 
existing DataFrameWriter to optimise writes to external databases.

I am going to use this new API to create a new Sqlite database and store a small 
DataFrame into it for simplicity.

_Note: Although this example uses Scala, it's trivial to achieve the same with Python
following the same principles defined here._

## Requirements
- java (I am using classic v8 here)
- [sbt](https://www.scala-sbt.org/1.x/docs/index.html) - to build and run the project 

## Set up
Let's start by creating a Scala project using **sbt** and provide project name.
```bash
$ sbt new scala/scala-seed.g8
...
name [Scala Seed Project]: dataframewriter_v2

```

Navigate to directory and remove sample file
```bash
$ cd dataframewriter_v2
# Remove unnecessary sample file
$ rm src/main/scala/example/Hello.scala
```

Add `build.sbt` file with below content. This defines Spark, Sqlite
java libraries to download for build and run.
```scala
import Dependencies._

ThisBuild / scalaVersion     := "2.13.8"
ThisBuild / version          := "0.1.0-SNAPSHOT"
ThisBuild / organization     := "com.example"
ThisBuild / organizationName := "example"

lazy val root = (project in file("."))
  .settings(
    name := "dataframewriter_v2",
    libraryDependencies += scalaTest % Test
  )

// See https://www.scala-sbt.org/1.x/docs/Using-Sonatype.html for instructions on how to publish to Sonatype.

libraryDependencies ++= Seq("org.apache.spark" %% "spark-core"% "3.3.0",
  "org.apache.spark" %% "spark-sql" % "3.3.0",
  "org.apache.spark" %% "spark-hive" % "3.3.0")

libraryDependencies += "org.json4s" % "json4s-jackson_2.13" % "3.7.0-M11"
// https://mvnrepository.com/artifact/org.json4s/json4s-core

// https://mvnrepository.com/artifact/org.xerial/sqlite-jdbc
libraryDependencies += "org.xerial" % "sqlite-jdbc" % "3.39.3.0"
```

Add example file:
```bash
$ touch src/main/scala/example/DataFrameWriterV2Test.scala
```
and paste below into that file:
```scala
package example

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.execution.datasources.v2.jdbc.JDBCTableCatalog

object DataFrameWriterV2Test extends App {

  val url = s"jdbc:sqlite:sample.db"

  val spark = SparkSession.builder.master("local[*]").appName("V2 app").getOrCreate()
  spark.conf.set("spark.sql.catalog.sqllite", classOf[JDBCTableCatalog].getName)
  spark.conf.set("spark.sql.catalog.sqllite.url", url)

  val df = spark.range(2)
  df.writeTo("sqllite.tab1").create()
  spark.sql("select * from sqllite.tab1").show()
}

```
### DataFrameWriterV2 part explanation
It defines a new **catalog** called `sqllite` and sets handler of that to `JDBCTableCatalog`
which is a generic class that is acting as an interface to JDBC backed data source. 

We also defined a JDBC `url` that points to `sample.db` file in current directory.

These settings are needed so we can refer to V2 target table through catalog. 

Although, it's not obvious where `DataFrameWriterV2` starts working, it kicks in when
we say `df.writeTo` and if you check the type of that it will provide **`DataFrameWriterV2`** (as 
apposed to **`DataFrameWriter`** usual)
```python
scala> df.writeTo("sqllite.tab1")
val res4: org.apache.spark.sql.DataFrameWriterV2[Long] = org.apache.spark.sql.DataFrameWriterV2@df3a499

```

## Execute
While inside `dataframewriter_v2` directory, start **sbt** and compile the project.
This will load `build.sbt`, resolve and download dependencies and compile the project.

```scala
$ sbt
...
sbt:dataframewriter_v2> reload
sbt:dataframewriter_v2> compile
```

Now, we are ready to run:
```scala
sbt:dataframewriter_v2> run
...
...
+---+
| id|
+---+
|  0|
|  1|
+---+

```

It creates a new Sqlite database called `sample.db` in the local
directory and creates a new table called `tab1` within that. 
Then it runs select SQL from that table.

Check:
```shell
$ ls -l
total 28
-rw-rw-r-- 1 khalid khalid  878 Sep 25 17:58 build.sbt
drwxrwxr-x 3 khalid khalid 4096 Sep 25 17:43 project
**-rw-rw-r-- 1 khalid khalid 8192 Sep 25 17:59 sample.db**
drwxr-xr-x 2 khalid khalid 4096 Sep 25 17:59 spark-warehouse
drwxrwxr-x 4 khalid khalid 4096 Sep 25 17:41 src
drwxrwxr-x 7 khalid khalid 4096 Sep 25 17:59 target

```

# Summary
This simple example explains how one can easily write data to a JDBC based
backend database using `DataFrameWriterV2` and you can change it easily to 
your database of choice by adding a jar for that database to dependencies 
and providing a JDBC URL for that with all required secrets and options.

