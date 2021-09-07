---
title: How arbitrary structured data per request helps Opsgenie to resolve and prevent incidents
date: 2021-09-07
--- 

It's been a long time since I've blogged. My backlog of topics is full, however, I could not publish due to both professional and personal developments in the last 1.5 years. Anyways, glad to be back.

For some context, I'm from [Opsgenie](https://www.atlassian.com/software/opsgenie). Since Opsgenie is an on-call and alert management tool, it needs to be highly available than most of the services. We take it very seriously. Over the years, we have employed and evolved many engineering practices and principles to consistently achieve this. One of them is how we monitor our running services and our dependencies.

When talking about monitoring in the software world, mostly tools like Prometheus, Nagios, Cloudwatch, NewRelic, Datadog, Honeycomb come into mind. We generalize monitoring as a practice in which you ask questions to a system and it gives answers, without actually live-debugging running code or systems.  That's why metrics, logs, and traces are exported to 3rd party systems, so you can ask your questions. In other words, you observe the system from outside, whatever it is.

However, although there are a variety of tools and vast data, some of the questions might be hard to answer. Considering the three pillars of observability, and various views [Datadog](https://www.datadoghq.com/three-pillars-of-observability/) or [Honeycomb](https://www.honeycomb.io/blog/they-arent-pillars-theyre-lenses/) on them, they are mostly categorized as metrics, traces, and logs. For each, I'll briefly talk about what's good, wrong about them, and how we find a middle way.

- **Metrics:** Apart from infrastructure provided (AWS resources) ones, we find them mostly uninformative. The reason is that, by nature, they are aggregated, lack detail, and most tools that handle them cannot handle cardinality at all. Have fun running daily 10k Kubernetes Pods and optimizing your Prometheus to not crash and negotiate with SaaS providers. Metrics are useful for fast detection and ease of writing alarms to notify on-call. Another useful thing is we can observe the trends if we are suspicious of a longer behavior because it's cheap to store them in longer retention. However, 90% of our queries hit 1-hour of data, 95% 1-day.

- **Logs:** They are way worse than metrics because they are unstructured and much more expensive to store.  The only additional detail in each log is the log level, timestamp, trace-id, service-name, and instance-id. This data only helps to identify logs; the message is free-form and requires parsing. Logs are only helpful in obscure support cases, where we hopefully put something as a log. However, if something unexpected happens, i.e a non-discardable, exception occurs (which is not in an ignore-list), we create an alert that is directed to the team who owns the service.
- **Traces:** Employing more than 70 microservices, and co-existing in a huge enterprise, traces can be helpful to visualize flows. Rarely, we identify some issues by looking at them individually to discover dependencies and which one had problems (latency, call count, etc.). But that's it. None of the toolings allows advanced filtering, i.e show me traces that are made by enterprise customers and that made >100 AWS calls. The data is there, but it's not flexible enough to query. For a tracing system to be useful, it must have an SQL capable engine, or something else advanced filtering which can also do aggregations or subqueries. But it would probably cost thousands of dollars to do it in real-time with dynamic data. (i.e tags). I once wrote a sink for Jaeger to a TimescaleDB cluster to play around with, it had some promise, but although I love SQL, I needed to level up my SQL skills.

## If you don't like metrics, logs, and traces, how is Opsgenie doing the monitoring?

My points above might seem snarky to some, but they are derived from real experience. Instead of those 3, we are publishing arbitrarily-wide structured data-blobs per request as [Charity Majors](https://twitter.com/mipsytipsy/status/1434985888920924160) advocates for, or in other words long key-value data per request, or what we prefer to call transaction, which can also mean event processing (SQS). In Opsgenie, we prefer minimal transactions and async flows. 

For instance, when a create alert request is received, after the authentication and verification, it's passed to an SQS queue for another service to pick up. And if this alert is going for multiple escalations, schedules, notifications they are all passed around as queue messages, so that they can be scaled very easily and can be retried without hesitation. After an alert is received, it's around 350ms to result in a notification to customers. We track [this metric](https://opsgenie.status.atlassian.com/) publicly. 

So, with many sync and async transactions, it's imperative to monitor them easily. In each transaction (request or event processing), we collect some values in the transaction context. The data in our transactions look a lot like spans, with around 100 unique values per request. In all of the transactions, we have around 2000 different keys. This data puts identifying information like IP address, user agent, customer-id, user-id but also puts rich data as customer plan. For each transaction, developers are free to put the data they want, but we review them to ensure no PII/UGC data ends up accidentally. 

Differentiating part is that all services use in-house libraries to interact with each other or external systems such as AWS DynamoDB, SQS. As those libraries are aware of the transaction context, they can put more data. To be more clear, when a transaction makes a read request to a resource, we automatically keep aggregated track of duration and counts per resource, i.e per table, per queue, per service. This looks like child-spans, but it's aggregated data. But having this data denormalized in the transaction context, allows us to query that data very fast. Engineers use [NewRelic Insights](https://newrelic.com/products/insights) to query this data on-demand, and automated monitors use an optimized internal Elasticsearch cluster. Insights have been very helpful to filter and aggregate data regardless of cardinality, with its SQL-like language NRQL. A similar thing can be achieved with Elasticsearch too, however, aggregations are huge JSONs, and its SQL-to-JSON translator does not work well for our custom index type mapping, and Insights have great auto-complete for keys and values. We also exported [Kubernetes events](https://mustafaakin.dev/posts/2019-12-12-how-to-export-kubernetes-events-for-observability-and-alerting/) (some-what structured events) to NR Insights and ability to query them was helpful.

In short, our data looks like the following: 

```json
{
  "traceID": "<id>",
  "name": "SomeTransaction",
  "timestamp": "2021-09-07 15:25:46",
  "service": "alert-service",
  "deployDate": "2021-09-03 10:00:02",
  "gitHash": "aecdjg",
  "ip": "1.2.3.4",
  "country": "TR",
  "useragent": "Go-SDK 1.45",
  "instanceID": "i-1245",
  "zone": "us-west-2b",
  "instanceType": "c5.xlarge",
  "customerId": "<uuid>",
  "userId": "<uuid>",
  "customerPlan": "enterprise",
  "customerUsers": 135,
  "integration": "Cloudwatch",
  "aws.dynamodb.total.count": 5,
  "aws.dynamodb.total.duration": 34,
  "aws.dynamodb.alert.count": 1,
  "aws.dynamodb.alert.duration": 8,
  "aws.dynamodb.user.count": 1,
  "aws.dynamodb.user.duration": 8,
  "cache.user.count": 3,  
  "cache.user.duration": 2,
  "service.x.count":1,
  "service.x.duration": 10,
  "rate.limit": false,
  "http.response.code": 200,
  "duration": 93
}
```

With this data present, we can ask most of the questions that would be useful to us for monitoring purposes, even during the incidents. For instance, finding out if any transactions have excessive usage of DynamoDB tables is a simple group by operation. With this data, we can slice & dice (filter) to find out whether the problems are isolated to an endpoint, certain customers, an integration, a group of customers, an internal service, availability zones, instance types, etc. by filtering and aggregating it. Since those queries require no joins, many data stores can answer these types of queries very fast. And since we did not aggregate anything, you can decide to look for a min, max, or p99 on-demand, without pre-computing them and thinking which one would be most useful. 

What we have above can be achieved similarly by using tracing, putting the data in tags, and use them to query. We can do it either by denormalizing the data as it arrives, or use a capable SQL engine on top of it. 

Since at Opsgenie, we use minimal transactions, and the key values are meaningful inside a span only, denormalizing the external service interaction stats was the most effective choice for us. Also, if needed, it has the traceId linked so that we can jump to the tracing tool, considering it was not sampled due to cost concerns.

## Please don't stare at dashboards, automate it

With the above data published to your capable data storage, you can easily construct dashboards. I'm not a fan of dashboards. Reasons will come later. However, there can be somewhat useful dashboards, if you find yourself repeating those queries during incident investigations. For instance, seeing the trend of DynamoDB latencies among availability zones, or notification failures to certain devices, countries, customers can be useful for us. 

However, there is no point in getting 50+ inch screens to offices and additional monitors from your WFH budget and staring at them. Seriously. Stop it. I agree it's sometimes easier for a human to see a correlation, trend than to write an alarm in code, but please just try to automate it in any way you can. It's a waste of precious engineering time that can be spent on other things and a waste of electricity for the TV. In Opsgenie, there are thousands of alarms and around a hundred complex monitors that query the above key-value data and notifies the on-call. 

Those alarms have context on them and we still use the NewRelic Insights to drill down to further investigate demand. Main reason we use NewRelic Insights is that its NRQL language (familiarity with SQL) with superb auto-complete, it can complete both keys and values very fast, and helps you immensely during indident investigations you don't need to select anything from dropdown, it feels like talking. It looks like following:

```sql
SELECT count(1), avg(duration), percentile(aws.dynamodb.total.duration, 95) FROM Transaction
WHERE customerPlan IN ('lite', 'free') AND name LIKE 'Alert%'
FACET customerId
SINCE 30 minutes ago
```

It does not matter how many different monitors we have, it's always the one you did not prepare for, because the ones you know have already automated solutions to mitigate impacts. We take lessons on previous incidents and ask what additional data we could have exposed our TTD (time-to-detection) would be under a minute and TTR (time-to-resolve) would be shortened significantly. This practice always yields in additional values and automated monitors and we can sleep well at night, knowing we would be aware of problems before they get large and affect customers and they notify us.