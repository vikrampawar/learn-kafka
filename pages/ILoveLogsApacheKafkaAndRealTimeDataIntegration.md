> [Home](Home.md)

# I ❤ Logs: Apache Kafka and Real-time Data Integration

## Status : DONE

## Course info

- [Link to O'Reilly video course](https://learning.oreilly.com/videos/i-logs/9781491908310/)
- Author : Jay Kreps
- Year : 2014

## What is Data Integration ?

Data integration and ETL is boring.

Working at LinkedIn.

Get data into good data processing systems. Both real-time and batch (Hadoop).

Maslows's hierarchy of needs.

```mermaid


flowchart  LR

    subgraph 1
        A[Automation]

    end
    subgraph 2
        U[Understanding]

    end
    subgraph 3
        S[Semantics]

    end

    subgraph 4
        AC[Acquisition Collection]
    end

    4 --> 3
    3 --> 2
    2 --> 1

```

Data infrastructures like Hadoop cluster is only as good as the data it gets. FTP data around.

Why is this a hard problem?

1. Type of data has changed in the last 10 - 15 years. Rather than transactional data, you now have event data - user activity data.
2. Specialise databases rather than general purpose relational databases. For example key value or monitoring, search etc.

How easy is it to get this data?

Started with Hadoop at LinkedIn.
Did not have data in that cluster.
More time spent on getting data into the cluster than doing analytics.

Data becomes a basis of everything. It can lead to something unexpected.

Custom capturing for several different sources, this is repeated, built in an adhoc manner. This leads to N2 pipeline.

Verify data before capture as it remains forever.

Centralised approach to data capture based around a datatime log.

Large data integration problem as related to logs.

## What is Apache Kafka ?

Kafka is a messaging system.

```mermaid
flowchart LR
subgraph Apache Kafka
    direction TB
    subgraph top
        direction LR
        A[Producer A]
        B[Producer B]
        C[Producer C]
    end
    subgraph middle
        direction LR
        ESP[Kafka cluster]
    end

    subgraph bottom
        direction LR
        D[Consumer D]
        E[Consumer E]
        F[Consumer F]
    end

    A --> |Events| ESP
    B --> |Events| ESP
    C --> |Events| ESP

    ESP --> |Events| D
    ESP --> |Events| E
    ESP --> |Events| F


end

```

These are real-time.
Each stage is distributed.
Implementation is architectural different from traditional messaging systems.

Amazon Kinesis is very similar to Kafka and possibly inspired by it.

First project was DynamoDB derivation, but now Amazon has cloned Kinesis. There are now two systems!

Both are very similar. Amazon Kinesis is hosted by Amazon whereas Kafka you have to host yourself.

Can this be solved in Infrastructure?

Each pipleline solved a different problem.

- database copied was different to ETL data.
- Active MQ ingestion was different.

A system was required to solve all these problems.

Tried using Active MQ, it didn't work as it was not build for high-throughput, logging or event data like page view events. Data not persistent for longer periods for example if Hadoop cluster is down. It would need something like 300 active brokers!! 300 logging brokers for 300 machines is not tenable!

ETL was done with files, messaging with messages.
There was a huge difference between the two.

Came up with

### 3 design principles:

**1. One pipeline to rule them all.**

**2. Stream processing >> messaging.**

Message Brokers hold on to data and do not do any processing on it.

Hadoopp is doing something higer level, in addition to HDFS, it allows map reduce and other processing.

One of the key characteristics should allow stream processing in a rich way.

**3. Clusters not servers**

True of HDFS. Processing of files across machines. Not true of individual message brokers!! Queues and topics are on a particular machine.

### Characteristics

**Scalability of the file system**

- Hundreds of MB/sec/server throughput. (Logging/ log copying)
- Many TB per server.

**Gaurantees of database**

- Messages are strictly ordered
- All data is persitent
  **Distributed by default**
- Replication -> individual machines can fail without losing data
- Partitioning model -> scale by adding more machines

Built and used heavily at LinkedIn.

### Kafka at LinkedIn

- 175 TB of in-flight data.
- Low-latency: ~ 1.5 ms
- Replicated to each data center
- Tens of thousands of data producers
- Thousands of data consumers
- 7 million messages written per second
- 35 million messages read per second ( loading of data happens automatically - versus a team doing it)

See

- [Latency numbers every programmar should now](https://gist.github.com/hellerbarde/2843375)
- [Low latency ](http://shorttermmemoryloss.com/nor/2015/01/04/low-latency/)
- [Latency Numbers Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html)

## Logs and Distributed Systems

Different way of thinking about data.
Database provides tables, Kafka provides logs.

Particular thing in mind when thinking about logs. Not the server logs.

- It's append only as a regular log.
- Structured array of feed of messages
  - It's ordered by time, newer writes happen at the end.
  - It's immutable -> records don't change after they are written.
  - each record can be denoted by a unique number (A log sequence number or an offset in Kafka terms)
  - Ignore the format of messages, can think of it like a log line.

```mermaid

flowchart TD
    F[First Record]
    0
    1
    2
    3
    4
    L[Next record when written]

    subgraph Log
        F --> 0
        N1[Next] --> 1
        N2[Next] --> 2
        N3[Next] --> 3
        L --> 4
    end

```

Provides a data. A log centric systems.

If you partition it, you get the Kafka data model.

```mermaid
flowchart
        FA[First Record]
        0A[0]
        1A[1]
        2A[2]
        3A[3]
        LA[Next record when written]


        FB[First Record]
        0B[0]
        1B[1]
        2B[2]
        3B[3]
        4B[4]

        FC[First Record]
        0C[0]
        1C[1]
        2C[2]
        LC[Next record when written]
        LB[Next record when written]

    subgraph "PP"

        subgraph Partition0
            direction LR
            FA --> 0A
            N1A[Next] --> 1A
            N2A[Next] --> 2A
            LA --> 3A
        end

        subgraph Partition1
            direction LR
            FB --> 0B
            N1B[Next] --> 1B
            N2B[Next] --> 2B
            N3B[Next] --> 3B
            LB --> 4B
        end


        subgraph Partition2
            direction LR
             FC       --> 0C
            N1C[Next] --> 1C
             LC       --> 2C
        end

    end

```

Topic is a category of data, for example page views, searches. Each of these is partitioned by say user id. Each partition is a sequence of records that are being contiously appended to. Maintained for a week or so.

How is this a a messaging system, it's like Apache log?
Logs are fantastic mechanism for implementing a publish subscribe system.

```mermaid
flowchart TD
    F[First Record]
    0
    1
    2
    3
    4

    subgraph Datasource
        DS
    end
    subgraph DestinationB
        DB
    end
    subgraph DestinationA
        DA
    end

    subgraph Log
        F --> 0
        N1[Next] --> 1
        N2[Next] --> 2
        2 --> |Reads | DA
        N3[Next] --> 3
        3 --> |Reads | DB
        DS --> |Writes | 4

    end
```

All subscribers see the same sequence of data.
In messaging not all consumers see the same data?!

Log sequence number acts as a proxy of time. A record that comes after another record in the log is newer.

Idea of logs keep coming in many different contexts. DBs replicate using logs. Google's spanner uses logs.

Recently Leslie lamport won a turing award for his work on a consensus algorithm called PAXOS which is a consensus algorithm. It maintains agreement on a distributed log. Allows you to build fault-tolerant distributed log.

Quite a fundamental idea.

Used for two things:

1. For replicating data
2. For consistency - multiple systems agree on the same sequence of events.

Example a fault tolerant distributed CEO Hash Table.

Single system -> not a problem.

Multiple systems will lead to many problems.

How databases maintain the sequence of updates.

```mermaid
flowchart TD
    F[PUT Microsoft, Bill Gates]
    0
    1
    2
    3
    4

    subgraph Datasource
        DS
    end
    subgraph DestinationB
        DB
    end
    subgraph DestinationA
        DA
    end

    subgraph Log
        F --> 0
        N1[PUT Microsoft Steve Balmer] --> 1
        N2[Put Apple Jobs] --> 2
        2 --> |Reads | DA
        N3[Put Microsoft Satya Nadella] --> 3
        3 --> |Reads | DB
        DS --> |Writes | 4

    end
```

There are two design styles

1. State machine replication
2. Primary backup replication (master slave)

## Logs and Data Integration

Example : User views job

```mermaid

flowchart TD
    subgraph User views job
        JF[Job Frontend]

        JF --> |Job Views| K[Kafka]
        K --> |Job Views| Hadoop
        K --> |Job Views| Security
        K --> |Job Views| JPA[Job Poster Analytics]
        K --> |Job Views| REC[Recommendation Engineering]
        K --> |Job Views| M[Monitoring]
    end
```

Gets complicated over time.

Job front end can publish and the consumers can subscribe.

It's all one big distributed system and they all have exactly the same replication and consistency problems like a database, but they have in the large.

A commit log keeps everybody in sync on the same set of data as that data changes in real time. 

Traditionally you have two solutions

1. Copy files around or
   1. Data delivered in milli seconds.
   2. Multi subscriber

2. Messaging
   1. Single log vs queue per subscriber
   2. Large persistent data

## Logs and Stream Processing

A processing layer will run in real time.

Think of all your data a big distributed database. Kafka is like a commit log for that database.


Stream processing is a trigger or a materialized view of that system.

A way to compute new things of those things. 

Different from map reduce as it's real time processing.

Next evolution of message processing.

Stream processing gives richer semantics which a messaging system doesn't

Stream processing is no longer niche. 

Different between a batch process and a stream process.

Think of US Census. Actually every 10 years, somebody goes to each house and counts the number of people. That is batch processing.

Easier way would be to have a log of all the births and deaths. This way you'll know the population at any point in time. You can go back in time as well. This is stream processing.

Anologous example. How many people are there in the party. Send someone to check every hour. This is batch processing. Another way is to have a log of people entering and leaving the party. This is stream processing. This log will tell us at any point in time how many people are there in the party.

Stream processing is taking logs and transforming them into new logs.

Similar to trigger or materialized view in a database.

Systems that subscribe do not care whether it's the original stream or a derived stream. 

Stream processing is the ability to do real time data processing of of your real time feeds.

Stream processing is a generalization of batch processing.

Batch - you process all your records.
Request-Response - you process one record at a time.

Stream processing is a generalisation in  between the two. You take some number of records and output some number of records.

There is no end of the stream. It's a continuous stream. But's that true for all data. Batch jobs only take a arbitrary subset of the time. It's better to have a control over the time, especially for domains that are low latency.

At LinkedIn
50 % - Request-Response
25 % - Batch
25 % - Stream - low latency, asynchronous processing  (not well supported now)

Examples of stream processing

 - Monitoring
 - Security
 - Content processing
 - Recommendations
 - Newsfeed
 - ETL

Samza and Storm make use of Kafka.


Samz architecture

```mermaid
flowchart
    subgraph Samza Architecture
        subgraph Top      
            Job1
            Job2
            Job3
            Job4
        end
        subgraph Middle
            Samza
        end

        subgraph Bottom
            subgraph Kafka1
                Kafka
            end
            subgraph YARN1
                YARN
            end
        end
   end

```

```mermaid
flowchart
    subgraph Log-centric Architecture
        GO[Graph DB, OLAP Store etc]
        KV[Key-Value Query Layer]
        SQL[Search Query Layer]
        MG[Monitoring and Graphs]
        SP[Stream Processing]
        H[Hadoop]
        L[Log]

        L --> GO
        L <--> KV
        L --> SQL
        L --> MG
        L <--> SP
        L <--> H



    end


```





> [Home](HOME.md)
