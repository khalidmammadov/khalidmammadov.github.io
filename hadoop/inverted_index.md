# Building Inverted Index with Hadoop MapReduce (for a search engine)

### Overview

In this article I am going to demonstrate how to build inverted index from crawled web page data for fast web look up. This type of index is used by all search engines to quickly find search term you normally type into search box and also used in some DBs for faster look-ups.

To give brief description on inverted index, it is an index where every word in the db has related list of web pages i.e. you type news it gives you bbc.co.uk, guardian.co.uk etc. This, provides efficient way looking up web for related data. The down side of it obviously is that a need to build and maintain a huge database of data (index) for this quick look up to work. Also, since it's a vast amount of data we need to store it in distributed computer environment and use sophisticated programs to pin point correct machine in the network to retrieve this information. Actually, this is the main reason Hadoop and MapReduce is build for initially.

### Objective

In this article I am going to go through an example which I used to build inverted index when I was working on Jumponio.com (now obsolete) project at one point to be stored eventually in Amazon Dynamo DB.

### Preparation

For this example I have already crawled the web and acquired all metadata needed and saved in JSON format using method I explained in my previous post.  My development and testing environment is as per below:

- Java - for MapReduce code
- Maven - dependency management and package builder
- Eclipse - IDE
- Docker - hadoop cluster running on Docker containers
- Ubuntu - of course.... ;)

Although, you can run Hadoop in your local machine in Pseudo-Distributed mode you can also run it in fully distributed mode using method I explained in my previous posts on how to set up Hadoop cluster on Docker containers. I am going to use distributed mode and run it my own cluster.

### Data

As I mentioned before, I have already crawled data got it in the local host as per below:
```
khalid@ubuntu:/JumpOnCrawler/out/original$ ls -l | head
total 504460
-rw-rw-r-- 1 khalid khalid   11233 Oct 22  2016 0044.co.uk.json
-rw-rw-r-- 1 khalid khalid   11405 Oct 23  2016 007escorts.co.uk.json
-rw-rw-r-- 1 khalid khalid   20978 Oct 22  2016 01100111011001010110010101101011.co.uk.json
-rw-rw-r-- 1 khalid khalid       0 Oct 22  2016 01casinos.co.uk.json
-rw-rw-r-- 1 khalid khalid   16134 Oct 22  2016 020.co.uk.json
-rw-rw-r-- 1 khalid khalid   18288 Oct 23  2016 08direct.co.uk.json
-rw-rw-r-- 1 khalid khalid       0 Oct 23  2016 10001games.co.uk.json
-rw-rw-r-- 1 khalid khalid       0 Oct 23  2016 1000-vouchers.co.uk.json
-rw-rw-r-- 1 khalid khalid      75 Oct 22  2016 1001hosting.co.uk.json
```

A total 504460 crawled files in JSON format.

Just to give you an idea below is how the first file looks like:
```
khalid@ubuntu:/media/khalid/home/feo/crawler/JumpOnCrawler/out/original$ head 0044.co.uk.json
[
{"url": "/", "base": "http://www.0044.co.uk/", "description": "International SIM cards & Data SIM cards help you to avoid all roaming charges.", "title": "International SIM cards & Cheap International Calls"},
{"url": "/about-us.htm#terms", "text": "Terms & Conditions", "base": "http://www.0044.co.uk/", "description": "0044 Ltd have been specialists for over 10 years in the sourcing the best global SIM Cards for cheap international calls and data roaming.", "title": "About 0044 Ltd Global SIM Cards and Cheap International Calls"},
{"url": "http://www.0044.co.uk/contact-us.htm", "text": "contact us", "base": "http://www.0044.co.uk/", "description": "We can help and advise about all aspects of global sim cards and cheap international calls", "title": "Global SIM Card advice - Contact 0044"},
....
```

Now, we need to upload it into my Hadoop cluster:

```

hdfs dfs -put original /users/khalid/
```
Application

Now, the application part. I have already developed and uploaded the code into my repository where you can fork from.

```

git clone https://github.com/khalidmammadov/hadoop_reverse_index_mapreduce.git

```

Here, I split the app into the three parts the Map, Reduce and Main classes. Lets have a look into each one:

Here the Mapper gets every line from the file as a text and then it converts it to JSON object. Then after some check the values from {"keywords", "description", "title"} keys are parsed in the word processor where it extracts every word, cleans and de-duplicates using java Set class. Then keys from the set are submitted to the context.

De-duplication and clean

**ParserMap.java**
```
String[] words = obj.has(key) ? obj.getString(key).split("\\s|,") : null;

if (words != null) {
    for (String val: words) {

        if (val.trim().length() > 0) {

            uniqueList.add(val.trim().replaceAll("[\\W+|\\\"]", ""));

        }

    }
}
```

Mapper

```
int lastIdx = line.length() - ((line.substring(line.length() - 1)) == "," ? 1 : 0);

JSONObject obj = null;
try {
    obj = new JSONObject(line.substring(0, lastIdx));
} catch (JSONException e) {
    return;
}

if (obj != null) {

    Set < String > strSet = new HashSet < String > ();

    if (!obj.has("keywords") || !obj.has("base")) {
        return;
    }

    String[] keys = {
        "keywords",
        "description",
        "title"
    };

    for (String s: keys) {
        setWord(obj, s, strSet);
    }

    if (!strSet.isEmpty()) {

        for (String w: strSet) {

            word.set(w);

            context.write(word, new Text(obj.getString("base")));

        }
    }

}

```

In reducer phase, the app de-duplicates values i.e. web addresses and join locations (where that word was mentioned) with pipe (|) and submits them into context.

**ReverseIndexReducer.java**
```
public void reduce(Text key, Iterable < Text > values, Context context) throws IOException, InterruptedException {

    String locations = "";

    Set <String> strSet = new HashSet <String> ();

    //Remove duplicates
    for (Text val: values) {
        strSet.add(val.toString());
    }
    //Join URLs
    for (String str: strSet) {
        locations = String.join("|", locations, str);
    }

    result.set(locations);
    context.write(key, result);

}
```

Finally, the runner app where the all pieces come together. Here we set mapper and reducer classes. Also, we indicate that input and output directories are supplied externally. You can see in the code I have left some commented lines. They are there for local testing purposes (hence, you can see input and output directories in the git repository). Additionally, I have added a line of code to delete target folder before starting so it does not fail for reruns.
Here is the code:

**Main.java**
```
public int run(String[] args) throws Exception {

        Job job = Job.getInstance(getConf(), "reverse index builder");

        job.setJarByClass(Main.class);
        job.setMapperClass(ParserMap.class);
        job.setReducerClass(ReverseIndexReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(Text.class);

        Path inputFilePath = new Path(args[0]);
        Path outputFilePath = new Path(args[1]);
        FileInputFormat.addInputPath(job, inputFilePath);
        FileOutputFormat.setOutputPath(job, outputFilePath);

        //For local run
        //Path inputFilePath = new Path("/home/khalid/workspace/hadoop_reverse_index_mapreduce/input");
        //Path outputFilePath = new Path("/home/khalid/workspace/hadoop_reverse_index_mapreduce/output/1");
        //FileInputFormat.addInputPath(job, inputFilePath);
        //FileOutputFormat.setOutputPath(job, outputFilePath);

        FileSystem fs = FileSystem.get(new URI(outputFilePath.toString()), getConf());
        fs.delete(outputFilePath, true);

        return job.waitForCompletion(true) ? 0 : 1;

}
```
Running the code

Before we can run the code we need to package it up. As mentioned I use Maven and it's great!

```
mvn clean
mvn package
```

Ok, now we can run it.

```
HADOOP_CLIENT_OPTS="-Xmx4g" hadoop jar target/hadoop_reverse_index_mapreduce-0.0.1-SNAPSHOT.jar co.uk.khalidmammadov.hadoop.Main \
hdfs://192.168.1.5:9000/users/khalid/input \
hdfs://192.168.1.5:9000/users/khalid/out
```

you might wonder what the heck is this?: HADOOP_CLIENT_OPTS="-Xmx4g"
This is here to stop my job from failure due to insufficient memory on the client so I am increasing it to 4Gb.

After running through 504460 files (~5min in my case) it finished successfully and produced output files:

```
khalid@ubuntu:$ hdfs dfs -ls hdfs://192.168.1.5:9000/users/khalid/out
Found 2 items
-rw-r--r--   3 khalid khalid          0 2017-12-25 22:22 hdfs://192.168.1.5:9000/users/khalid/out/_SUCCESS
-rw-r--r--   3 khalid khalid   97350207 2017-12-25 22:22 hdfs://192.168.1.5:9000/users/khalid/out/part-r-00000
```

Let's check out what it is inside part-r-00000. It's has got too much and long data so, I will put here only snippets to show the point:
```
giftcard	|http://www.hotdeals.co.uk/|http://www.rspb.org.uk/|http://www.currys.co.uk/gbuk/index.html
giftcards	|https://www.amazon.co.uk/|http://www.thegiftcardcentre.co.uk/
giftcardtcs	|http://blissworld.co.uk
giftcertificates	|https://www.spreadshirt.co.uk/
giftsCrabtree	|http://www.crabtree-evelyn.co.uk/home.html
giftsets	|https://www.solippy.co.uk/|https://www.eastendcosmetics.co.uk/
gifts	|http://www.flyerzone.co.uk/|http://deeplinkdirectory.co.uk|https://www.spreadshirt.co.uk/|http://wish.co.uk|http://www.adventuretravelmagazine.co.uk/|http://www.printster.co.uk/|http://www.expressgiftservice.co.uk/|http://www.designitnow.co.uk/|http://www.emmabridgewater.co.uk/
```

### Summary

As you can see we have got a URL for every word we are after and we can find related information hopefully :) easily with our own search engine! It's hopefully because here I have not mentioned anything about search engine algorithms such as Markov Chain or Page Rank which are the essential parts to find required data, but I think this is completely different story! Please do do not hesitate to leave comments below and Thanks for reading!