##Change Port
cd apache-druid-24.0.2/conf/druid/single-server/micro-quickstart
pl@plAMD6R micro-quickstart % grep 8081 */*
coordinator-overlord/runtime.properties:druid.plaintextPort=8081


Start

./bin/start-micro-quickstart

bin/supervise -c conf/supervise/single-server/micro-quickstart.conf

##Druid Architure and Dataflow
Segments -> for every data range s segment is created , these are pushed to deep storage

all historical data is written to deep storage (hdfs/s3)

Master Node
Overload Process
Coordinate Process

Data node
Middle Manager Process
Peon Process
Historical Process

Query Node
Broker Process
Router Process

Ingestion ---> overload Process ->   Middle processor -> Peon Process -> Deep Storage -> historical process <->  Process Brokers <- client


Druid Cluster

1. Query
   Locate Data
   Issue Query
   Merge Results
2. Data
   Ingest Data
   Store Data
   Respond to Queries
3. Master
   Issue Tasks
   Catalog data
   Monitor State


##Kafka Integration

https://druid.apache.org/docs/latest/tutorials/tutorial-kafka.html

kafka-server-start.sh -daemon /Users/pl/kafka_2.12-3.3.1/config/server.properties

kafka-topics.sh --create --topic wikipedia --bootstrap-server localhost:9092


kafka-topics.sh --bootstrap-server localhost:9092 --list

kafka-console-producer.sh --broker-list localhost:9092 --topic wikipedia < /Users/pl/apache-druid-24.0.2/quickstart/tutorial/wikiticker-2015-09-12-sampled.json

kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic wikipedia --from-beginning --partition 0

kafka-console-producer.sh --bootstrap-server localhost:9092 --topic wikipedia


 kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic wikipedia --from-beginning --formatter kafka.tools.DefaultMessageFormatter --property print.key=true --property print.value=true --property print.timestamp=true

kafka-topics.sh --bootstrap-server localhost:9092 --topic wikipedia --describe

kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic wikipedia



kafka-topics.sh --create --bootstrap-server localhost:9092  --replication-factor 1 --partitions 1 --topic wikipedia

 kafka-streams-application-reset.sh --application-id wikimedia-druid-ingestion --input-topics wikipedia --bootstrap-servers localhost:9092 --to-offset 0 --force


 kafka-topics.sh --create --topic nifi-topic --bootstrap-server localhost:9092

 kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic nifi-topic --from-beginning


 kafka-streams-application-reset.sh --application-id nifi-kafka-int --input-topics nifi-topic --bootstrap-servers localhost:9092 --to-offset 0 --force




 ##Druid Batch ingestion

 1. Api
 Ingestion specification
 {
   "type":"index_parallel"
   "spec":{
     "dataSchema":{
       "datasource":"wikipedia",
       "timestampSpec":{
         "column":<>
         "format":<>
       },
       "dimensionSpec":{
         "dimensions":[
         "name",
         "address",
         {"type":"long","name":"userid"}
         ]
         },
         "metricspec":[
         {
           type:count,
           name:recordSum
         },
         {
           filedname:added,
           name:addedSum,
           type:longsum
         }
         ],
         "granuralitySpec":{
           "segmentGranularity":"day"
           "queryGranularity":"hour"
           "intervals":[<date-1>/<date-2>]
           "rollup":true

         }
       },
       "ioConfig":{
         "type":"index_parallel",
         "inputSource":{
           "type":"local",
           "baseDir":<>
           "filter":<filename>
         }
         "inputformat":{
           "type":"json"
           },
           "appendtoexisting":false
         },
         "tuningConfig":{}

   }
 }
 type -> to specify batch or streaming
 spec -> this has 3 sections
    dataSchema -> table name and columns
    ioConfig -> source of data
            "type":"index_parallel" --> Batch Processing
            "appendtoexisting":false --> OVERWRITE

    tuningConfig -> create segments
 dataSchema
    datasource->Name of table/data source to be created
    granuralitySpec -> defines time range for segments
        queryGranularity -> defines presion for date for querying , while ingesting date is truncted to this presesion , date with milliseconds are trimmed to day/hour
        interval list -> define range or time period of data to ingest
        segmentGranularity -> time range for segment creation , if day data for all given day is put in single segment
    timestampSpec-> how to interpret time data , time format and column to map


              Data Schema
          /        |          \
Timestamp      Dimension         Metrics (Aggrgation on Measures -> Rollup)
                /      \
            Attributes Measures


Dimension specs -> specs to define columns (Attributes and Measures)

Dimension Measures columns are speical

In Dimension specs if fileds are string no datatype to be given

Matrics --> are aggregate columns like max,min,sum are created during rollup phase , these are created during ingestion so reduces workload during runtime

{
  filedname:added,-->Measure column names
  name:addedSum,
  type:longsum --> aggregation types
}

rollup stage creates group by * , while creating segments
group by on agg_fun on Measures , using all attribute fields(diemsnion fileds and time field), time granuality sepecifed in queryGranularity


dsql -> command line


##Batch Ingestion
curl -X 'POST' -H 'Content-Type:application/json' -d @/Users/pl/DruidLearning/configs/transaction_batch_load.json http://localhost:8081/druid/indexer/v1/task

post-index-task --file /Users/pl/DruidLearning/configs/transaction_batch_load.json --url http://localhost:8081


curl -X 'POST' -H 'Content-Type:application/json' -d @/Users/pl/DruidLearning/configs/transaction_batch_load_metric.json http://localhost:8081/druid/indexer/v1/task

post-index-task --file /Users/pl/DruidLearning/configs/transaction_batch_load_metric.json --url http://localhost:8081

##Stream Ingestion

kafka-topics.sh --bootstrap-server localhost:9092 --list

kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic transaction-topic

kafka-topics.sh --create --topic transaction-topic --bootstrap-server localhost:9092

kafka-console-producer.sh --broker-list localhost:9092 --topic transaction-topic

kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic transaction-topic --from-beginning


curl -X 'POST' -H 'Content-Type:application/json' -d @/Users/pl/DruidLearning/configs/transaction_stream_load.json http://localhost:8081/druid/indexer/v1/supervisor

curl -X 'POST' -H 'Content-Type:application/json' -d @/Users/pl/DruidLearning/configs/transaction_stream_agg.json http://localhost:8081/druid/indexer/v1/supervisor
