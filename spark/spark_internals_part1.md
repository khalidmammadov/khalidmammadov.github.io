# Apache Spark internals: RDD and Stage creation with Shuffle behind the scenes 

In these "Spark internals" series I am describing how Spark manages our instructions 
and converts them to actionable tasks that is distributed to a cluster and results are collected 
back.

The example I am going to use is below classic word count code that shows essential parts of the Spark 
i.e. map and shuffle. It uses RDD API and that's easier to comprehend and showcase Spark's distributed framework.  

## Example code
Below code is executed in the Scala REPL with all Spark dependencies included (you can check my other 
articles for how to create an environment)

This code reads a file splits words and the counts them:

```scala
import org.apache.spark.{SparkConf, SparkContext}

import java.nio.file.{Path, Paths}

val logFile:Path = Paths.get(System.getProperty("user.home") + "/dev/sample_data/games.csv")

val conf = new SparkConf().setMaster("local[*]").setAppName("My app")
val sc = new SparkContext(conf)

val txtRDD = sc.textFile(logFile.toString, minPartitions = 2)

val flatmap = txtRDD.flatMap(_.split(" "))
val mapped = flatmap.map(w=>(w, 1))
val reduced = mapped.reduceByKey(_+_)

reduced.collect()
```

## Explanation
There is nothing special first three lines of the code as it's classic Scala/Java code. Then we create 
Spark configuration and set our cluster to local mode via **setMaster** and set app name.

Then we create SparkContext with this configuration.  

Below is explanation what happens for each line where we get RDD as a result for our function invocations:

- **txtRDD**: At this phase SparkContext creates a HadoopRDD instance and then it's wrapped to MapPartitionsRDD instance. 
So, we get something like MapPartitions(HadoopRDD). HadoopRDD is responsible for actually connecting to source file (HDFS, S3, ADLS etc.)
and reading data when it will be called from executors. The partition count for now just registered and 
actual partitions will be computed when necessary (will be explained in later series).
- **flatmap**: This one uses RDD class' flatMap() function that captures the lambda we supply i.e. _.split(" ").
This function is saved in the newly created MapPartitions instance that also has link as a parent to previous MapPartitions
RDD.
- **mapped**: This is very similar to **flatmap** step and creates MapPartitionsRDD and captures the given lambda function.
- **reduced**: At this phase our previous RDD is enriched with additional RDD functions using implicit **PairRDDFunction**
to get this **reduceByKey** method that is using generic **combineByKeyWithClassTag** method to create ShuffleRDD
instance with an aggregator. An aggregator is container that holds functions for data reduction for the given RDD
with functions like "given the values then combine" and "how to create a starting combiner".

See below my illustration of these steps and relationships:  

![RDD](../images/RDDmap.jpg)

It is important to note that the first three steps created MapPartitionsRDD instance and the last one ShuffledRDD. 
These two groups are deliberate and creates distinction between these two types of steps within Spark framework. 
They also define boundary where steps need to be split to create two Stages in the DAG 
and usually referred as "Narrow" and "Wide" transformations. 
- **Narrow**: means that those MapPartitionsRDD can be chained together to do required steps
and can be done independently on each partition on a cluster node
- **Wide**: means for this **reduce** step the data on a node is not sufficient and must be sent to other node to be combined 
to get final result to apply desired lambda function. Hence, ShuffleRDD brakes the chain and forces to move data. 

Each of these stages will receive one part of the RDD i.e. a data partition to work on. Stage 1 will get partitions 
from shuffle phase which is done using in combination with ShuffleDependency class and ShuffleMapStage.

Furthermore, actual implementation of these two stages are a bit different though:
- **Stage 0**: This is implemented/represented internally in DAG as **ShuffleMapStage**. Which means that partitions in this stage must be computed based on functions
defined on MapPartitionsRDD and then made ready for shuffle
- **Stage 1**: This is implemented/represented internally in DAG as **ResultStage**. And means that data this is a final stage 
and data can be collected to combined and produce the result and can be returned to user.

Below picture depicts these two stages. Also, image on the right shows the same but taken from Spark UI. 
![Stages](../images/Stages.jpg)

## Pause and digest
It's vital to stop here and think about what has been said above as often we hear that we need to avoid or 
reduce shuffle steps etc. So, one thing is apparent here is that we need to maximise "narrow" transformations 
as much as possible and "shift to the left" i.e. to the source as much as possible then we can do most processing 
on the source or on given partition first before we can send it to shuffle stage. 
For example check below pseudocode code with two similar tasks that gets to the same result with different performance 
characteristics:
```shell
rdd.reduceByKey(...).filter(...)
rdd.filter(...).reduceByKey(...)
```
First example is inefficient as forces all data to be sent across network to other nodes before it filters not needed
data. And this incurs cost as we ship more data we actually need for our result.
The second example more efficient as it filter unnecessary values first and reduces network traffic and processing power
needed and achieves better performance.

# Summary
I will go further in the next series to show how these are converted into tasks and 
distributed to cluster.
