# Apache HBASE on Docker containers


<!-- wp:heading {"level":3} -->
<h3>Overview</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In this short article I am going to install HBASE distributed database on Docker containers and set underlying file system to HDFS which is configured beforehand (see<a href="http://www.khalidmammadov.co.uk/distributed-hadoop-on-docker-containers/"> this post</a>)</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Here I will create a distributed Hbase cluster with Master, Backup Master and two Region Servers with ZooKeeper. The Backup Master and one Region Server are going to share the same host (although it can also be separated). This set up follows official Hbase set up guide "<a href="https://hbase.apache.org/book.html#quickstart_fully_distributed">2.4. Advanced - Fully Distributed</a>" from Apache web site.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Here is how it is going to look like at a the end:</p>
<!-- /wp:paragraph -->

<!-- wp:table {"className":"tableblock frame-all grid-all spread"} -->
<figure class="wp-block-table tableblock frame-all grid-all spread"><table><thead><tr><th>Container, Node Name</th><th>Master</th><th>ZooKeeper</th><th>RegionServer</th></tr></thead><tbody><tr><td>
<p class="tableblock">hbase_master</p>
</td><td>
<p class="tableblock">yes</p>
</td><td>
<p class="tableblock">yes</p>
</td><td>
<p class="tableblock">no</p>
</td></tr><tr><td>
<p class="tableblock">hbase_regionserver_backup</p>
</td><td>
<p class="tableblock">backup</p>
</td><td>
<p class="tableblock">yes</p>
</td><td>
<p class="tableblock">yes</p>
</td></tr><tr><td>
<p class="tableblock">hbase_regionserver2</p>
</td><td>
<p class="tableblock">no</p>
</td><td>
<p class="tableblock">yes</p>
</td><td>
<p class="tableblock">yes</p>
</td></tr></tbody></table></figure>
<!-- /wp:table -->

<!-- wp:heading {"level":3} -->
<h3>Preparation</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>As mentioned earlier I have already got my Hadoop cluster running so <a href="http://www.khalidmammadov.co.uk/distributed-hadoop-on-docker-containers/">you should have as well.</a></p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Then we need to download Hbase from Apache web site. I am using 1.4.0 but you are free to chose version you like but you will also need to make appropriate updates in the Dockerfile.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>So:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```
cd ~
mkdir -p docker/hbase.img

cd docker/hbase.img/
wget http://www-eu.apache.org/dist/hbase/1.4.0/hbase-1.4.0-bin.tar.gz
```

<!-- /wp:html -->

<!-- wp:paragraph -->
<p>Now clone my git repository specifically created for this article:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```

git clone  https://github.com/khalidmammadov/hbase_docker.git

```
<!-- /wp:html -->

<!-- wp:paragraph -->
<p>This repository contains all required data.</p>
<!-- /wp:paragraph -->

<!-- wp:preformatted -->
<pre class="wp-block-preformatted">Dockerfile	
backup-masters	
bootstrap_master.sh
bootstrap_region.sh
hbase-env.sh
hbase-site.xml
regionservers
run.sh
</pre>
<!-- /wp:preformatted -->

<!-- wp:paragraph -->
<p>I will go through the list to explain what each one contains:</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2><span class="css-truncate css-truncate-target"><a id="3254677a7917c6c01f55212f86c57fbf-ec8444e0eaba0b8fc1a2057ae7e7205fac1b7247" class="js-navigation-open" title="Dockerfile" href="https://github.com/khalidmammadov/hbase_docker/blob/master/Dockerfile">Dockerfile</a></span></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This is main Docker file that has got all the step by step instructions to build an image. The image is based on latest ubuntu image from Docker repos.&nbsp; It sets up JAVA (Open JDK), installs HBASE, makes SSH passwordless for inner cluster communication for Hbase and open ports for connectivity.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2><span class="css-truncate css-truncate-target"><a id="c3ddef3bdcfee4ffd74b141c67a30b32-bc632411b9bc58a76a1933cbbfe7098b57d8a8ce" class="js-navigation-open" title="backup-masters" href="https://github.com/khalidmammadov/hbase_docker/blob/master/backup-masters">backup-masters</a></span></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Here we set hostname or IP address of backup server. I have set it to hbase_regionserver_backup as per plan.</p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2><span class="css-truncate css-truncate-target"><a id="e545e9adf05575d30c48c11368480759-be53f0ee086ced2f112f0c793f842419bfebdb97" class="js-navigation-open" title="hbase-env.sh" href="https://github.com/khalidmammadov/hbase_docker/blob/master/hbase-env.sh">hbase-env.sh</a></span></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This file is executed before Hbase starts, so I set here location of JAVA_HOME.</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```

export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/jre

```
<!-- /wp:html -->

<!-- wp:heading -->
<h2><span class="css-truncate css-truncate-target"><a id="55b845df1e87888d1132f2cf18792b2a-669bdab08a385bf3810fbb74d62eddd954502942" class="js-navigation-open" title="hbase-site.xml" href="https://github.com/khalidmammadov/hbase_docker/blob/master/hbase-site.xml">hbase-site.xml</a></span></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This file has all main parameters for the cluster set up. So, we first set Hbase directory, the ZooKeeper estate and data dir. As you can see I pointed the file stores to a HDFS on Hadoop cluster.</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```

<property>
  <name>hbase.cluster.distributed</name>
  <value>true</value>
</property>

<property>
  <name>hbase.rootdir</name>
  <value>hdfs://namenode:9000/hbase</value>
</property>

<property>
  <name>hbase.zookeeper.quorum</name>
  <value>hbase_regionserver2,hbase_regionserver_backup,hbase_master</value>
</property>

<property>
  <name>hbase.zookeeper.property.dataDir</name>
  <value>hdfs://namenode:9000/zookeeper</value>
</property>


```
<!-- /wp:html -->

<!-- wp:heading -->
<h2><span class="css-truncate css-truncate-target"><a id="33b130adbacf8753dd0b274887024eac-824c429ef153b62cc3451e9660f37447dedfc3cb" class="js-navigation-open" title="regionservers" href="https://github.com/khalidmammadov/hbase_docker/blob/master/regionservers">regionservers</a></span></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>We need to list all RegionServers in this file as a separate line:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```
 hbase_regionserver_backup
 hbase_regionserver2
 ```
<!-- /wp:html -->

<!-- wp:heading -->
<h2><span class="css-truncate css-truncate-target"><a id="c452e2da8cafc5b9f646ad523ad97822-840cf425eb76a24b51a306e6330c42471f4cd89c" class="js-navigation-open" title="bootstrap_region.sh" href="https://github.com/feorean/hbase_docker/blob/master/bootstrap_region.sh">bootstrap_region.sh</a></span></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This script will be used for RegionServer containers to start SSH on start up.<br></p>
<!-- /wp:paragraph -->

<!-- wp:heading -->
<h2><a href="https://github.com/feorean/hbase_docker/blob/master/bootstrap_region.sh">bootstrap_region.sh</a></h2>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>This script starts Hbase cluster (also starts SSH).</p>
<!-- /wp:paragraph -->

<!-- wp:heading {"level":3} -->
<h3>Building and Running</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>Before we start Hbase we need to build Docker image and define network parameters. We are going to run Docker build command in a second but before that make sure your build folder looks like below (if you did as described in Preparation section)</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```
khalid@ubuntu:~/docker/hbase.img$ ll
total 109752
drwxrwxr-x  5 khalid khalid      4096 Jan  3 00:04 ./
drwxr-xr-x 11 khalid khalid      4096 Jan  1 15:33 ../
-rw-rw-r--  1 khalid khalid        13 Jan  2 22:51 backup-masters
-rw-rw-r--  1 khalid khalid       106 Jan  2 23:20 bootstrap_master.sh
-rw-rw-r--  1 khalid khalid        62 Jan  2 23:27 bootstrap_region.sh
-rw-rw-r--  1 khalid khalid      1581 Jan  2 23:59 Dockerfile
drwxrwxr-x  7 khalid khalid      4096 Jan  1 15:30 hbase-1.4.0/
-rw-rw-r--  1 khalid khalid 112324081 Dec  9 00:44 hbase-1.4.0-bin.tar.gz
-rw-r--r--  1 khalid khalid      7586 Jan  2 23:34 hbase-env.sh
-rw-r--r--  1 khalid khalid      1368 Jan  2 22:32 hbase-site.xml
-rw-r--r--  1 khalid khalid        26 Jan  2 23:58 regionservers
-rw-rw-r--  1 khalid khalid       527 Jan  3 00:04 run.sh
```
<!-- /wp:html -->

<!-- wp:paragraph -->
<p>Now, lets build Docker image from current folder:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```

docker build -t hbase:0.1 .

```
<!-- /wp:html -->

<!-- wp:paragraph -->
<p>Normally you should get successfully built result.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Verify image creation:</p>
<!-- /wp:paragraph -->

<!-- wp:preformatted -->
```
khalid@ubuntu:~$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hbase               0.1                 a211d1985471        21 hours ago        1.31GB
....
```
<!-- /wp:preformatted -->

<!-- wp:paragraph -->
<p>Now the image is ready and we can run Hbase with all required servers.<br>I have created a script to create and run servers on Docker containers as per below:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```
#Clean
docker stop hbase_regionserver2 hbase_regionserver_backup hbase_master

docker ps -a| grep hbase| awk '{system("docker rm " $1)}'



#Run
docker run \
      --name hbase_regionserver_backup \
      --hostname hbase_regionserver_backup \
      -itd \
      --net=hadoop.net \
      hbase:0.1 \
      /bin/bootstrap_region.sh

sleep 2

docker run \
      --name hbase_regionserver2 \
      --hostname hbase_regionserver2 \
      -itd \
      --net=hadoop.net \
      hbase:0.1 \
      /bin/bootstrap_region.sh

sleep 2

docker run \
      --name hbase_master \
      --hostname hbase_master \
      -itd \
      --net=hadoop.net hbase:0.1 \
      /bin/bootstrap_master.sh

```
<!-- /wp:html -->

<!-- wp:paragraph -->
<p>Let me take through this script before executing it.<br>The first line stops any running containers. Then it deletes old containers. These are here mainly for development and testing purposes and deletes old containers before creating new ones.<br>Next comes actual run bit. Here I start Backup Master &amp; Region Server container first from previously created hbase:0.6 image and give it a static IP in the local network. Also, it starts with a shell script as a entry point. In the entry point it start SSH server and opens a shell session.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Afterwards, I start second configured region server similarly on the same local network.<br>Finally, the Master server is started. It has similar execution signature apart from the entry point script. Here along side of creating SSH and opening shell session it actually start the Hbase and communicates through ssh protocol with other nodes/containers and starts them as well.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Lets start:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```
khalid@ubuntu:~/docker/hbase.img$ . run.sh
hbase_regionserver2
hbase_regionserver_backup
hbase_master
c1a2f408db9d
954f7c910c11
7b8295f50c47
9c44fbe244caa4ddaf8ce45860cfb5fe7a6177df9d07810838d9f2625f110e35
792eb7a8c63327fb8dd29cd08b536ec6adc24dc89957ebad7b5738212a4089fd
a767f9ad5550b019ce573795b3b8e7696710ddc78eebc2cc299155b5669c205b

```

<p><br>As you can see three containers started. (It first deletes old ones, you wouldn't see first 6 lines if you are running this script first time)<br>Verify:</p>
```
khalid@ubuntu:~/docker/hbase.img$ docker ps| grep hbase
a767f9ad5550        hbase:0.1           "/bin/bootstrap_mast…"   36 seconds ago      Up 35 seconds                           hbase_master
792eb7a8c633        hbase:0.1           "/bin/bootstrap_regi…"   38 seconds ago      Up 37 seconds                           hbase_regionserver2
9c44fbe244ca        hbase:0.1           "/bin/bootstrap_regi…"   41 seconds ago      Up 40 seconds                           hbase_regionserver_backup
```
Lets connect to each of the containers and see what processes are running:
```
khalid@ubuntu:~/docker/hbase.img$ docker attach hbase_regionserver2
root@792eb7a8c633:/bin#
root@792eb7a8c633:/bin# jps
325 Jps
163 HRegionServer
74 HQuorumPeer
root@792eb7a8c633:/bin# read escape sequence

khalid@ubuntu:~/docker/hbase.img$ docker attach hbase_regionserver_backup
root@9c44fbe244ca:/bin#
root@9c44fbe244ca:/bin# jps
157 HRegionServer
228 HMaster
464 Jps
74 HQuorumPeer
root@9c44fbe244ca:/bin# read escape sequence

khalid@ubuntu:~/docker/hbase.img$ docker attach hbase_master
root@a767f9ad5550:/usr/local/hbase/bin#
root@a767f9ad5550:/usr/local/hbase/bin# jps
175 HQuorumPeer
458 Jps
244 HMaster
root@a767f9ad5550:/usr/local/hbase/bin# read escape sequence
```
<!-- /wp:html -->

<!-- wp:paragraph -->
<p>As you can see RegionServers are running on first two and there are two masters as well, one main and one backup.</p>
<!-- /wp:paragraph -->

<!-- wp:paragraph -->
<p>Lets, connect to the web GUI and see how cluster looks like.<br>Navigate to  http://172.19.0.4:16010/master-status  page on your browser. (you can find this IP address buy inspecting "docker network inspect hadoop.net")</p>
<!-- /wp:paragraph -->

<!-- wp:image {"id":396,"sizeSlug":"large"} -->
<figure class="wp-block-image size-large"><img src="http://www.khalidmammadov.co.uk/wp-content/uploads/2019/12/Hbase_master_dash-1024x704.png" alt="" class="wp-image-396"/></figure>
<!-- /wp:image -->

<!-- wp:heading {"level":3} -->
<h3>Summary</h3>
<!-- /wp:heading -->

<!-- wp:paragraph -->
<p>In this article I have showed how it's possible to run distributed Hbase database on Docker containers that uses Hadoop HDFS as a backend storage. Another, interesting thing here is that Hadoop itself also runs on Docker containers as well :).<br>See below my docker estate:</p>
<!-- /wp:paragraph -->

<!-- wp:html -->
```
khalid@ubuntu:~/docker/hbase.img$ docker ps --format '{{.Names}}\t\t\t{{.Image}}'
hbase_master			hbase:0.1
hbase_regionserver2			hbase:0.1
hbase_regionserver_backup			hbase:0.1
wordpressapp			wordpress:0.1
datanode3			datanode:0.1
datanode2			datanode:0.1
datanode1			datanode:0.1
namenode			namenode:0.1

```
<!-- /wp:html -->

<!-- wp:paragraph -->
<p></p>
<!-- /wp:paragraph -->