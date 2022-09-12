# Setting up Single node HADOOP on docker container

## This Dockerfile builds Hadoop Docker image using Ubuntu image and OpenJDK jvm on single node.

NOTE: I have not included hadoop instalation file as it big (too big for image rebuilds). But you can download it from Apache web site.

Download and save hadoop to the same folder (e.g. hadoop-2.8.2.tar.gz).

I am using 2.8.2 version but feel free to download the other version and don't forget to update Dockerfile respectevely.

For connectivity and web interface access I am using here BRIDGE network. It allows me to access the container and Hadoop web interfaces easily.

You can build image with following command:

``` sudo docker build -t hadoop:1 .```


And start a container:

```docker run -dit --name h1 hadoop:1```

Now Hadoop should be up and running. Lets do some tests and access to the HDFS and create a folder and add some file.

First connect to the container:
```
khalid@ubuntu:~/docker/hadoop.img$ docker attach h1
root@8d63c87c6c97:~#
root@8d63c87c6c97:~# hdfs dfs -ls /
root@8d63c87c6c97:~# hdfs dfs -mkdir /data
root@8d63c87c6c97:~# echo "Hello world" > file.txt
root@8d63c87c6c97:~# hdfs dfs -copyFromLocal file.txt /data/
root@8d63c87c6c97:~# hdfs dfs -ls /
Found 1 items
drwxr-xr-x   - root supergroup          0 2019-12-08 14:01 /data
root@8d63c87c6c97:~# hdfs dfs -ls /data/
Found 1 items
-rw-r--r--   1 root supergroup         12 2019-12-08 14:01 /data/file.txt
root@8d63c87c6c97:~#
```

Now disconnect from the container by pressing Ctrl+P+Q

We are going to access it from external host. For that we will need to find out IP address it's running on, so run below docker command and look for our container named "h1" as per below:
```
khalid@ubuntu:~/docker/hadoop.img$ docker network inspect bridge 
..."Containers": {
"8d63c87c6c971304bc32997523bbeb06c81ebaedef2f988b605f276c77cc0971": {
"Name": "h1",
"EndpointID": "a26bc5df1fab8d2df150fc7a59bf6e3b8cbf369ad8ad6dfc93e3a66e1a0e51b1",
"MacAddress": "02:42:ac:11:00:02",
"IPv4Address": "172.17.0.2/16",
"IPv6Address": ""
}
},
```
Then you can query it again like so:

```
khalid@ubuntu:~/docker/hadoop.img$ hdfs dfs -cat hdfs://172.17.0.2:9000/data/file.txt
Hello world
khalid@ubuntu:~/docker/hadoop.img$
```

Then you can also access NameNode info on

```
http://172.17.0.2:50070
```

And Yarn (Resource Manager) on
```
http://172.17.0.2:8088/cluster
```
