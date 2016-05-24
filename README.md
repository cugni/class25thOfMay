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
- [CQL CREATE TABLE](https://docs.datastax.com/en/cql/3.0/cql/cql_reference/create_table_r.html]
- [CQL TYPES](http://docs.datastax.com/en/cql/3.1/cql/cql_reference/cql_data_types_c.html)
- [Twitter Status](https://dev.twitter.com/overview/api/tweets)
- [Twitter Media](https://dev.twitter.com/overview/api/entities#obj-media)


# Quering Cassandra from Python

Requisites: python, pip, virtualenv

We will install the [Python Cassandra driver](https://github.com/datastax/python-driver) and Ipython so that we query Cassandra iteractivly. 

```bash
virtualenv may25
source ~/may25/bin/activate
pip install cassandra-driver
pip install ipython
```



First, we import all the libraries and we connect to the clsuter
```python
# ipython
from cassandra.cluster import Cluster
cluster = Cluster(['localhost'])
session = cluster.connect("tensorsparkkandra")
```

Now the database is empty, so we have to make 1000 inserts. 
We use *session.prepare* for sending the body of the query only once to the server, so we can save some bytes on the network. 

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

In this exercise we will use [ccm](https://github.com/pcmanus/ccm.git), a script which allows creating and managing Cassandra clusters on localhost. 

```bash
pip install ccm
```

Let's create and start a 3 nodes cluster 
```bash 
ccm create test -v 3.5.0 -n 3 -s
```



