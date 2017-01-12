# High availability (HA)

## Overview

### Minimum of servers to achieve HA:
- 2 reverse-proxy
- 2 nodejs (esn)
- 2 james
- 2 sabre
- 3 mongo
- 3 Elasticsearch
- 3 cassandra
- 1 redis
- 1 rabbitmq

### Description:
- Cassandra: deploy 1 cluster with 3 nodes
- Elasticsearch: deploy 1 cluster with 3 nodes : 1 master and 2 client (no master and no data) .
- Mongo: deploy 1 cluster with 3 nodes
- Redis and rabbitmq are installed just for esn and sabre
- Jame, esn, and sabre have 2 node per each.
nginx  for load balancing.

## Architecture

## Implement



node1.ha: 149.202.185.166
node2.ha: 213.32.72.41
node3.ha: 213.32.75.95


I. Implement

1. Install java8 on each node

First you need to add webupd8team Java PPA repository in your system. Edit a new ppa file /etc/apt/sources.list.d/java-8-debian.list in text editor

```
sudo vim /etc/apt/sources.list.d/java-8-debian.list
```

And add following content in it:

```
deb http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main
deb-src http://ppa.launchpad.net/webupd8team/java/ubuntu trusty main
```

Now import GPG key on your system for validate packages before installing them.

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys EEA14886
```

Now use the following commands to update apt cache and then install Java 8 on your Debian system.

```
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

At this stage you have successfully installed oracle Java on your Debian system. Let’s use following command to verify installed version of Java on your system.

```
$ java -version

java version "1.8.0_111"
Java(TM) SE Runtime Environment (build 1.8.0_111-b14)
Java HotSpot(TM) 64-Bit Server VM (build 25.111-b14, mixed mode)
```

2. Install elasticsearch 2.2.1

a. Install elasticsearch on each node

Download and install ElasticSearch 2.2.1 deb package from elastic website

```
wget https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-2.2.1.deb
sudo dpkg -i elasticsearch-2.2.1.deb
```

Pin the version to avoid unwanted updates

```
echo "elasticsearch hold" | sudo dpkg --set-selections
```

Create a symlink to your elastic search bin somewhere in your path if this was not done during the previous step (alternatively, you can add /usr/share/elasticsearch/bin/ to your path)

```
sudo ln -s /usr/share/elasticsearch/bin/elasticsearch /usr/bin/elasticsearch
```

Config dir may have not been created (when in a sysV system)

```
sudo ln -s /etc/elasticsearch /usr/share/elasticsearch/config
```

b. Setup a cluster of elasticsearch

Open the Elasticsearch configuration file for editing:

```
sudo vi /etc/elasticsearch/elasticsearch.yml
```

Uncomment and modify following lines:

```
.....

cluster.name: openpaas

.....

node.name: node1

....

bootstrap.mlockall: true

....

network.host: node_ip

....

discovery.zen.ping.unicast.hosts: ["149.202.185.166", "213.32.72.41", "213.32.75.95"]

....

```

Add this lines for node1:

```
node.master: true
node.data: true
```

Add this lines for node2 and node3:


```
node.master: false
node.data: false
```

Save and exit file.


Next, open the /etc/default/elasticsearch file for editing:

```
sudo vi /etc/default/elasticsearch
```

First, find ES_HEAP_SIZE, uncomment it, and set it to about 50% of your available memory. For example, if you have about 4 GB free, you should set this to 2 GB (2g):

```
ES_HEAP_SIZE=2g
```

Next, find and uncomment MAX_LOCKED_MEMORY=unlimited. It should look like this when you're done:

```
MAX_LOCKED_MEMORY=unlimited
```

Save and exit.

Now restart Elasticsearch to put the changes into place:

```
sudo service elasticsearch restart
```

Be sure to repeat this step on all of your Elasticsearch servers.

- Check Cluster State

```
curl -XGET 'http://one-of-nodes-ip:9200/_cluster/state?pretty'
```

You should see output that indicates that a cluster named "production" is running. It should also indicate that all of the nodes you configured are members:

```
{
  "cluster_name" : "openpaas",
  "version" : 38,
  "state_uuid" : "SiM3Z-GBRKOWvNOq-oUYrA",
  "master_node" : "G93FhltDS3az8wFEm6gMug",
  "blocks" : { },
  "nodes" : {
    "G93FhltDS3az8wFEm6gMug" : {
      "name" : "node1",
      "transport_address" : "149.202.185.166:9300",
      "attributes" : {
        "master" : "true"
      }
    },
    "ho6xCJ7TQQm5gUALv_C69Q" : {
      "name" : "node2",
      "transport_address" : "213.32.72.41:9300",
      "attributes" : {
        "data" : "false",
        "master" : "false"
      }
    },
    "-dxAZWjUSBOPCTZyL7Ihkg" : {
      "name" : "node3",
      "transport_address" : "213.32.75.95:9300",
      "attributes" : {
        "data" : "false",
        "master" : "false"
      }
    }
  },
  "metadata" : {
    "cluster_uuid" : "HjfOGe3LTwyuLgELDCeG8Q",
    "templates" : { },
    "indices" : { }
  },
  "routing_table" : {
    "indices" : { }
  },
  "routing_nodes" : {
    "unassigned" : [ ],
    "nodes" : {
      "G93FhltDS3az8wFEm6gMug" : [ ]
    }
  }
}
```

To verify that mlockall is working on all of your Elasticsearch nodes, run this command from any node:

```
curl http://localhost:9200/_nodes/process?pretty
```

Each node should have a line that says "mlockall" : true, which indicates that memory locking is enabled and working:

```
{
  "cluster_name" : "openpaas",
  "nodes" : {
    "G93FhltDS3az8wFEm6gMug" : {
      "name" : "node1",
      "transport_address" : "149.202.185.166:9300",
      "host" : "149.202.185.166",
      "ip" : "149.202.185.166",
      "version" : "2.2.1",
      "build" : "d045fc2",
      "http_address" : "149.202.185.166:9200",
      "attributes" : {
        "master" : "true"
      },
      "process" : {
        "refresh_interval_in_millis" : 1000,
        "id" : 19386,
        "mlockall" : true
      }
    },
    "ho6xCJ7TQQm5gUALv_C69Q" : {
      "name" : "node2",
      "transport_address" : "213.32.72.41:9300",
      "host" : "213.32.72.41",
      "ip" : "213.32.72.41",
      "version" : "2.2.1",
      "build" : "d045fc2",
      "http_address" : "213.32.72.41:9200",
      "attributes" : {
        "data" : "false",
        "master" : "false"
      },
      "process" : {
        "refresh_interval_in_millis" : 1000,
        "id" : 18731,
        "mlockall" : true
      }
    },
    "-dxAZWjUSBOPCTZyL7Ihkg" : {
      "name" : "node3",
      "transport_address" : "213.32.75.95:9300",
      "host" : "213.32.75.95",
      "ip" : "213.32.75.95",
      "version" : "2.2.1",
      "build" : "d045fc2",
      "http_address" : "213.32.75.95:9200",
      "attributes" : {
        "data" : "false",
        "master" : "false"
      },
      "process" : {
        "refresh_interval_in_millis" : 1000,
        "id" : 18694,
        "mlockall" : true
      }
    }
  }
}
```

If mlockall is false for any of your nodes, review the node's settings and restart Elasticsearch. A common reason for Elasticsearch failing to start is that ES_HEAP_SIZE is set too high.

2. Install mongo

a. Install mongo on each node

Go to https://docs.mongodb.com/manual/tutorial/install-mongodb-on-debian/

b. Setup replicate set

Stop mongod service on each node

```
sudo service mongod stop
```

Run mongod service backgroud  with params:

```
nohup mongod --dbpath /srv/mongodb/ --replSet rs0 --smallfiles --oplogSize 128 > /var/log/mongodb/mongod.log 2>&1 &
```

In node which is prmary node:

Start mongo shell:

```
mongo
```

In the mongo shell, use rs.initiate() to initiate the replica set.

```
rs.initiate(
  {
    _id: "rs0",
    members: [
    {
     _id: 0,
     host: "primary_node_ip:27017"
    }
    ]
  }
);
```

Display the current replica configuration by issuing the following command:

```
rs.conf()
```

In the mongo shell connected to the primary, add the second and third mongod instances to the replica set using the rs.add() method.

```
rs.add("secondary_node_ip:27017");
rs.add("secondary_node_ip:27017");
```

3. Install redis and rabbitmq on node1

a. Install redis
- Go to [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-redis) to install redis servers
- Config redis to allow connecting from remote servers
 - Open redis configuration file.

 ```
 sudo nano /etc/redis/6379.conf
 ```
 - Locate this line and make sure it is uncommented (remove the `#` if it exists) and change:

 ```
 bind 0.0.0.0
 ```

b. Install rabbitmq

- Go to [here](https://www.rabbitmq.com/download.html) to install rabbitmq
- Config rabbitmq to allow remote server can connect.
  - Create configuration file of rabbitmq

  ```
  sudo vim /etc/rabbitmq/rabbitmq.config
  ```
  - With content:

  ```
  [{rabbit, [{loopback_users, []}]}].
  ```

  Save and exit.

  Exports environment variable to tell rabbitmq where is configuration file:

  ```
  export RABBITMQ_CONFIG_FILE="/etc/rabbitmq/rabbit"
  ```

  - Restart rabbitmq:

  ```
  sudo service rabbitmq-server Restart
  ```


4. Install rse

a. For debian, may be have error while install canvas.
Fix problem by issuing the following command:

```
wget http://ftp.us.debian.org/debian/pool/main/libj/libjpeg8/libjpeg8-dev_8d1-2_amd64.deb

sudo dpkg -i libjpeg8-dev_8d1-2_amd64.deb

wget http://ftp.us.debian.org/debian/pool/main/libj/libjpeg8/libjpeg8_8d-1+deb7u1_amd64.deb

sudo dpkg -i libjpeg8_8d-1+deb7u1_amd64.deb
```

b. Connect to mongo replica set

```
cd rse/

node ./bin/cli db --host one_of_mongo_node_ip --port 27017 --database esn
```

Add options to esn connecting to mongo replica set

```
vi config/db.json
```

And change value of field "connectionString" to:

```
mongodb://213.32.72.41:27017/esn?replicaSet=rs0
```

Save and exit.

5. nginx

a. Install

```
sudo apt-get install nginx
```

b. configuration
