# wikiwiki

[![Git](https://app.soluble.cloud/api/v1/public/badges/f7e515e1-ba98-44a4-b4a7-ba5b987825aa.svg?orgId=679096383598)](https://app.soluble.cloud/repos/details/github.com/desteves/wikiwiki?orgId=679096383598)  

Confluent Cloud Kafka + MongoDB Atlas. Source + Sink Connector Demos.
We'll be using  Wikipedia change edits for both demos.


___WORK IN PROGRESS__ __NEEDS TESTING__

___WORK IN PROGRESS__ __NEEDS TESTING__

___WORK IN PROGRESS__ __NEEDS TESTING__

___WORK IN PROGRESS__ __NEEDS TESTING__

___WORK IN PROGRESS__ __NEEDS TESTING__


## MongoDB as the sink

Sends Wikipedia page edits to a Kafka Producer -> Persist messages to MongoDB collection.

## MongoDB as the source

Sends Wikipedia page edits to a MongoDB collection -> Connector picks these up and sends them to a Kafka Topic.

## Prep

### Set up "free" Confluent Cloud Kafka Cluster

- Pre-reqs
  - Register [here](https://confluent.cloud/signup)

### Via the `ccloud` cli for CentOS

```bash

# Install ccloud
curl -L --http1.1 https://cnfl.io/ccloud-cli | sudo sh -s -- -b /usr/local/bin
echo $PATH | grep /usr/local/bin
sudo rpm --import http://packages.confluent.io/rpm/3.1/archive.key
sudo vi /etc/yum.repos.d/confluent.repo
[Confluent.dist]
name=Confluent repository (dist)
baseurl=http://packages.confluent.io/rpm/3.1/7
gpgcheck=1
gpgkey=http://packages.confluent.io/rpm/3.1/archive.key
enabled=1

[Confluent]
name=Confluent repository
baseurl=http://packages.confluent.io/rpm/3.1
gpgcheck=1
gpgkey=http://packages.confluent.io/rpm/3.1/archive.key
enabled=1

sudo yum clean all
sudo yum install -y confluent-platform-oss-2.11


# Log in to ccloud and view connectors
ccloud login
ccloud connector-catalog list
ccloud connector list
ccloud connector-catalog describe MongoDbAtlasSink 

# Create cluster
ccloud kafka cluster create demoKafkaCluster --cloud aws --region us-east-2 --availability single-zone --type basic  --output  json -vvvv


ccloud kafka cluster list
#//  Id        |       Name       | Type  | Provider |  Region   | Availability | Sta     
#//* lkc-kk02p | demoKafkaCluster | BASIC | aws      | us-east-2 | LOW          | UP 
#//   ^^^^^^
#//   ||||||
#// repalce with YOUR cluter `Id`
export CLUSTERID=lkc-kk02p
ccloud kafka cluster describe ${CLUSTERID}


# Create API Key
ccloud api-key create --resource ${CLUSTERID} --description "For demo, delete afterwards" --output json -vvvv
#// {
#//   "key": "ABC123ABC123",
#//   "secret": "123abc123abc123abc123abc123abc123abc"
#// }
ccloud api-key list 

```

### Set up MongoDB Atlas

- Pre-reqs
  - Register [here](https://account.mongodb.com/account/register)
  - Generate API Key [follow these steps](https://docs.atlas.mongodb.com/configure-api-access/)

#### Via `mongocli`

```bash

# Create new project


# Deploy a free Atlas Cluster


# Add database access


# Add network access


# Test connectivity
mongo "mongodb+srv://demoatlascluster.bwkgk.mongodb.net/wiki" --username wikiUser`
```

### Set up Sink Connector

```bash

# Configure
ccloud connector-catalog describe MongoDbAtlasSink --output json > sink.json
vi sink.json

{
  "connection.host": "demoatlascluster.bwkgk.mongodb.net",
  "connection.password": "******",
  "connection.user": "wikiUser",
  "connector.class": "MongoDbAtlasSink",
  "database": "wiki",
  "input.data.format": "JSON",
  "kafka.api.key": "**********",
  "kafka.api.secret": "****************",
  "name": "sink",
  "tasks.max": "1",
  "topics": "wikiRecentChanges"
}

# Create
ccloud connector create --cluster ${CLUSTERID} --config sink.json --output json -vvvv
```

#### Test Sink Connector

```bash

# Install kafka producer
# TODO
# TODO
# TODO
# TODO


# Test producer
curl -s  https://stream.wikimedia.org/v2/stream/recentchange |   grep data |  sed 's/^data: //g' |  jq -rc 'with_entries(if .key == "$schema" then .key = "schema" else . end)' | kafka-console-producer --broker-list pkc-ep9mm.us-east-2.aws.confluent.cloud:9092 --producer.config config.properties --topic wikiRecentChanges
```

### Set up Source Connector

```bash

# Configure
ccloud connector-catalog describe MongoDbAtlasSource --output json > source.json
vi source.json
{
  "connector.class": "MongoDbAtlasSource",
  "name": "demoAtlasSourceConnector",
  "kafka.api.key": "****",
  "kafka.api.secret": "****",
  "topic.prefix": "mdb",
  "connection.host": "demoatlascluster.bwkgk.mongodb.net",
  "connection.user": "wikiUser",
  "connection.password": "****",
  "database": "wiki",
  "collection": "recentChangesSource",
  "poll.await.time.ms": "5000",
  "poll.max.batch.size": "1000",
  "copy.existing": "false",
  "tasks.max": "1"
}


# Create
ccloud connector create --cluster ${CLUSTERID} --config source.json --output json -vvvv

```

#### Test Source Connector

```bash
# Create Consumer
ccloud kafka topic consume -b  mdb.wiki.recentChangesSource --group sauceGroup


# Generate Events
curl -s  https://stream.wikimedia.org/v2/stream/recentchange | grep data |  sed 's/^data: //g' |  jq 'with_entries(if .key == "$schema" then .key = "schema" else . end)' | mongoimport --quiet --host atlas-nn787r-shard-0/demoatlascluster-shard-00-00.bwkgk.mongodb.net:27017,demoatlascluster-shard-00-01.bwkgk.mongodb.net:27017,demoatlascluster-shard-00-02.bwkgk.mongodb.net:27017 --ssl --username wikiUser --password YOURPASSWORDK --authenticationDatabase admin --db wiki --collection recentChangesSource --type json
```
