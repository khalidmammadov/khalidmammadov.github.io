# Apache Spark ML: Using Random Forest Classifier to predict MOT test results [Scala]

In this article I am going to use Spark's ML functionality to do some predictions.
The complete code sample can be found here [spark-ml-random-forest](https://github.com/khalidmammadov/scala/tree/main/spark-ml-random-forest)

### Objective
The goal is to find if given particular car is going to pass MOT test or not.
For this purpose I am going to use number of columns as `features`. Particularly, I will use: 
make, model, first use year, milage, engine capacity and fuel type.   
_I know that these alone are not enough to make accurate prediction if a car is going to fail an MOT test
as it also depends on many other factors of the car e.g. tiers, wipers, lites etc._

But I will do my best to do prediction using available data. 

### Data
I will use MOT test results provided by [UK's Department for Transport](https://www.data.gov.uk/dataset/e3939ef8-30c7-4ca8-9c7c-ad9475cc9b2f/anonymised-mot-tests-and-results).
It contains all test results data from all cars in UK from last few years. I have picked up 2020 as an example and 
downloaded that zip file. 

Sample data from the files:
```commandline
user:~/download/dft_test_result_2020$ head dft_test_result-from-2020-10-01_00-00-00-to-2021-01-01_00-00-00.csv 
test_id,vehicle_id,test_date,test_class_id,test_type,test_result,test_mileage,postcode_area,make,model,colour,fuel_type,cylinder_capacity,first_use_date
840415303,1367588451,2020-10-01,4,NT,P,36788,LE,FORD,TRANSIT,SILVER,DI,2198,2014-10-03
797766817,146223609,2020-10-01,4,NT,P,21856,LU,NISSAN,JUKE,BLUE,PE,1618,2017-02-03
968360761,144911119,2020-10-01,4,NT,P,45041,LE,MINI,PACEMAN,BLACK,PE,1598,2015-03-26
755118331,740243933,2020-10-01,4,NT,P,30389,LU,FORD,FOCUS,BLACK,PE,1999,2017-11-22
1224251677,1368373019,2020-10-01,4,NT,P,38749,LU,LAND ROVER,RANGE ROVER EVOQUE,BLACK,DI,1999,2016-09-27
243336499,359109671,2020-10-01,4,RT,P,45397,LU,PEUGEOT,2008,WHITE,DI,1398,2014-06-30
158039527,163738397,2020-10-01,4,RT,P,93462,LU,MERCEDES-BENZ,C,GREY,DI,2143,2015-05-05
456578929,703774257,2020-10-01,4,NT,P,33700,HD,AUDI,A6,BLACK,DI,1968,2017-06-30
584524387,1498372684,2020-10-01,4,NT,PRS,173053,E,VAUXHALL,CORSAVAN,BLUE,DI,1686,2001-12-14
```

As you can see data is in CSV format and it has got all the cars' details with registration year.
This data will be cleaned and some irrelevant column will be dropped like `test_id`.  

### Method
Since this problem is `binary` classification problem I will use `decision tree classification` models and 
in particular `Random Forest classifier` to do the actual job. I think it's well enough for given problem.

### Implementation
I am using Apache Spark's ML libraries and ecosystem to create an ML model that can predict this results.

Ok, lets gets started. 

You can download complete source code of the model below:

https://github.com/khalidmammadov/scala/blob/main/spark-ml-random-forest/src/main/scala/Main.scala

I am going to list part of the above code below and explain what each parts do.

First we import all necessary packages and classes. These will be explained in due course.
Then code wrapped to a Scala object to be able to execute it as an application
```commandline
import org.apache.spark.ml.{Pipeline, PipelineModel}
import org.apache.spark.ml.classification.{RandomForestClassificationModel, RandomForestClassifier}
import org.apache.spark.ml.evaluation.MulticlassClassificationEvaluator
import org.apache.spark.ml.feature.{IndexToString, StringIndexer, StringIndexerModel, VectorAssembler, VectorIndexer}
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions.expr

object Main {
  def main(args: Array[String]): Unit = {
  ...
```

We then instantiate a Spark session and set log level to ERROR to reduce logs during execution.
```commandline
val spark = SparkSession.builder.master("local[*]").appName("Predict MOT result").getOrCreate()
spark.sparkContext.setLogLevel("ERROR")
```

The data from GOV.UK website is in CSV format and that means Spark needs to read whole file to
do filtration and read data and so it would be better to save it as immutable Parquet columnar format as it very 
efficient to read and filter data and we are not intending to alter it anyway.

So, this reads CSV files and does some simple transformations and saves the data as Parquet.
```commandline
val filesPath = "/home/user/download/dft_test_result_2020"
val motData = spark.read.option("header", "true").csv(filesPath)
motData
  .withColumn("year", year('first_use_date))
  .repartition('year, 'make, 'model)
  .write
  .format("parquet")
  .saveAsTable("/home/user/motdata")
```

_Disclaimer: I know this can be done as one step like checkpointing but I'll keep it as is for simplicity_

Then it reads the same data as Parquet and filters only new tests. As cars can retake the test later again once they 
fix the faults and pass the test. So, we are interested in the first test for a given year and to find out weather it passed 
or not first time.
```commandline
val _data =
      spark.read
        .parquet("/home/user/motdata")
        .filter(
        """test_type = 'NT'
            | and test_result in ('F', 'P')
            |""".stripMargin)
```

### Feature engineering 

Then it cleans data by dropping not used columns and converting numeric columns.

I also fiter here specific mark and model due to data 
size and my server that does not have enough capacity to handle it all. So, we will focus on one car model 
but with different milage, year, engine and fuel type.

_Disclaimer: Fiesta chosen due to popularity reasons and data size. It's one of the popular cars in UK!_
```commandline
val data = _data
  .drop("colour", "vehicle_id", "test_id", "test_date", "test_class_id", "test_type", "postcode_area", "first_use_date")
  .withColumn("indexed_test_mileage", expr("int(test_mileage)"))
  .withColumn("indexed_year", expr("int(year)"))
  .withColumn("indexed_cylinder_capacity", expr("int(cylinder_capacity)"))
  .filter("make = 'FORD'")
  .filter("model = 'FIESTA'")
```

This is how data now looks like:
```commandline
data.show()
+-----------+------------+----+------+---------+-----------------+----+--------------------+------------+-------------------------+
|test_result|test_mileage|make| model|fuel_type|cylinder_capacity|year|indexed_test_mileage|indexed_year|indexed_cylinder_capacity|
+-----------+------------+----+------+---------+-----------------+----+--------------------+------------+-------------------------+
|          P|       86485|FORD|FIESTA|       PE|             1388|2008|               86485|        2008|                     1388|
|          F|       43971|FORD|FIESTA|       PE|             1242|2008|               43971|        2008|                     1242|
|          F|       59431|FORD|FIESTA|       PE|             1242|2008|               59431|        2008|                     1242|
|          P|       46361|FORD|FIESTA|       PE|             1388|2008|               46361|        2008|                     1388|
|          P|      100075|FORD|FIESTA|       PE|             1596|2008|              100075|        2008|                     1596|
|          P|       63489|FORD|FIESTA|       PE|             1388|2008|               63489|        2008|                     1388|
.....
```

Now, we need to set our prediction column that we are trying to find out given the test sample. 
So we `test_result` column that can have either `P` or `F` and since it's a string column we use StringIndexer class
to convert them to a numeric values.
```commandline
val labelIndexer = new StringIndexer()
      .setInputCol("test_result")
      .setOutputCol("indexedTestResult")
      .fit(data)
```

We declare which column in our source data are string and which columns we are going to use 
as `features`.
```commandline
val stringCols = List("make", "model", "fuel_type")
val featureCols = List("test_mileage", "fuel_type", "year", "cylinder_capacity")
```

Here I use helper function getStringIndexer to create a StringIndexer instance given a column name.
Then data is cleaner further to remove `nulls` and then all string columns are traversed to be converted 
to numerical values using helper function.
```commandline
def getStringIndexer(col: String): StringIndexer = {
  new StringIndexer()
    .setInputCol(col)
    .setOutputCol(s"indexed_$col")
}

val nullsFiltered =
  featureCols
    .foldLeft(data){case (df, c)=> df.filter(s"$c is not null")}
    .localCheckpoint(true)

val convertedStringColsDf =
  stringCols
    .map(getStringIndexer)
    .foldLeft(nullsFiltered){case(df, i)=>i.fit(df).transform(df)}
```

Quick check how data looks like after transformations so far:
```commandline
onvertedStringColsDf.show()
+-----------+------------+----+------+---------+-----------------+----+--------------------+------------+-------------------------+------------+-------------+-----------------+
|test_result|test_mileage|make| model|fuel_type|cylinder_capacity|year|indexed_test_mileage|indexed_year|indexed_cylinder_capacity|indexed_make|indexed_model|indexed_fuel_type|
+-----------+------------+----+------+---------+-----------------+----+--------------------+------------+-------------------------+------------+-------------+-----------------+
|          P|       86485|FORD|FIESTA|       PE|             1388|2008|               86485|        2008|                     1388|         0.0|          0.0|              0.0|
|          F|       43971|FORD|FIESTA|       PE|             1242|2008|               43971|        2008|                     1242|         0.0|          0.0|              0.0|
|          F|       59431|FORD|FIESTA|       PE|             1242|2008|               59431|        2008|                     1242|         0.0|          0.0|              0.0|
|          P|       46361|FORD|FIESTA|       PE|             1388|2008|               46361|        2008|                     1388|         0.0|          0.0|              0.0|
|          P|      100075|FORD|FIESTA|       PE|             1596|2008|              100075|        2008|                     1596|         0.0|          0.0|              0.0|
|          P|       63489|FORD|FIESTA|       PE|             1388|2008|               63489|        2008|                     1388|         0.0|          0.0|              0.0|
|          P|       81777|FORD|FIESTA|       PE|             1242|2008|               81777|        2008|                     1242|         0.0|          0.0|              0.0|
|          P|       45296|FORD|FIESTA|       PE|             1242|2008|               45296|        2008|                     1242|         0.0|          0.0|              0.0|
```
As you can see we have got source columns and corresponding encoded numerical ones as well.

Now, we need to collect these numerical columns into one column as a vector and VectorAssembler is here to help.
I am also using Spark's DataFrame checkpointing feature to materialise lazy transformation and save them as Parquet files 
locally for faster repeated reads. 
```commandline
    val featuresAssembler = new VectorAssembler()
      .setInputCols(featureCols.map(c=>s"indexed_$c").toArray)
      .setOutputCol("features")

    val withFeaturesVectorDf =
      featuresAssembler.
        transform(convertedStringColsDf)
        .drop(featureCols.map(c=>s"indexed_$c"): _*)
        .localCheckpoint(true)
```
Quick check the result and as expected one column with all features bundled:
```commandline
withFeaturesVectorDf.show()
+-----------+------------+----+------+---------+-----------------+----+------------+-------------+--------------------+
|test_result|test_mileage|make| model|fuel_type|cylinder_capacity|year|indexed_make|indexed_model|            features|
+-----------+------------+----+------+---------+-----------------+----+------------+-------------+--------------------+
|          P|       86485|FORD|FIESTA|       PE|             1388|2008|         0.0|          0.0|[86485.0,0.0,2008...|
|          F|       43971|FORD|FIESTA|       PE|             1242|2008|         0.0|          0.0|[43971.0,0.0,2008...|
|          F|       59431|FORD|FIESTA|       PE|             1242|2008|         0.0|          0.0|[59431.0,0.0,2008...|
|          P|       46361|FORD|FIESTA|       PE|             1388|2008|         0.0|          0.0|[46361.0,0.0,2008...|
|          P|      100075|FORD|FIESTA|       PE|             1596|2008|         0.0|          0.0|[100075.0,0.0,200...|
|          P|       63489|FORD|FIESTA|       PE|             1388|2008|         0.0|          0.0|[63489.0,0.0,2008...|
|          P|       81777|FORD|FIESTA|       PE|             1242|2008|         0.0|          0.0|[81777.0,0.0,2008...|
```

Then we need to set this `features` column as input data to our training model. 
```commandline
val featureIndexer = new VectorIndexer()
  .setInputCol("features")
  .setOutputCol("indexedFeatures")
  .setMaxCategories(4)
  .fit(withFeaturesVectorDf)
```

We then split data into two parts by 70% and 30% for training and testing purposes:
```commandline
val Array(trainingData, testData) = withFeaturesVectorDf.randomSplit(Array(0.7, 0.3))
```

Now, we create an instance of our RandomForestClassifier
```commandline
val rf = new RandomForestClassifier()
  .setLabelCol("indexedTestResult")
  .setFeaturesCol("indexedFeatures")
  .setNumTrees(30)
  .setMaxBins(50)
```

Also, create an object that will convert our predictions back to original values:
```commandline
val labelConverter = new IndexToString()
  .setInputCol("prediction")
  .setOutputCol("predictedTestResult")
  .setLabels(labelIndexer.labelsArray(0))
```

Then we use `pipelines` to chain the transformations, training and converter as stages:
```commandline
val pipeline = new Pipeline()
  .setStages(Array(labelIndexer, featureIndexer, rf, labelConverter))
```

## Training

Here actual training of the model happens and a Model object is returned.
As you can imagine training process take long time and in order to save time for testing and checks for 
development we can save it into a location and load it as necessary. I am using a simple toggle `USE_SAVED_MODEL`
in the code for simplicity 

```commandline
val USE_SAVED_MODEL = false
val path = "/home/user/models/spark-rf/model"
// Train model. This also runs the indexers.
val model = if (USE_SAVED_MODEL) {
  PipelineModel.load(path)
}else {
  val m = pipeline.fit(trainingData)
  m.write.overwrite().save(path)
  m
}
```

## Prediction
We can now use our test data to make predictions using our trained model

```commandline
val predictions = model.transform(testData)
```

We can then save predictions to a file and calculate accuracy of our model: 
```commandline
    val cols = List("predictedTestResult", "test_result") ++ featureCols
    // predictions.selectExpr(cols: _*).show(1500)
    predictions
      .repartition(1)
      .selectExpr(cols: _*)
      .write
      .mode("overwrite")
      .csv("/tmp/output.csv")

    // Select (prediction, true label) and compute test error.
    val evaluator = new MulticlassClassificationEvaluator()
      .setLabelCol("indexedTestResult")
      .setPredictionCol("prediction")
      .setMetricName("accuracy")
    val accuracy = evaluator.evaluate(predictions)
    println(s"Test Accuracy = $accuracy")
    println(s"Test Error = ${(1.0 - accuracy)}")
```

Here is the output values in my tests:
```commandline
Test Accuracy = 0.7326978697853702
Test Error = 0.2673021302146298
```

## Conclusion
As you can see 73% accuracy is not great but as I said from beginning to predict this output we need to 
have more information about a car's current condition usage etc. 

In later articles I will use this MOT data again using different classification models (perhaps
`Gradient boosting machines`) and see if I am 
going to get any better output given the same data. 

