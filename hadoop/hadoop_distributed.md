# Distributed Hadoop cluster on Docker containers 

## Overview

In this article I will set up distributed Hadoop cluster based on Docker containers. For networking I will use user defined BRIDGE network that will segregate the cluster and will allow easy communication between nodes.

## Set up

First clone repository I have created for this article:

```
cd ~
mkdir hadoop_dist
cd hadoop_dist
git clone https://github.com/khalidmammadov/hadoop_dist_docker.git
```

Dowload and save hadoop to the subfolders, one per directory (e.g. NameNode/hadoop-2.8.2.tar.gz and DataNode/hadoop-2.8.2.tar.gz). I am using 2.8.2 version but feel free to download the other version and don't forget to update Dockerfile respectevely.

```wget http://apache.mirror.anlx.net/hadoop/common/hadoop-2.8.2/hadoop-2.8.2.tar.gz
cp hadoop-2.8.2.tar.gz NameNode
cp hadoop-2.8.2.tar.gz DataNode
```
Make changes in configurations (Optional)

Set Hadoop to distributed mode and replication factor and update Namenode fsimage path in hdfs-site.xml file:
```
<configuration>

    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/usr/data/hdfs/namenode</value>
    </property>

    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/usr/data/hdfs/datanode</value>
    </property>


    <property>
        <name>dfs.blocksize</name>
        <value>10485760</value>
    </property>


</configuration>

```

Then update Namenode's Container address (this will be defined during container's run-time) on  in core-site.xml:

```
<configuration>

    <property>
        <name>io.file.buffer.size</name>
        <value>131072</value>
    </property>

    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://namenode:9000</value>
    </property>

</configuration>
```
Building images

Now you can build images with following commands:

```
cd NameNode
sudo docker build -t namenode:0.1 .
cd ../DataNode
sudo docker build -t datanode:0.1 .
```
Starting the cluster

Run Namenode with specific name as that one set up for Datanodes to point at and bind local path to persist fsimage file for reloads
```
docker run \
	--name namenode \
	-v  "/home/khalid/docker/hadoop_dist.img/NameNode/namenodedata/:/usr/data/hdfs/namenode/" \
	-dit \
	--network="hadoop.net" \
        namenode:0.1 
```

Then run few datanodes (7 to be precise)
```
for i in {1..7}
do  
  docker run --name=datanodes$i -dit --network=hadoop.net datanode:0.1
done
```
To check if all up and running we need to obtain IP address of the Namenode. For that we need to run below docker command and look for our container named "namenode" as per below:
```
khalid@ubuntu:~/docker/hadoop_dist.img$ docker network inspect hadoop.net 
..."Containers": {
"8d63c87c6c971304bc32997523bbeb06c81ebaedef2f988b605f276c77cc0971": {
"Name": "namenode",
"EndpointID": "a26bc5df1fab8d2df150fc7a59bf6e3b8cbf369ad8ad6dfc93e3a66e1a0e51b1",
"MacAddress": "02:42:ac:11:00:02",
"IPv4Address": "172.19.0.2/16",
"IPv6Address": ""
}
},
```
Then you can access NameNode info on that address like so:
```
http://172.19.0.2:50070

```
And Yarn (Resource Manager) on
`
http://172.19.0.2:8088/cluster
`
also query the file system:
```
hdfs dfs -ls hdfs://172.19.0.2:9000
```
Thanks for reading.