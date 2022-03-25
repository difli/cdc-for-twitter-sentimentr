# cdc-for-twitter-sentimentr

DataStax CDC for Apache Cassandra® is open-source software (OSS) that sends Cassandra mutations for tables having Change Data Capture (CDC) enabled to Luna Streaming or Apache Pulsar™, which in turn can write the data to any technology like for example platforms such as Elasticsearch®.

Here are installation steps for CDC with elasticsearch sink. This is based on [twitter-sentimentr](https://github.com/difli/twitter-sentimentr).
![alt text](/images/sentimentr.png)
The twitter-sentiment application inserts data into cassandra. CDC ensures that all twitter.tweet_by_id mutations are continuously streamed to elasticserch in order to leverage the data in realtime with the capabilities of elasticsearch.
![alt text](/images/elasticsearch.png)

The described steps presume you have followed [twitter-sentimentr](https://github.com/difli/twitter-sentimentr) and have now a local installation of apache pulsar, apache cassandra and the twitter-sentimentr demo application up and running.

# Run CDC with a local pulsar and cassandra instance
- follow the DataStax Change Agent for Apache Cassandra (CDC) [installation guide](https://docs.datastax.com/en/cdc-for-cassandra/cdc-apache-cassandra/1.0.2/index.html)
- [download](https://downloads.datastax.com/#cassandra-change-agent) the change agent
- export the JVM_EXTRA_OPT environment variable for apache cassandra
```
export JVM_EXTRA_OPTS="-javaagent:/Users/dieter.flick/Documents/techstuff/cdc/cassandra-source-agents-1.0.1/agent-c4-pulsar-1.0.1-all.jar=pulsarServiceUrl=pulsar://localhost:6650"
```
- set the following properties in the cassandra.yml file
```
vi /Users/dieter.flick/Documents/techstuff/cassandra/apache-cassandra-4.0.3-cdc/conf/cassandra.yaml
```
```
  cdc_enabled: true
  commitlog_sync_period_in_ms: 2000
  cdc_total_space_in_mb: 50000
```
- activate CDC for the twitter.tweet_by_id table
```
ALTER TABLE twitter.tweet_by_id WITH cdc=true;
```
- follow the [DataStax Cassandra Source Connector for Apache Pulsar (CSC)  installation guide](https://docs.datastax.com/en/cdc-for-cassandra/cdc-apache-cassandra/1.0.2/install.html#_download_csc_for_pulsar)
- [download](https://downloads.datastax.com/#cassandra-source-connector) the cassandra source connector
- copy the cassandra source connector to the connectors folder in the pulsar base folder
```
cp pulsar-cassandra-source-1.0.1.nar /Users/dieter.flick/Documents/techstuff/pulsar/apache-pulsar-2.8.2/connectors
```
- reload the sources to ensure pulsar is aware of the just copied cassandra source connector
```
bin/pulsar-admin sources reload
bin/pulsar-admin sources available-sources
bin/pulsar-admin sources list
```
- create a cassandra source connector instance in pulsar

```
bin/pulsar-admin source create \
--name cassandra-source-tweet_by_id \
--archive /Users/dieter.flick/Documents/techstuff/pulsar/apache-pulsar-2.8.2/connectors/pulsar-cassandra-source-1.0.1.nar \
--tenant public \
--namespace default \
--destination-topic-name public/default/data-twitter.tweet_by_id \
--parallelism 1 \
--source-config '{
    "events.topic": "persistent://public/default/events-twitter.tweet_by_id",
    "keyspace": "twitter",
    "table": "tweet_by_id",
    "contactPoints": "localhost",
    "port": 9042,
    "loadBalancing.localDc": "datacenter1",
    "auth.provider": "PLAIN",
    "auth.username": "cassandra",
    "auth.password": "cassandra"
}'
```
- check the status of the cassandra source connector instance
```
bin/pulsar-admin source status --name cassandra-source-tweet_by_id
```
- you could in addition check CDC operates already by consuming the topic created by the cassandra source connector
```
sh -c "bin/pulsar-client consume public/default/data-twitter.tweet_by_id -s tweet-data -n 60 -r 1"
```
- in case you want delete the cassandra source connector: bin/pulsar-admin source delete --name cassandra-source-tweet_by_id
- follow the steps in the section [Start Elasticsearch and Kibana in Docker](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-stack-docker.html#run-docker-secure)
- INTERACTION with with elasticsearch and kibana only via https
- copy the credentials from elasticsearch container console output:

```
->  Password for the elastic user (reset with `bin/elasticsearch-reset-password -u elastic`):
  6-e+Yg9tUJG_BsSh44R=

->  HTTP CA certificate SHA-256 fingerprint:
  b727a55fbf9d3bac5f059aad0218e3220ebcd973573983c677c7e79c72571ad8

->  Configure Kibana to use this cluster:
* Run Kibana and click the configuration link in the terminal when Kibana starts.
* Copy the following enrollment token and paste it into Kibana in your browser (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjEuMCIsImFkciI6WyIxNzIuMTkuMC4yOjkyMDAiXSwiZmdyIjoiYjcyN2E1NWZiZjlkM2JhYzVmMDU5YWFkMDIxOGUzMjIwZWJjZDk3MzU3Mzk4M2M2NzdjN2U3OWM3MjU3MWFkOCIsImtleSI6IjVLNndjbjhCdjNYVlp2SW53eWZSOnRmampwSVd5VEhHZXBsc2pybEZWancifQ==

-> Configure other nodes to join this cluster:
* Copy the following enrollment token and start new Elasticsearch nodes with `bin/elasticsearch --enrollment-token <token>` (valid for the next 30 minutes):
  eyJ2ZXIiOiI4LjEuMCIsImFkciI6WyIxNzIuMTkuMC4yOjkyMDAiXSwiZmdyIjoiYjcyN2E1NWZiZjlkM2JhYzVmMDU5YWFkMDIxOGUzMjIwZWJjZDk3MzU3Mzk4M2M2NzdjN2U3OWM3MjU3MWFkOCIsImtleSI6IjVhNndjbjhCdjNYVlp2SW53eWZSOk12UmJsQmZJU3NPdUV3Vlk1RkdTRGcifQ==
```
- get the https cert in order to allow the pulsar elasticsearch sink to ingest data into elasticsearch
```
docker exec -it es01 /bin/bash -c "find /usr/share/elasticsearch -name http_ca.crt"
/usr/share/elasticsearch/config/certs/http_ca.crt
```
- create a “tweets” index in elasticsearch
```
curl -k -X PUT "https://localhost:9200/tweets" -H 'Content-Type: application/json' -d'{ "settings" : { "index" : { } }}' -u elastic:'6-e+Yg9tUJG_BsSh44R='
```
- in case you want to delete an index: curl -k -X DELETE "https://localhost:9200/tweets" -u elastic:'6-e+Yg9tUJG_BsSh44R='
- pulsar elasticsearch sink installation
- here is the [elasticsearch sink documentation](https://pulsar.apache.org/docs/en/io-elasticsearch-sink/)
- [download](https://www.apache.org/dyn/mirrors/mirrors.cgi?action=download&filename=pulsar/pulsar-2.9.1/connectors/pulsar-io-elastic-search-2.9.1.nar) the elasticsearch sink
- copy elasticsearch sink to the connectors folder in the pulsar base folder
```
cp pulsar-io-elastic-search-2.9.1.nar
/Users/dieter.flick/Documents/techstuff/pulsar/apache-pulsar-2.8.2/connectors
```
- reload the sinks to ensure pulsar is aware of the just copied elasticsearch sink
```
bin/pulsar-admin sinks reload
bin/pulsar-admin sinks available-sinks
```
- create a truststore with the https cert from elasticsearch.
```
keytool -importcert -alias elastic -keystore truststore.jks -storepass elastic -file http_ca.crt
```
- create a config file for the elasticsearch connector
```
dieter.flick@dflick-rmbp16 elasticsearch % cat elastic-sink-conf.yml
configs:
  elasticSearchUrl: "https://localhost:9200"
  indexName: "tweets"
#  createIndexIfNeeded: true
  username: "elastic"
  password: "6-e+Yg9tUJG_BsSh44R="
  schemaEnable: true
  ssl:
    enabled: true
    truststorePath: "/Users/dieter.flick/Documents/techstuff/elasticsearch/truststore.jks"
    truststorePassword: "elastic"
    hostnameVerification: false
```
- for first test run the elastic search sink local
```
$ bin/pulsar-admin sinks localrun \
    --archive connectors/pulsar-io-elastic-search-2.9.1.nar \
    --tenant public \
    --namespace default \
    --name elasticsearch-test-sink \
    --sink-config-file /Users/dieter.flick/Documents/techstuff/elasticsearch/elastic-sink-conf.yml \
    --inputs data-twitter.tweet_by_id
```
- when that works create a elastic search sink in pulsar
```
bin/pulsar-admin sinks create \
    --sink-type elastic_search \
    --tenant public \
    --namespace default \
    --name elasticsearch-sink \
    --sink-config-file /Users/dieter.flick/Documents/techstuff/elasticsearch/elastic-sink-conf.yml \
    --inputs data-twitter.tweet_by_id
```
- check the status of the just created elasticsearch sink
```
bin/pulsar-admin sink list
bin/pulsar-admin sink status --name elasticsearch-sink
```
- you can find logs for the sinks and sources in your pulsar installation logs folder:
![alt text](/images/logs.png)
- in case you want to delete the sink: bin/pulsar-admin sink delete --name elasticsearch-sink
- test the data pipeline works and ingest data in cassandra and see if it appears in elasticsearch
```
cqlsh:twitter> insert into twitter.tweet_by_id (lang,createdat,id,sentiment,tweet) values ('de','xyz','9',3,'test4');
```
- See if you get something in elastic search
```
curl -v -k 'https://localhost:9200/tweets/_refresh' -u elastic:'6-e+Yg9tUJG_BsSh44R='
curl -v -k 'https://localhost:9200/tweets/_search' -u elastic:'6-e+Yg9tUJG_BsSh44R='
```
- or better use [kibana](http://localhost:5601/) - Discover and hit refresh from time to time
![alt text](/images/discover.png)
- import via saved objects management the [tweets dashboard](dashboard) into kibana   
- no you should be able to watch your Tweets dashboard
![alt text](/images/elasticsearch.png)
# Run CDC powered by Astra
- NO HASSLE with any configuration!
- Just enable CDC on the table you want to stream data from Astra DB to any destination for leveraging your data in realtime.
![alt text](/images/cdc-enable.png)
- wait until it is initalized
![alt text](/images/cdc-initializing.png)
 - than see your data stream as it gets puplished to the data topic that was automatically created for your
 ![alt text](/images/cdc-data-topic.png)
 - let"s create a elastic search sink
 ![alt text](/images/cdc-elastic-1.png)
 - you could choose from various connectors (sinks) in order to stream your realtimedate
 ![alt text](/images/cdc-connectors.png)
 - elasticsearch sink details
 ![alt text](/images/cdc-elastic-2.png)
