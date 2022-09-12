# Finding Min and Max FX Rates for every country using Hadoop MapReduce

In this article I will put together MapReduce program that scans data in HDFS file system and make data analysis. For this I have selected freely available data from

[https://github.com/datasets/exchange-rates]()
GitHub repository. This data set have all historic FX rates for many countries.

There are tree files in data folder :
```
daily.csv
monthly.csv
yearly.csv
```
and as their names implies each have Daily, Monthly and Yearly FX rates respectively.

Below is the snippet from files:

**Daily**
```
Date,Country,Value
1971-01-04,Australia,0.8987
1971-01-05,Australia,0.8983
1971-01-06,Australia,0.8977
1971-01-07,Australia,0.8978
1971-01-08,Australia,0.899
1971-01-11,Australia,0.8967
1971-01-12,Australia,0.8964
1971-01-13,Australia,0.8957
1971-01-14,Australia,0.8937
```
**Monthly**
```
Date,Country,Value
1971-01-01,Australia,0.8944
1971-02-01,Australia,0.8898
1971-03-01,Australia,0.8894
1971-04-01,Australia,0.8898
1971-05-01,Australia,0.8894
1971-06-01,Australia,0.8894
1971-07-01,Australia,0.8896
1971-08-01,Australia,0.8837
1971-09-01,Australia,0.8712
```
**Yearly**
```
Date,Country,Value
1971-01-01,Australia,0.8803
1972-01-01,Australia,0.8387
1973-01-01,Australia,0.7047
1974-01-01,Australia,0.695
1975-01-01,Australia,0.7647
1976-01-01,Australia,0.8187
1977-01-01,Australia,0.9024
1978-01-01,Australia,0.874
1979-01-01,Australia,0.8947
```
## Goal

To find minimum and maximum registered FX rate for every country and for each year.

I am taking into account possibility missing data so, we will scan every file and look for data in each of them.

### Data preparation

First thing first we will need to download and upload data into hadoop cluster. I am using my docker cluster that I have created in one of my previous articles.

So, lets download data:
```
cd ~
mkdir fxdata
cd fxdata
git clone https://github.com/datasets/exchange-rates.git
```
Upload data into hdfs:
```
cd data

#Create folder and Copy files
hadoop fs -cp -r exchange-rates /users/khalid/

#Verify
hadoop fs -ls /users/khalid/
```
Test data is ready now we can move and look at the program bit

### Application

For my development I am using Eclipse IDE and Maven for dependency management. I have already developed the application and you can fork it in below git hub repo:

git clone https://github.com/khalidmammadov/hadoop_mapreduce_minmax

### Algorithm

As any Map Reduce program it has two parts map and reduce. In map phase it reads data line by line and generate Key-Value pair for reducer. Here key will be joined Year and Country data e.g. 2010 France. And value will the FX rate.  Next Hadoop does nice job for us by sorting and partitioning this keys together and feeding to the reducer. In the reducer phase  it calculates minimum and maximum fx rates and submits it to the context with new Key-Value pairs for each result i.e. . 1971 Australia MIN.

Below is the full code:
```
package com.fx.minmaxfx;

import java.io.IOException;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.DoubleWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class App {

    public static class ReaderMapper
    extends Mapper < Object, Text, Text, DoubleWritable > {

        private Text YearCountry = new Text();
        private final static String comma = ",";

        private DoubleWritable rate = new DoubleWritable();

        public void map(Object key, Text value, Context context) throws IOException,
        InterruptedException {

            try {
                //Split columns
                String[] columns = value.toString().split(comma);

                if (columns.length & amp; amp; amp; amp; amp; lt; 3 ||
                    columns[2] == null ||
                    columns[2].equals("Value")) {
                    return;
                }
                //Set FX rate
                rate.set(Double.parseDouble(columns[2]));

                //Construct key
                YearCountry.set(columns[0].substring(0, 4) + " " + columns[1]);

                //Submit value into the Context
                context.write(YearCountry, rate);
            } catch (NumberFormatException ex) {
                context.write(new Text("ERROR"), new DoubleWritable(0.0 d));
            }
        }
    }

    public static class MinMaxReducer
    extends Reducer < Text, DoubleWritable, Text, DoubleWritable > {
        private DoubleWritable minW = new DoubleWritable();
        private DoubleWritable maxW = new DoubleWritable();

        public void reduce(Text key, Iterable < DoubleWritable > values,
            Context context
        ) throws IOException,
        InterruptedException {
            double min = 9999999999999999 d;
            double max = 0;

            for (DoubleWritable val: values) {
                min = val.get() < min ? val.get() : min;
                max = val.get() > max ? val.get() : max;
            }
            minW.set(min);
            maxW.set(max);

            //Set key as Year Country Min/Max

            Text minKey = new Text(key.toString() + " " + "MIN");
            Text maxKey = new Text(key.toString() + " " + "MAX");

            context.write(minKey, minW);
            context.write(maxKey, maxW);
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "Min Max FX by YEAR");
        job.setJarByClass(App.class);
        job.setMapperClass(ReaderMapper.class);
        //job.setCombinerClass(MinMaxReducer.class);
        job.setReducerClass(MinMaxReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(DoubleWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}

```

### Compiling

As I said earlier I am using Maven for package and dependency management. Below is how to compile and package the code:
```
mvn compile
```

Output:
```
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building minmaxfx 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ minmaxfx ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/khalid/hadoop_mapreduce_minmax/src/main/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ minmaxfx ---
[INFO] Nothing to compile - all classes are up to date
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 0.591 s
[INFO] Finished at: 2017-12-19T22:10:08Z
[INFO] Final Memory: 13M/319M
[INFO] ------------------------------------------------------------------------

Packaging

Now lets package the code and create jar archive which we will use for actual code execution.

mvn package


You should have similar to below output:

[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] Building minmaxfx 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ minmaxfx ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/khalid/hadoop_mapreduce_minmax/src/main/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ minmaxfx ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ minmaxfx ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /home/khalid/hadoop_mapreduce_minmax/src/test/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ minmaxfx ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ minmaxfx ---
[INFO] Surefire report directory: /home/khalid/hadoop_mapreduce_minmax/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.fx.minmaxfx.AppTest
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.008 sec

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO]
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ minmaxfx ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 1.037 s
[INFO] Finished at: 2017-12-19T22:13:46Z
[INFO] Final Memory: 16M/319M
[INFO] ------------------------------------------------------------------------
```

### Execution

Now we can go and execute the code using jar file on hdfs cluster for our FX data set:

```
hadoop jar target/minmaxfx-0.0.1-SNAPSHOT.jar com.fx.minmaxfx.App \
/users/khalid/exchange-rates/data/ \
/users/khalid/out
```
After successful execution you should find part-r-00000 file in new "out" folder.
Verify:
```
hadoop fs -ls /users/khalid/out
```
Lets now look to the output:
```
hadoop fs -cat /users/khalid/out11/part-r-00000| head -100
```
You should get something like this:
```
1971 Australia MIN	0.8412
1971 Australia MAX	0.899
1971 Austria MIN	23.638
1971 Austria MAX	25.873
1971 Belgium MIN	45.49
1971 Belgium MAX	49.73
1971 Canada MIN	0.9933
1971 Canada MAX	1.0248
1971 Denmark MIN	7.0665
1971 Denmark MAX	7.5067
1971 Finland MIN	4.1926
1971 Finland MAX	4.2154
1971 France MIN	5.3947
1971 France MAX	5.5332
1971 Germany MIN	3.2688
1971 Germany MAX	3.637
1971 Ireland MIN	0.3958
1971 Ireland MAX	0.4157
1971 Italy MIN	600.57
1971 Italy MAX	624.65
1971 Japan MIN	314.96
1971 Japan MAX	358.44
1971 Malaysia MIN	2.8885
1971 Malaysia MAX	3.0867
1971 Netherlands MIN	3.2784
1971 Netherlands MAX	3.6003
...
```

So, as you can see data is grouped by year and country and minimum, maximum rate registered for that year shown.

## Summary

This short code shows how easy is to make calculations on huge amounts of data.
I hope you liked this article.