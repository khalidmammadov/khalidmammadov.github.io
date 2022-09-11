# Apache Hive in action

### Overview
In this article I am going to look at Apache Hive tool. This is mainly designed to work in Data Warehouse projects. The main differentiation form corporate solutions is that (other than the fact that it is free (!)) its running on Hadoop ecosystem. So, it can run on petabytes of data which is distributed across thousands of nodes and generate required output with few lines of code. It is obfuscating "complexities" of MapReduce framework by using SQL syntax and so most RDBMS background developers will find it easy to grasp.

I will set up Hive and run some sample SQLs to demonstrate its abilities. The example I am going to work on is the same as described in my previous article. The key difference here will be that I am not going to touch any Java code and all will be done native Hive commands and SQL.

### Set up
This article makes assumption that you have got running Hadoop cluster. If you don't have it you can use one of my previous article to set this up.

Download Hive from the Apache website:

```
wget http://www-eu.apache.org/dist/hive/hive-2.3.2/apache-hive-2.3.2-bin.tar.gz
```
extract

```
tar -xzvf apache-hive-2.3.2-bin.tar.gz
```
set up environment variables for Hadoop and Hive and add executable to PATH in .bashrc file so they are there when we connect from ssh or direct from shell (I know it can be done in /etc/environments as well).

below is my .bashrc file. (end of it)

```
....
export JAVA_HOME="/usr/lib/jvm/java-8-openjdk-amd64"
export HADOOP_HOME=/opt/hadoop-2.8.2
export HIVE_HOME=/opt/apache-hive-2.3.2-bin
export PIG_HOME=/opt/pig-0.17.0
export PATH="/opt/apache-maven-3.5.2/bin:$HIVE_HOME/bin:$PIG_HOME/bin:$HADOOP_HOME/bin":$PATH
```
Also, make sure you have Job History Server is running on Namenode server.

```

jps

```
Note: it will be running by default if you build your Hadoop cluster from my previous article. (see bootstrap.sh file)

Next, we need to prepare two HDFS folders for Hive to work:

```
sudo ./hadoop fs -mkdir /tmp
sudo ./hadoop fs -mkdir /user
sudo ./hadoop fs -mkdir /user/hive
sudo ./hadoop fs -mkdir /user/hive/warehouse
sudo ./hadoop fs -chmod g+w /user/hive/warehouse
sudo ./hadoop fs -chmod g+w /tmp
```
Note: this will create these folders as the root owner. This will stop running Hive if you use your own user. To resolve it go to Namenode server and add group and user and change owner of the folders:

```

#do on Namenode

#addgroup
groupadd hadoop

#adduser
adduser khalid
#check default group
id khalid
#change default group
usermod -g hadoop khalid
#check default group again
id khalid
#output should be similar:
#uid=1000(khalid) gid=1000(hadoop) groups=1000(hadoop)

#Now, change owner and group of HDFS folders for Hive:
hadoop fs -chown -R khalid:hadoop /tmp
hadoop fs -chown -R khalid:hadoop /user/hive
```
Final notes, in my case Hive is installed on my local machine and I also have Hadoop installation folder on my local machine. But, actual cluster i.e. Namenode and DataNodes are sitting on separate servers. So, in order your local Hadoop (or hive) tools can execute commands (e.g. hdfs ..., hadoop ...) without specifying location of the cluster in the command line (e.g. hadoop fs -cat hdfs://192.168.1.30:9000/users/.... ) you need to set default cluster address in core-site.xml

```
cat /opt/hadoop-2.8.2/etc/hadoop/core-site.xml
```

    <property>
       <name>fs.defaultFS</name>     
       <value>hdfs://192.168.1.30:9000</value>
    </property>

### Hive client
Hive comes with few interfaces for interaction. I will use command line tool. There are two of them:

Hive CLI - looks like its depreciated and community/supporters are discouraging using it.

Beeline - this is the latest available command line tool that replaces Hive CLI. It requires HiveServer2 and a small database (for metadata storage) to operate. Looks like this small database usually is Derby DB. HiveServer2 is a daemon and required when we have many users that use Hive and uses JDBC connections. It can be run in a single user mode as well.

I will use single user mode. Before starting anything it we need to initialise Derby DB.

```
schematool -dbType derby -initSchema
```
This will initiate metadata db in our home directory (~/metastore_db).

### Checking
Now we can run beeline tool to connect Hive in single mode:

```

beeline -u jdbc:hive2://

```
You should get command prompt similar to below.

```
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/apache-hive-2.3.2-bin/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/hadoop-2.8.2/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Connecting to jdbc:hive2://
17/12/28 13:01:44 [main]: WARN session.SessionState: METASTORE_FILTER_HOOK will be ignored, since hive.security.authorization.manager is set to instance of HiveAuthorizerFactory.
Connected to: Apache Hive (version 2.3.2)
Driver: Hive JDBC (version 2.3.2)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 2.3.2 by Apache Hive
0: jdbc:hive2://&amp;gt;;
```
Now, you can run normal Hive SQLs.

```

0: jdbc:hive2://> create table some_table (col1 STRING, col2 STRING);
0: jdbc:hive2://> insert into  some_table values ("Hello", "World");
0: jdbc:hive2://> select * from some_table;
OK
+------------------+------------------+
| some_table.col1  | some_table.col2  |
+------------------+------------------+
| Hello            | World            |
+------------------+------------------+
```
### Exercise
Now, I am going to create a table and load data from external file and run some grouping SQLs. The example is similar to one from previous article finding MIN and MAX FX rates for every country for every year.

get a copy of the test data from here:

```
wget https://github.com/datasets/exchange-rates/tree/master/data/daily.csv
```
delete first line from daily.csv as it has column headers

```
cat daily.csv | grep -v Country&amp;gt; daily_data.csv
```
create a table in Hive:

```

0: jdbc:hive2://> create table fx_rates (fxdate STRING, country STRING, rate DOUBLE)
0: jdbc:hive2://>  ROW FORMAT DELIMITED
0: jdbc:hive2://>          FIELDS TERMINATED BY ',' STORED AS TEXTFILE;

```
Load data into the table:

```
0: jdbc:hive2://> load data local inpath "./daily_data.csv" into table fx_rates;
```
Check the loaded data:

```

0: jdbc:hive2://> select * from fx_rates limit 100;

```
Now, we are ready to extract required data i.e. find MIN and MAX FX rates per country per year.
It's actually very simple one line of SQL:

```

0: jdbc:hive2://> select substr(fxdate, 1, 4) year, country, min(rate) min_rate, max(rate) max_rate from fx_rates group by substr(fxdate, 1, 4), country limit 10;

OK
+--------+---------------+-----------+-----------+
| year  |    country    | min_rate  | max_rate  |
+--------+---------------+-----------+-----------+
| 1971   | Australia     | 0.8412    | 0.899     |
| 1971   | Canada        | 0.9933    | 1.0248    |
| 1971   | Denmark       | 7.0665    | 7.5067    |
| 1971   | Japan         | 314.96    | 358.44    |
| 1971   | Malaysia      | 2.8885    | 3.0867    |
| 1971   | New Zealand   | 0.8394    | 0.8978    |
| 1971   | Norway        | 6.7019    | 7.1465    |
| 1971   | South Africa  | 0.696     | 0.7685    |
| 1971   | Sweden        | 4.8631    | 5.1813    |
| 1971   | Switzerland   | 3.8754    | 4.318     |
+--------+---------------+-----------+-----------+
10 rows selected (17.125 seconds)
```
Under the hood, Hive generates Map and Reduce code automatically and reads and calculates data and stores it in a temporary file before printing that data into the screen.

Now, if I want to save the data for later use I can prepend CREATE TABLE statement and remove LIMIT from suffix for all data:

```

0: jdbc:hive2://> create table fx_grouped_year as select substr(fxdate, 1, 4) year, country, min(rate) min_rate, max(rate) max_rate from fx_rates group by substr(fxdate, 1, 4), country;

```
check the tables

```

0: jdbc:hive2://> show tables;

OK
+------------------------+
|        tab_name        |
+------------------------+
| fx_grouped_year        |
| fx_rates               |
| some_table             |
| values__tmp__table__1  |
+------------------------+
5 rows selected (0.079 seconds)
```

### Summary
As you can see Hive makes it very easy to extract required information from Hadoop. Comparing it to my previous example we achieved the same result with one line of code and for less time.

I hope you enjoyed this article and please do let me know if you see any issues.
