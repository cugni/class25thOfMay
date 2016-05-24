# Class of Data managment Layer - Cassandra

## Download Cassandra - 1 node

```bash
wget http://ftp.cixug.es/apache/cassandra/3.5/apache-cassandra-3.5-bin.tar.gz
tar -xf apache-cassandra-3.5-bin.tar.gz
cd apache-cassandra-3.5
bin/cassandra -f  # The -f stands for foreground

```
Open a new terminal

```bash
cd apache-cassandra-3.5
bin/nodetool ring # nodes joined the ring
bin/cqlsh
```
Now we can create our keyspace

```cql

CREATE KEYSPACE tensorsparkkandra WITH replication = {'class': 'SimpleStrategy' , 'replication_factor': 1};
# We use replciation factor one.. as long we have only 1 node. 
```

Now time to create the schema to store the interesting information from the Twitter Statuses and Images

```cql
CREATE TABLE images(
...
```
References:
- [CQL CREATE TABLE](https://docs.datastax.com/en/cql/3.0/cql/cql_reference/create_table_r.html)
- [CQL TYPES](http://docs.datastax.com/en/cql/3.1/cql/cql_reference/cql_data_types_c.html)
- [Twitter Status](https://dev.twitter.com/overview/api/tweets)
- [Twitter Media](https://dev.twitter.com/overview/api/entities#obj-media)


# Querying Cassandra from Python

Requisites: python, pip, virtualenv

We will install the [Python Cassandra driver](https://github.com/datastax/python-driver) and Ipython so that we query Cassandra interactively. 

```bash
virtualenv may25
source ~/may25/bin/activate
pip install cassandra-driver
pip install ipython
```



First, we import all the libraries and we connect to the cluster
```python
# ipython
from cassandra.cluster import Cluster
cluster = Cluster(['localhost'])
session = cluster.connect("tensorsparkkandra")
```

Now the database is empty: let's make 1000 inserts!
Use *session.prepare* for sending the body of the query only once to the server so that we can save some bytes on the network. 

```python
preparedStatement = session.prepare("INSERT INTO images (imgid, confidence , category ) VALUES (?,?,?)")

categories = ['cat','boat','dog','dress','human']

import random
for i in xrange(1000):
  category = categories[random.randint(0,len(categories)-1)]
  confidence = random.random()
  session.execute(preparedStatement.bind([i,confidence,category]))
```

Now we can read all the inserted elements. 
```python 

resultSet = session.execute("SELECT * FROM images")
for row in resultSet:
  print row
  
```


What if we want to filter the results on their confidence score?

```python
  
resultSet = session.execute("SELECT * FROM images WHERE confidence > 0.1") # FAILS: the PRIMARY KEY is missing!

imgids = [ i.imgid for i in session.execute("SELECT imgid FROM images")]

for id in imgids:
  res = session.execute("SELECT * FROM images WHERE imgid = %s AND confidence < %s",(id,0.1))
  for row in res:
     print row

  
```


# Extra - Cassandra multinode

In this exercise, we will use [ccm](https://github.com/pcmanus/ccm.git), a script which allows creating and managing Cassandra clusters on localhost. 

```bash
pip install ccm
wget https://raw.githubusercontent.com/pcmanus/ccm/master/misc/ccm-completion.bash
source ccm-completion.bash  # this is for bash completion - pretty handy 
```

Let's create and start a 3 nodes cluster 
```bash 
ccm create test -v 3.5.0 -n 3 -s
ccm node1 status # nodetool status on nodeone
```
Now we can create again the keyspace, but this time with replication factor 2
```bash
ccm node1 cqlsh
```
```sql
CREATE KEYSPACE tensorsparKkandra WITH replication = {'class': 'SimpleStrategy', 'replication_factor':  2 };
USE tensorsparKkandra;
CREATE TABLE ..
```

By modifying the previous python code you can generate a random workload on the nodes. In the meantime, you can use jconsole to look at the cluster behavior. For instance, you can look into the  MBeans tab at the metric *org.apache.cassandra.metrics.ClientRequest.Read* to see how the load distributes between nodes. One tip: double clicking on the metrics opens a graph of the value changing over the time. 

```bash
ccm jconsole # Starts a jconsole for each node. 
ccm node1 stop # You can stop a node and see what happens to the cluster. 
```

