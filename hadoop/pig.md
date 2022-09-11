# Apache Pig with examples

### Overview

This article is about one of the main data manipulation tools used on Hadoop environment called Pig. Pig simplifies data processing operations by using its own language called Pig Latin. So, you don't need to write MapReduce code for every data manipulation task you are going to make and instead you write few lines of code in Pig and it will do the rest for you. Here, I will demonstrate how it can be done. First, we will set up Pig, prepare data and write example code.

### Prerequisites

This article makes assumption that you have got Java and running Hadoop cluster. If you don't have it you can use one of my previous article to set this up.

### Set up

Download Pig from from Apache web site

```

cd /opt
wget http://www-eu.apache.org/dist/pig/pig-0.17.0/pig-0.17.0.tar.gz

```

Extract the tar archive

```

tar -xvf pig-0.17.0.tar.gz
rm pig-0.17.0.tar.gz

```

Set up environment variables for Hadoop and Pig and add executables to PATH in .bashrc file so they are there all the time

below is my .bashrc file. (bottom of the file)

```
....
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
export HADOOP_HOME=/opt/hadoop-2.8.2;
export HIVE_HOME=/opt/apache-hive-2.3.2-bin
export PIG_HOME=/opt/pig-0.17.0;
export PATH="/opt/apache-maven-3.5.2/bin:$HIVE_HOME/bin:$PIG_HOME/bin:$HADOOP_HOME/bin";:$PATH
```

Execute .bashrc file in your shell to make changes effective

```
. .bashrc
```

check if Pig working. Just type pig and you should get Pig prompt called Grunt. Here is my output:

```
khalid@ubuntu:~$ pig
17/12/29 17:23:12 INFO pig.ExecTypeProvider: Trying ExecType : LOCAL
17/12/29 17:23:12 INFO pig.ExecTypeProvider: Trying ExecType : MAPREDUCE
17/12/29 17:23:12 INFO pig.ExecTypeProvider: Picked MAPREDUCE as the ExecType
2017-12-29 17:23:12,040 [main] INFO  org.apache.pig.Main - Apache Pig version 0.17.0 (r1797386) compiled Jun 02 2017, 15:41:58
2017-12-29 17:23:12,040 [main] INFO  org.apache.pig.Main - Logging error messages to: /home/khalid/pig_1514568192039.log
2017-12-29 17:23:12,055 [main] INFO  org.apache.pig.impl.util.Utils - Default bootup file /home/khalid/.pigbootup not found
2017-12-29 17:23:12,473 [main] INFO  org.apache.hadoop.conf.Configuration.deprecation - mapred.job.tracker is deprecated. Instead, use mapreduce.jobtracker.address
2017-12-29 17:23:12,473 [main] INFO  org.apache.pig.backend.hadoop.executionengine.HExecutionEngine - Connecting to hadoop file system at: hdfs://192.168.1.30:9000
2017-12-29 17:23:12,896 [main] INFO  org.apache.pig.PigServer - Pig Script ID for the session: PIG-default-5dada147-783d-4d88-9f1e-b1b2af602684
2017-12-29 17:23:12,896 [main] WARN  org.apache.pig.PigServer - ATS is disabled since yarn.timeline-service.enabled set to false
grunt>
```

As you can see it automatically picked up my local Hadoop installation. (Please note that I have got have Hadoop installation folder on my local machine. But, actual cluster i.e. Namenode and DataNodes are sitting on separate servers. So, in order your local Hadoop (or hive) tools to execute commands (e.g. hdfs ..., hadoop ...) without specifying location of the cluster in the command line (e.g. hadoop fs -cat hdfs://192.168.1.30:9000/users/.... ) you need to set default cluster address in core-site.xml)

Exercise

I am going to create a test script that loads data from CSV file and extract information I am after. The test file is FX Rates data that you can freely download from here:

```
wget https://github.com/datasets/exchange-rates/tree/master/data/daily.csv
```

Just to give you an idea how internally it looks:

```
khalid@ubuntu:~/test/exchange-rates/data$ head daily.csv
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

we need remove first row as it's not required.

```
cat daily.csv | grep -v Country> daily_data.csv
```

Next, I am going to upload it into Hadoop HDFS:

```

hadoop fs -mkdir /user/khalid/fx_rates
hadoop fs -put daily_data.csv /user/khalid/fx_rates/
hadoop fs -ls /user/khalid/fx_rates/

```

Now we can start Pig and work on this sample data.

Below is the sample code. Every line of statement is usually assigned to a variable and used in the next line of code as an input. This links tasks to each other and creates a chain of actions.

```
grunt> data = LOAD '/user/khalid/fx_rates/daily_data.csv' USING PigStorage(',') AS (date, country, rate);
grunt> filter_country = FILTER data BY country eq 'Australia';
grunt> extract_year = FOREACH filter_country GENERATE SUBSTRING(date, 0, 4) as year, country, rate;
grunt> filter_year_2000 = FILTER extract_year BY year eq '2000';
grunt> illustrate filter_year_2000;
```

Now, lets go through the script line by line.
First we tell Pig where the file is located and what the column splitter is and what the columns are. Here file is located in HDFS cluster and the file format comma delimited file (CSV). The final output is assigned to a variable data (as a reference).

```
data = LOAD '/user/khalid/fx_rates/daily_data.csv' USING PigStorage(',') AS (date, country, rate);
```

Just to note that at this phase Pig does not do any LOAD but just validates the statement. All Pig statements are like that. They are only executed at the end with explicit statements which we will see soon.

Next, as an example I want to filter data by country. So, use FILTER statement and output again is assigned to a variable.

```
filter_country = FILTER data BY country eq 'Australia';
```

At this point I could stop and show data but there are a lot of it so I want to do one more filter. This also shows how we can chain statements.
So, I am going to filter by year. To do this, we need first extract year from the date string. It can be done by inbuilt function SUBSTRING and we print rest of the columns:

```
grunt> extract_year = FOREACH filter_country GENERATE SUBSTRING(date, 0, 4) as year, country, rate;
```

Next, we can filter by year:

```
grunt> filter_year_2000 = FILTER extract_year BY year eq '2000';
```

Ok, now the complete script is ready and to show how data is going to look like at every stage execution we can run handy ILLUSTRATE command which is great to check to see if everything as planned:

```
grunt> ILLUSTRATE filter_year_2000;
```

Below is full execution of the script and the output:

```
khalid@ubuntu:~/docker/test$ pig
17/12/29 17:59:32 INFO pig.ExecTypeProvider: Trying ExecType : LOCAL
17/12/29 17:59:32 INFO pig.ExecTypeProvider: Trying ExecType : MAPREDUCE
17/12/29 17:59:32 INFO pig.ExecTypeProvider: Picked MAPREDUCE as the ExecType
2017-12-29 17:59:32,299 [main] INFO  org.apache.pig.Main - Apache Pig version 0.17.0 (r1797386) compiled Jun 02 2017, 15:41:58
2017-12-29 17:59:32,299 [main] INFO  org.apache.pig.Main - Logging error messages to: /home/khalid/docker/test/pig_1514570372298.log
2017-12-29 17:59:32,315 [main] INFO  org.apache.pig.impl.util.Utils - Default bootup file /home/khalid/.pigbootup not found
2017-12-29 17:59:32,752 [main] INFO  org.apache.hadoop.conf.Configuration.deprecation - mapred.job.tracker is deprecated. Instead, use mapreduce.jobtracker.address
2017-12-29 17:59:32,752 [main] INFO  org.apache.pig.backend.hadoop.executionengine.HExecutionEngine - Connecting to hadoop file system at: hdfs://192.168.1.30:9000
2017-12-29 17:59:33,146 [main] INFO  org.apache.pig.PigServer - Pig Script ID for the session: PIG-default-ff0e98a6-d216-40a9-9cd0-330d13eac3ba
2017-12-29 17:59:33,146 [main] WARN  org.apache.pig.PigServer - ATS is disabled since yarn.timeline-service.enabled set to false
grunt> data = LOAD '/user/khalid/fx_rates/daily_data.csv' USING PigStorage(',') AS (date, country, rate);
grunt> filter_country = FILTER data BY country eq 'Australia';
2017-12-29 17:59:41,153 [main] WARN  org.apache.pig.newplan.BaseOperatorPlan - Encountered Warning IMPLICIT_CAST_TO_CHARARRAY 1 time(s).
grunt> extract_year = FOREACH filter_country GENERATE SUBSTRING(date, 0, 4) as year, country, rate;
2017-12-29 17:59:41,204 [main] WARN  org.apache.pig.newplan.BaseOperatorPlan - Encountered Warning USING_OVERLOADED_FUNCTION 1 time(s).
2017-12-29 17:59:41,204 [main] WARN  org.apache.pig.newplan.BaseOperatorPlan - Encountered Warning IMPLICIT_CAST_TO_CHARARRAY 2 time(s).
grunt> filter_year_2000 = FILTER extract_year BY year eq '2000';
2017-12-29 17:59:42,921 [main] WARN  org.apache.pig.newplan.BaseOperatorPlan - Encountered Warning USING_OVERLOADED_FUNCTION 1 time(s).
2017-12-29 17:59:42,921 [main] WARN  org.apache.pig.newplan.BaseOperatorPlan - Encountered Warning IMPLICIT_CAST_TO_CHARARRAY 2 time(s).
grunt> illustrate filter_year_2000;
....
---------------------------------------------------------------------------
| data     | date:bytearray    | country:bytearray    | rate:bytearray    |
---------------------------------------------------------------------------
|          | 2000-05-11        | Australia            | 1.7271            |
|          | 1978-10-02        | Australia            | 0.8651            |
|          | 2000-05-11        | 0                    | 1.7271            |
---------------------------------------------------------------------------
-------------------------------------------------------------------------------------
| filter_country     | date:bytearray    | country:bytearray    | rate:bytearray    |
-------------------------------------------------------------------------------------
|                    | 2000-05-11        | Australia            | 1.7271            |
|                    | 1978-10-02        | Australia            | 0.8651            |
-------------------------------------------------------------------------------------
-----------------------------------------------------------------------------------
| extract_year     | year:chararray    | country:bytearray    | rate:bytearray    |
-----------------------------------------------------------------------------------
|                  | 2000              | Australia            | 1.7271            |
|                  | 1978              | Australia            | 0.8651            |
-----------------------------------------------------------------------------------
---------------------------------------------------------------------------------------
| filter_year_2000     | year:chararray    | country:bytearray    | rate:bytearray    |
---------------------------------------------------------------------------------------
|                      | 2000              | Australia            | 1.7271            |
---------------------------------------------------------------------------------------

```

Please, note that ILLUSTRATE command is nice command to show you the execution flow and shows you a snippets of the data. In order to actually execute the code and extract all data and print it into a standard output you need to execute DUMP command:

```
grunt> DUMP filter_year_2000;
```

and you should get output like this:

```
....
(2000,Australia,1.5172)
(2000,Australia,1.5239)
(2000,Australia,1.5267)
(2000,Australia,1.5291)
(2000,Australia,1.5272)
(2000,Australia,1.5242)
(2000,Australia,1.5209)
...
```
Grouping data

Now, I would like to create similar code to group the data and find MIN and MAX fx rates for every country and every year. The example is similar to one from my previous articles  finding MIN and MAX FX rates for every country for every year using MapReduce also it's the similar to this one where I use Apache Hive to find the same result.

Here is the script:

```
v_data = LOAD '/user/khalid/fx_rates/daily_data.csv' USING PigStorage(',') AS (date:chararray, country:chararray, rate:float);
v_filtered = FILTER v_data BY country eq 'Australia';
v_year = FOREACH v_filtered GENERATE SUBSTRING(date, 0, 4) as year, country, rate;
v_grouped = GROUP v_year BY (year, country);
v_min_max = FOREACH v_grouped GENERATE group,  MIN($1.rate) as min_rate, MAX($1.rate) as max_rate;

DUMP v_min_max;
```

It's very similar to our first script with the exceptions that here I specify types for every column. This is important as to calculate MIN/MAX we need to tell Pig that the rate column is actually a number type. In previous example Pig implicitly converted all columns to String.

```
v_data = LOAD '/user/khalid/fx_rates/daily_data.csv' USING PigStorage(',') AS (date:chararray, country:chararray, rate:float);
```

Also, in this script I have prefixed all stages/variables with "v_" so it's clear what are the variables and what aren't.
Lines from two to three are the same. Remaining lines are the important changes.

We first group data by GROUP command:
```
v_grouped = GROUP v_year BY (year, country);
```

And then we find MIN and MAX from grouped data:

```
v_min_max = FOREACH v_grouped GENERATE group,  MIN($1.rate) as min_rate, MAX($1.rate) as max_rate;
```

and DUMP the data:

```
DUMP v_min_max;
```

This time I saved it in a file (fx_rates.pig) as an alternative way to run the code but you can execute it in the Grunt prompt as per usual as well.

```

khalid@ubuntu:~/test/pig$ cat fx_rates.pig
v_data = LOAD '/user/khalid/fx_rates/daily_data.csv' USING PigStorage(',') AS (date:chararray, country:chararray, rate:float);

v_filtered = FILTER v_data BY country eq 'Australia';

v_year = FOREACH v_filtered GENERATE SUBSTRING(date, 0, 4) as year, country, rate;

v_grouped = GROUP v_year BY (year, country);

v_min_max = FOREACH v_grouped GENERATE group,  MIN($1.rate) as min_rate, MAX($1.rate) as max_rate;

DUMP v_min_max;

```

Now, we can execute it in linux shell:

```
khalid@ubuntu:~/docker/test/pig$ pig fx_rates.pig
....
Success!

Job Stats (time in seconds):
JobId Maps Reduces MaxMapTime MinMapTime AvgMapTime MedianMapTime MaxReduceTime MinReduceTime AvgReduceTime MedianReducetime Alias Feature Outputs
job_1514561589733_0010 1 1 3 3 3 3 2 2 2 2 v_data,v_filtered,v_grouped,v_min_max,v_year GROUP_BY,COMBINER hdfs://192.168.1.30:9000/tmp/temp-1812120051/tmp-2140948113,
Input(s):
Successfully read 226533 records (6108533 bytes) from: "/user/khalid/fx_rates/daily_data.csv"

Output(s):
Successfully stored 47 records (1598 bytes) in: "hdfs://192.168.1.30:9000/tmp/temp-1812120051/tmp-2140948113"

Counters:
Total records written : 47
Total bytes written : 1598
Spillable Memory Manager spill count : 0
Total bags proactively spilled: 0
Total records proactively spilled: 0

Job DAG:
job_1514561589733_0010

2017-12-29 21:45:21,154 [main] INFO  org.apache.hadoop.yarn.client.RMProxy - Connecting to ResourceManager at /192.168.1.30:8032
2017-12-29 21:45:21,157 [main] INFO  org.apache.hadoop.mapred.ClientServiceDelegate - Application state is completed. FinalApplicationStatus=SUCCEEDED. Redirecting to job history server
2017-12-29 21:45:21,181 [main] INFO  org.apache.hadoop.yarn.client.RMProxy - Connecting to ResourceManager at /192.168.1.30:8032
2017-12-29 21:45:21,185 [main] INFO  org.apache.hadoop.mapred.ClientServiceDelegate - Application state is completed. FinalApplicationStatus=SUCCEEDED. Redirecting to job history server
2017-12-29 21:45:21,210 [main] INFO  org.apache.hadoop.yarn.client.RMProxy - Connecting to ResourceManager at /192.168.1.30:8032
2017-12-29 21:45:21,213 [main] INFO  org.apache.hadoop.mapred.ClientServiceDelegate - Application state is completed. FinalApplicationStatus=SUCCEEDED. Redirecting to job history server
2017-12-29 21:45:21,239 [main] INFO  org.apache.pig.backend.hadoop.executionengine.mapReduceLayer.MapReduceLauncher - Success!
2017-12-29 21:45:21,243 [main] INFO  org.apache.pig.data.SchemaTupleBackend - Key [pig.schematuple] was not set... will not generate code.
2017-12-29 21:45:21,256 [main] INFO  org.apache.hadoop.mapreduce.lib.input.FileInputFormat - Total input files to process : 1
2017-12-29 21:45:21,256 [main] INFO  org.apache.pig.backend.hadoop.executionengine.util.MapRedUtil - Total input paths to process : 1
((1971,Australia),0.8412,0.899)
((1972,Australia),0.7854,0.8418)
((1973,Australia),0.6718,0.7868)
((1974,Australia),0.6723,0.7678)
((1975,Australia),0.7323,0.7988)
((1976,Australia),0.793,0.9946)
((1977,Australia),0.8764,0.924)
((1978,Australia),0.8432,0.8905)
((1979,Australia),0.8682,0.9173)
((1980,Australia),0.8465,0.9372)
((1981,Australia),0.841,0.8909)
((1982,Australia),0.8857,1.0715)
((1983,Australia),1.0096,1.1726)
((1984,Australia),1.0365,1.2213)
((1985,Australia),1.2165,1.5785)
((1986,Australia),1.3396,1.6781)
((1987,Australia),1.3581,1.5528)
((1988,Australia),1.1344,1.4245)
((1989,Australia),1.1217,1.3523)
((1990,Australia),1.1975,1.3419)
((1991,Australia),1.2483,1.3321)
((1992,Australia),1.2987,1.4658)
((1993,Australia),1.3864,1.5504)
((1994,Australia),1.2857,1.462)
((1995,Australia),1.2982,1.4085)
((1996,Australia),1.2225,1.3665)
((1997,Australia),1.2534,1.5408)
((1998,Australia),1.456,1.8018)
((1999,Australia),1.4899,1.6184)
((2000,Australia),1.4954,1.9562)
((2001,Australia),1.7507,2.0713)
((2002,Australia),1.7397,1.9763)
((2003,Australia),1.3298,1.7765)
((2004,Australia),1.2533,1.462)
((2005,Australia),1.2541,1.3772)
((2006,Australia),1.2636,1.4172)
((2007,Australia),1.0673,1.2947)
((2008,Australia),1.0207,1.6466)
((2009,Australia),1.0673,1.587)
((2010,Australia),0.9849,1.2237)
((2011,Australia),0.9069,1.0579)
((2012,Australia),0.9254,1.0322)
((2013,Australia),0.9453,1.1289)
((2014,Australia),1.054,1.235)
((2015,Australia),1.2177,1.4457)
((2016,Australia),1.2793,1.4588)
((2017,Australia),1.239,1.3829)
2017-12-29 21:45:21,337 [main] INFO  org.apache.pig.Main - Pig script completed in 19 seconds and 422 milliseconds (19422 ms)

```
### Summary

If you compare results from my below articles to this result you will see the outputs are exactly the same. These all different ways of getting the same result. Obviously it's very simple example and in reality there will be compelling reasons why you would prefer to use one method over another.

To me, the main advantage using for example Pig over Hive it's the fact that Pig allows to work with data without imposing metadata layer (in Hive you need to create a tables, import etc.) and you can easily load data into Hadoop start analysing it immediately.

http://www.khalidmammadov.co.uk/apache-hive-in-action/

http://www.khalidmammadov.co.uk/finding-min-and-max-fx-rates-for-every-country-using-hadoop-mapreduce/