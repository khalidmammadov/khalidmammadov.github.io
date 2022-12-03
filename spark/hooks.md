# Adding "hooks" to Apache Spark core to act on various Spark events (Scala)

Occasionally, you may need to run some code on various Spark events, such as 
doing some extra steps on Spark startup (e.g. creating a Spark GlobalTempViews or collect some adhoc stats)

This article explains on example how one can hook up some additional actions to Spark.
I am going to create a simple hook that is listening to spark Application start up and shutdown 
events and logs respective times to a log file in /tmp location.

_Note: The is written in Scala but can be used in conjunction with Python as well. It only 
requires setting up one config and jar file as explained in below Scala example_

_Note: Databricks users can copy below explained jar to **databricks/jars** folder in **dbfs**
and add relevant config in the cluster configuration_

## Requirements
- java (I am using classic v8 here)
- [sbt](https://www.scala-sbt.org/1.x/docs/index.html) - to build and run the project 

## Or just clone repo
You can clone code in this example from:[SparkHook](https://github.com/khalidmammadov/scala.git) 

## Set up
Let's start by creating a Scala project using **sbt** and provide project name.
```bash
$ sbt new scala/scala-seed.g8
...
name [Scala Seed Project]: SparkHook

```

Navigate to directory and remove sample file
```bash
$ cd SparkHook
# Remove unnecessary sample file
$ rm src/main/scala/example/Hello.scala
```

Add `build.sbt` file with below content. This defines Spark dependencies.

**We need to make sure version of the Scala is the same as one that was used to build 
that specific Spark version.** In my example I am using Spark 3.3.0 which was compiled with 2.12.15.
So, I also use that version for this project.
```scala
ThisBuild / version := "0.1.0-SNAPSHOT"

ThisBuild / scalaVersion := "2.12.15"

lazy val root = (project in file("."))
  .settings(
    name := "SparkHook"
  )


libraryDependencies ++= Seq("org.apache.spark" %% "spark-core"% "3.3.0",
  "org.apache.spark" %% "spark-sql" % "3.3.0",
  "org.apache.spark" %% "spark-hive" % "3.3.0")
```

Add example file:
```bash
$ touch src/main/scala/AdHocListener.scala
```
and paste below into that file:
```scala
import org.apache.spark.SparkFirehoseListener
import org.apache.spark.scheduler.{SparkListenerApplicationEnd, SparkListenerApplicationStart, SparkListenerEvent}
import org.joda.time.LocalDateTime

import java.io.FileWriter


class AdHocListener extends SparkFirehoseListener {

  var appName: String = _

  def logApp(txt: String): Unit = {
    val fw = new FileWriter("/tmp/startedApps.log", true)
    fw.write(txt)
    fw.close()
  }

  override def onEvent(event: SparkListenerEvent): Unit = {
    try {
      val time = LocalDateTime.now()
      event match {
        case app: SparkListenerApplicationStart =>
          appName = app.appName
          logApp(s"${app.appName} started at $time \n")
          println(app.appName)
        case app: SparkListenerApplicationEnd =>
          logApp(s"$appName stopped at $time \n")
        case _ =>
      }
    } catch {
      case e: Throwable =>
        println(e)
    }
  }
}

```
### AdHocListener explanation

SparkFirehoseListener class is an implementation of SparkListenerInterface  that defines all possible events
that we could listen to (or hook on) to do some additional tasks. Below are the some examples:
- onStageCompleted
- onTaskStart
- onJobStart
- etc.

So, SparkFirehoseListener implements all these events and forwards them to **onEvent** method which we need to 
override when we subclass from it, in order to catch relevant events which we are doing by using 
Scala's pattern matching for specific type.

Then we say which events we are interested and here I am saying I want to act on **SparkListenerApplicationStart**
and **SparkListenerApplicationEnd** events. 

Once message is received we simple grab the application name 
and save into a temp file with current timestamp. We also print that name to standard output.

Application end event does the same but it uses pre-saved application name instead as "**app**" variable 
for this event does not contain this information.

## Testing
In order to test we need to create **jar** file and copy it to Spark's classpath or set in the config.
It's important to note that this jar is going to be part of Spark core

So, we need to first build and package it to a Jar 
```scala
$ sbt package
```

## Spark testing environment
I am great fun of Docker and containers so I have already got an image for these kind of testing which 
I am going to use here to create a sandbox environment to test it. But you can test anywhere you like even in
Python env where you can copy this jar to jars folder in pyspark site-packages.

Below command creates a container with Spark already available in home directory and maps my local path
where previously built jar is located. (Other commands says just delete it when done and open interactive shell) 
```shell
docker run --rm -it -v /home/examples/SparkHook/target/scala-2.12:/home/extrajars/ spark-local:0.4
```

Once in container we can now test it by calling spark-shell and providing configs.
First config asks Spark to add additional listener (and we can add more) and
second config adds our jar to classpath 
```shell
./spark/bin/spark-shell --conf spark.extraListeners=AdHocListener --conf spark.driver.extraClassPath=extrajars/sparkhook_2.12-0.1.0-SNAPSHOT.jar
```
when we run, it should emit standard messages and one extra which is the app name (Spark shell):
```shell
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
22/12/03 21:50:37 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Spark shell
Spark context Web UI available at http://481853e9fb36:4040
Spark context available as 'sc' (master = local[*], app id = local-1670104238126).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 3.3.0
      /_/
         
Using Scala version 2.12.15 (OpenJDK 64-Bit Server VM, Java 1.8.0_312)
Type in expressions to have them evaluated.
Type :help for more information.

```

We can now close the spark-shell and make another test via spark-submit:
```shell
./spark/bin/spark-submit --conf spark.extraListeners=AdHocListener --conf spark.driver.extraClassPath=extrajars/sparkhook_2.12-0.1.0-SNAPSHOT.jar --class org.apache.spark.examples.SparkPi --master local[1] /home/spark/examples/jars/spark-examples_2.12-3.3.0.jar
```
This will produce a lot of logs but we can see again app name printed in there:
```shell
22/12/03 21:50:54 INFO SparkContext: Running Spark version 3.3.0
22/12/03 21:50:54 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
22/12/03 21:50:54 INFO ResourceUtils: ==============================================================
22/12/03 21:50:54 INFO ResourceUtils: No custom resources configured for spark.driver.
22/12/03 21:50:54 INFO ResourceUtils: ==============================================================
...
Spark Pi
...
```

Now, we can check our log file to see if indeed the application names and start/stop times are logged:
```shell
cat /tmp/startedApps.log
Spark shell started at 2022-12-03T21:50:38.576 
Spark shell stopped at 2022-12-03T21:50:43.393 
Spark Pi started at 2022-12-03T21:50:55.988 
Spark Pi stopped at 2022-12-03T21:50:57.537
```

# Summary
This simple example explains how one can easily add hooks to Spark code 
to do various actions and each event provides relevant variables to use in the action.
Although, in this example I am using *--conf spark.driver.extraClassPath* to add jar but you can also
add it into Spark's classpath whichever you like such as adding during your docker image build. Then 
you won't need to specify this parameter, and it will work for all apps! 

