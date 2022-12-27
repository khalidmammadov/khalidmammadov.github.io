# Apache Spark ML: Using Gradient Boost Classifier to predict MOT test results [Python]

In this article I am going to use Spark's ML functionality to do some predictions.
The complete code sample can be found here [spark_ml_gradient_boost](https://github.com/khalidmammadov/python_code/tree/master/spark_ml_gradient_boost)

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
This is `binary` classification problem and I'm going to use `Gradient boost classifier` to do the actual job. 
I have used different classification algorithm in my other post, see [Spark ML Random Forest](spark/spark_ml_random_forest_mot.md) 

### Implementation
I am using Apache Spark's ML libraries and ecosystem to create an ML model that can predict this results.

Ok, lets gets started. 

You can download complete source code of the model below:

https://github.com/khalidmammadov/python_code/blob/master/spark_ml_gradient_boost/spark_gradient_boost.py

I am going to list part of the above code below and explain what each part do.

First we import all necessary packages and classes. These will be explained in due course.
```Python
from pyspark.ml import Pipeline, PipelineModel
from pyspark.ml.classification import GBTClassifier
from pyspark.ml.evaluation import MulticlassClassificationEvaluator
from pyspark.ml.feature import IndexToString, StringIndexer, VectorAssembler, VectorIndexer
from pyspark.sql import SparkSession
from pyspark.sql.functions import expr, year, col
```

We then instantiate a Spark session and set log level to ERROR to reduce logs during execution.
```Python
spark = SparkSession.builder.master("local[*]").appName("MOT Test result prediction").getOrCreate()
spark.sparkContext.setLogLevel("ERROR")
```

The data from GOV.UK website is in CSV format and that means Spark needs to read whole file to
do filtration and read data and so it would be better to save it as immutable Parquet columnar format as it very 
efficient to read and filter data and we are not intending to alter it anyway.

So, this reads CSV files and does some simple transformations and saves the data as Parquet.
```Python
 source_files_path = "/home/user/download/dft_test_result_2020"
    mot_data = spark.read.option("header", "true").csv(source_files_path)
    (mot_data
     .withColumn("year", year(col("first_use_date")))
     .repartition(col("year"), col("make"), col("model"))
     .write
     .format("parquet")
     .saveAsTable("/home/user/motdata"))
```

_Disclaimer: I know this can be done as one step like checkpointing but I'll keep it as is for simplicity_

Then it reads the same data as Parquet and filters only new tests. As cars can retake the test later again once they 
fix the faults and pass the test. So, we are interested in the first test for a given year and to find out weather it passed 
or not first time.
```Python
df = (spark.read
          .parquet("/home/user/motdata")
          .filter("""test_type = 'NT'
                       and test_result in ('F', 'P') """))
```

### Feature engineering 

Then it cleans data by dropping not used columns and converting numeric columns.

I also filter here specific mark and model due to data 
size and my server that does not have enough capacity to handle it all. So, we will focus on one car model 
but with different milage, year, engine and fuel type.
Here Ford Fiesta chosen due to popularity reasons and data size. It's one of the popular cars in UK!

```Python
df = (df
          .drop("colour", "vehicle_id", "test_id", "test_date", "test_class_id", "test_type", "postcode_area",
                "first_use_date")
          .withColumn("indexed_test_mileage", expr("int(test_mileage)"))
          .withColumn("indexed_year", expr("int(year)"))
          .withColumn("indexed_cylinder_capacity", expr("int(cylinder_capacity)"))
          .filter("make = 'FORD'")
          .filter("model = 'FIESTA'"))
```

This is how data now looks like:
```Python
df.show()
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

We declare which column in our source data are string and which columns we are going to use 
as `features`.
```Python
string_cols = ["make", "model", "fuel_type"]
feature_cols = ["test_mileage", "fuel_type", "year", "cylinder_capacity"]
```

Here I use helper function `get_string_indexer` to create a StringIndexer instance given a column name.
Then data is cleaner further to remove `nulls` and then all string columns are traversed to be converted 
to numerical values using helper function.
```Python
    def get_string_indexer(c) -> StringIndexer:
        return (StringIndexer()
                .setInputCol(c)
                .setOutputCol(f"indexed_{c}"))

    for _col in feature_cols:
        df = df.filter(f"{_col} is not null")

    df = df.localCheckpoint(True)

    for _col in string_cols:
        indexer = get_string_indexer(_col)
        df = indexer.fit(df).transform(df)
```

Quick check how data looks like after transformations so far:
```Python
df.show()
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
```Python
    idx_feature_cols = [f"indexed_{c}" for c in feature_cols]
    features_assembler = (VectorAssembler()
                          .setInputCols(idx_feature_cols)
                          .setOutputCol("features"))

    with_features_vector_df = (features_assembler
                               .transform(df)
                               .drop(*idx_feature_cols)
                               .localCheckpoint(True))
```
Quick check the result and as expected one column with all features bundled:
```Python
df.show()
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
```Python
    feature_indexer = (VectorIndexer()
                       .setInputCol("features")
                       .setOutputCol("indexedFeatures")
                       .setMaxCategories(4)
                       .fit(with_features_vector_df))
```

We then split data into two parts by 70% and 30% for training and testing purposes:
```Python
(training_data, test_data) = with_features_vector_df.randomSplit([0.7, 0.3])
```

Now, we create an instance of our GBTClassifier
```Python
gb = (GBTClassifier()
          .setLabelCol("indexedTestResult")
          .setFeaturesCol("indexedFeatures")
          .setMaxIter(10)
          .setFeatureSubsetStrategy("auto"))
```

Now, we need to set our prediction column that we are trying to find out given the test sample. 
So we `test_result` column that can have either `P` or `F` and since it's a string column we use StringIndexer class
to convert them to a numeric values.
```Python
label_indexer = (StringIndexer()
                     .setInputCol("test_result")
                     .setOutputCol("indexedTestResult")
                     .fit(df))
```

Also, create an object that will convert our predictions back to original values:
```Python
label_converter = (IndexToString()
                       .setInputCol("prediction")
                       .setOutputCol("predictedTestResult")
                       .setLabels(list(label_indexer.labelsArray[0])))
```

Then we use `pipelines` to chain the transformations, training and converter as stages:
```Python
pipeline = Pipeline().setStages([label_indexer, feature_indexer, gb, label_converter])
```

## Training

Here actual training of the model happens and a Model object is returned.
As you can imagine training process take long time and in order to save time for testing and checks for 
development we can save it into a location and load it as necessary. I am using a simple toggle `USE_SAVED_MODEL`
in the code for simplicity 

```Python
use_saved_model = False
    path = "/home/user/models/spark-rf/model"
    # Train model. This also runs the indexers.
    model: PipelineModel
    if use_saved_model:
        model = PipelineModel.load(path)
    else:
        model = pipeline.fit(training_data)
        model.write().overwrite().save(path)
```

## Prediction
We can now use our test data to make predictions using our trained model

```Python
predictions = model.transform(test_data)
```

We can then save predictions to a file and calculate accuracy of our model: 
```Python
cols = [*["predictedTestResult", "test_result"], *feature_cols]
    # predictions.selectExpr(cols: _*).show(1500)
    (predictions
     .repartition(1)
     .selectExpr(*cols)
     .write
     .mode("overwrite")
     .csv("/tmp/output.csv"))

    # Select (prediction, true label) and compute test error.
    evaluator = (MulticlassClassificationEvaluator()
                 .setLabelCol("indexedTestResult")
                 .setPredictionCol("prediction")
                 .setMetricName("accuracy"))
    accuracy = evaluator.evaluate(predictions)
    print(f"Test Accuracy = {accuracy}")
    print(f"Test Error = {(1.0 - accuracy)}")
```

Here is the output values in my tests:
```Python
Test Accuracy = 0.7334723480442346
Test Error = 0.26652765195576544
```

## Conclusion
As you can see 73% accuracy is not great (as I said from beginning to predict this output we need to 
have more information about a car's current condition usage and how it was looked after etc.)

But I think it's still usable in certain contexts. For example, one pattern the prediction model revealed that
in general there is a bias for MOT Pass, which makes sense as inherently people taking their cars to certify has already checked their
car's state and fixed obvious issues. Another was the strong correlation between predicted and actual MOT test result 
for one particular Ford Fiesta model with a milage above 120K and particular petrol engine!


