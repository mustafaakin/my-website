---
title: Flink Streaming SQL Example
date: 2017-07-02
---

Apache Flink has hit version 1.3 lately and the SQL support has been extended. You can find what is supported from the docs. In this blog post, I will be giving a small example for using Streming SQL a Tumbling Window.

Consider you are streaming room temperatures that are being read from many sensors. You want to find the average tempetarute in windows of 10 seconds and act upon it. You want to do it in SQL. The SQL Query that is supported in Flink looks like the following:

```sql
SELECT 
  room, 
  TUMBLE_END(rowtime, INTERVAL '10' SECOND), 
  AVG(temperature) AS avgTemp 
FROM sensors 
GROUP BY 
  TUMBLE(rowtime, INTERVAL '10' SECOND), 
  room
``` 

After setting up the Flink Execution environment, you need to get your data from a stream, parse and format it to a Tuple or a POJO format, and assign timestamps so that Flink knows how to handle time (Event Time vs Processing Time) and finally execute the SQL and print the results to a stream. In our case we will be printing the results to another stream, but you can pipe the stream to another stream if you wish, filter it, and execute another SQL in it. Possibilities are only limited by your imagination.

You maybe wondering why this Streaming SQL is needed. Surely you could just use regular SQL and for 10 second intervals, you could query the latest 10 seconds data to find the average. However, it would travel the whole data at once, while in streaming SQL, the data is being filtered/aggregated in real-time without actually storing it and the results are also being updated real-time. It can also work in parallel. This might be an interesting and a differentiating use case for your applications. Of course, it is not always the feasible option, for instance if your time window is very large, it might be slowing things down, or requiring more memory than the regular SQL version.

```java
// A data stream, in reality it would be Kafka, Kinesis or other streams
DataStream<String> text = env.socketTextStream("localhost", port, "\n");

// Format the streaming data into a row format
DataStream<Tuple3<String, Double, Time>> dataset = text
        .map(mapFunction)
        .assignTimestampsAndWatermarks(extractor); 

// Register it so we can refer it as 'sensors' in SQL
tableEnv.registerDataStream("sensors", dataset, "room, temperature, creationDate, rowtime.rowtime");

String query = "SELECT room, TUMBLE_END(rowtime, INTERVAL '10' SECOND), AVG(temperature) AS avgTemp FROM sensors GROUP BY TUMBLE(rowtime, INTERVAL '10' SECOND), room";
Table table = tableEnv.sql(query);

// Just for printing purposes, in reality you would need something other than row
// a more formatted object, so that you can chain it through for any other purpose
tableEnv.toAppendStream(table, Row.class).print()
```

As a result of running the program above with proper configuration, you will see the following output:

```
living room  2017–07–02 08:51:40.0    36.7043450149447
kitchen      2017–07–02 08:51:40.0    33.876180051046205
attic        2017–07–02 08:51:40.0    36.16675359462062
outside      2017–07–02 08:51:40.0    31.82492651162825
bedroom      2017–07–02 08:51:40.0    35.57564839912154

kitchen      2017–07–02 08:51:50.0    35.17374622822952
attic        2017–07–02 08:51:50.0    34.70059246571888
bedroom      2017–07–02 08:51:50.0    33.42090378358368
outside      2017–07–02 08:51:50.0    36.28734383828417
living room  2017–07–02 08:51:50.0    35.69019700021718
```

The full example can be found on this [Github Gist](https://gist.github.com/mustafaakin/457859b8bf703c64029071c1139b593d)