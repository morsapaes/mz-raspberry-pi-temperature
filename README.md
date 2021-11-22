# Materialize - Raspberry Pi Temperature Sensors Demo

This is a self-contained demo using [Materialize](https://materialize.com/) to process data directly from a PostgreSQL server. The data is generated by a Raspberry Pi temperature mock service simulating 50 devices reporting to an AdonisJS API mock service. Finally, we will create a sink to let us stream the data out of Materialize to a [Redpanda](https://materialize.com/docs/third-party/redpanda/) topic.

![mz-raspberry-pi-temperature diagram](https://user-images.githubusercontent.com/21223421/142861869-cf8dc904-6585-4027-a38f-795886c66622.png)

## Prerequisites

Before you get started, you need to make sure that you have Docker and Docker Compose installed.

You can follow the steps here on how to install Docker:

> [Installing Docker](https://materialize.com/docs/third-party/docker/)

## Overview

In this demo, we’ll look at monitoring the temperature of a set of Raspberry Pi devices and extracting some insights from them and stream the data out to an external source.

### Raspberry Pi Mock

The main source of data is a Raspberry Pi Mock service, that simulates 50 devices reporting their CPU temperature to a mock API service built with AdonisJS.

The mock service generates about ~25 new requests to the mock API service every second.

### API Mock service and PostgreSQL

The API mock service receives the data from the 50 simulated Raspberry Pi and stores each request in a PostgreSQL instance.

The data that is being received with each request is:

* The name of the Raspberry Pi device.
* The timestamp when the temperature was measured.
* The temperature of the device, in celsius.

The Mock API will save all data in a table called `sensors`. The columns of the `sensors` table are:

* `name`
* `timestamp`
* `temperature`

### Materialize

Materialize presents an interface to ingest the temperature data from the PostgreSQL database.

In this demo, we are going to use Materialize to:

* [Create a PostgreSQL source](https://materialize.com/docs/sql/create-source/postgres/)
* Materialize the PostgreSQL data, which will be retained all in memory.
* Provide a SQL interface to query the temperature data. We will connect to Materialize through mzcli, which is our forked version of pgcli.
* Explore the Materialize data via Metabase.

## Running the demo

Clone the repository:

```
git clone https://github.com/bobbyiliev/mz-raspberry-pi-temperature.git
```

Access the directory:

```
cd mz-raspberry-pi-temperature
```

Build the Raspberry Pi Mock images:

```
docker-compose build
```

Start all of the services:

```
docker-compose up -d
```

### Access Materialize

```
docker-compose run mzcli
```

### Create Materialize Source:

To create a PostgreSQL Materialize Source run the following statement:

```sql
CREATE MATERIALIZED SOURCE "mz_source" FROM POSTGRES
CONNECTION 'user=postgres port=5432 host=postgres dbname=postgres password=postgres'
PUBLICATION 'mz_source';
```

A quick rundown of the above statement:

* `MATERIALIZED`: Materializes the PostgreSQL source’s data. All of the data is retained in memory and makes sources directly selectable.
* `mz_source`: The name for the PostgreSQL source.
* `CONNECTION`: The PostgreSQL connection parameters.
* `PUBLICATION`: The PostgreSQL publication, containing the tables to be streamed to Materialize.


### Create a view:

Once we've created the PostgreSQL source, in order to be able to query the PostgreSQL tables, we would need to create views that represent the upstream publication’s original tables. In our case, we only have one table called `sensors` so the statement that we would need to run is:

```sql
CREATE VIEWS FROM SOURCE mz_source (sensors);
```

To see the available views run:

```sql
SHOW FULL VIEWS;
```

Once that is done, you can query the new views directly:

```sql
SELECT * FROM sensors;
```

Next, let's go ahead and create a few more views.

### Creating more materialized views

Before, we start let's enable timing so we could actually see how long it takes for each statement to be executed:

```sql
\timing
```

* Example 1: Create a materialized view to show the total number of sensors data:

```sql
CREATE MATERIALIZED VIEW mz_count AS SELECT count(*) FROM sensors;
```

Querying the `mz_count` view:

```
SELECT * FROM mz_count;
```

Output:

```sql
 count
-------
 34565
(1 row)

Time: 2.299 ms
```

* Example 2: Create a view to show the average temperature of all sensors:

```sql
CREATE MATERIALIZED VIEW mz_total_avg AS SELECT avg(temperature::float) FROM sensors;
```

Query the `mz_total_avg`:

```sql
SELECT * FROM mz_total_avg;
```

Output:

```sql
        avg
-------------------
 59.02989081226408
(1 row)

Time: 2.984 ms
```

* Example 3: Create a view to show the average temperature of each separate sensor:

```sql
CREATE MATERIALIZED VIEW average AS
    SELECT name::text, avg(temperature::float) AS temp 
    FROM sensors
    GROUP BY (name);
```

Query the `average` view:

```sql
SELECT * FROM average LIMIT 10;
```

Output:

```sql
     name     |        temp
--------------+--------------------
 raspberry-1  |  58.60756530123859
 raspberry-2  |  58.95694631912029
 raspberry-3  | 58.628198038515066
 raspberry-4  |  59.40673999174753
 raspberry-5  | 59.079367226960734
 raspberry-6  |  58.96244838239402
 raspberry-7  |   58.4658871719401
 raspberry-8  |   58.9830811196705
 raspberry-9  | 59.398486896836936
 raspberry-10 | 59.669463513068024
(10 rows)

Time: 2.353 ms
```

Feel free to experiment by creating more materialized views.

## Creating a Sink

[Sinks](https://materialize.com/docs/sql/create-sink/) let you send data from Materialize to an external source.

For this Demo we will be using [Redpanda](https://materialize.com/docs/third-party/redpanda/).

Redpanda is a Kafka API-compatible and Materialize can process data from it just as it would processes data from a Kafka sources.

Let's create a materialized view, that will hold all of the devices with an average temperature of more than 60 celsius:

```sql
CREATE MATERIALIZED VIEW mz_high_temperature AS
    SELECT * FROM average WHERE temp > 60;
```

If you were to do a `SELECT` on this new materialized view, it would return only the devices with an average temperature of above 60 celsius:

```sql
SELECT * FROM mz_high_temperature;
```

Let's create a Sink where we will send the data of the above materialized view:

```sql
CREATE SINK high_temperature_sink
    FROM mz_high_temperature
    INTO KAFKA BROKER 'redpanda:9092' TOPIC 'high-temperature-sink'
    FORMAT AVRO USING
    CONFLUENT SCHEMA REGISTRY 'http://redpanda:8081';
```

Now if you were to connect to the Redpanda container and use the `rpk topic consume` command, you will be able to read the records from the topic.

However, as of the time being, we won’t be able to preview the results with `rpk` because it’s AVRO formatted. Redpanda would most likely implement this in the future, but for the moment, we can actually stream the topic back into Materialize to confirm the format.

First, get the name of the topic that has been automatically generated:

```sql
SELECT topic FROM mz_kafka_sinks;
```

Output:

```sql
                              topic
-----------------------------------------------------------------
 high-temperature-sink-u12-1637586945-13670686352905873426
```

> For more information on how the topic names are generated check out the documentation [here](https://materialize.com/docs/sql/create-sink/#kafka-sinks).

Then create a new Materialized Source from this Redpanda topic:

```sql
CREATE MATERIALIZED SOURCE high_temp_test
FROM KAFKA BROKER 'redpanda:9092' TOPIC 'high-temperature-sink-u12-1637586945-13670686352905873426'
FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY 'http://redpanda:8081';
```

> Make sure to change the topic name accordingly!

Finally, query this new materialized view:

```sql
SELECT * FROM high_temp_test LIMIT 2;
```

Now that you have the data in the topic, you can have other services connect to it and consume it and then trigger emails or alerts for example.

## Metabase

In order to access the [Metabase](https://materialize.com/docs/third-party/metabase/) instance visit `http://localhost:3030` if you are running the demo locally or `http://your_server_ip:3030` if you are running the demo on a server. Then follow the steps to complete the Metabase setup.

Make sure to select Materialize as the source of the data.

Once ready you will be able to visualize your data just as you would with a standard PostgreSQL database.

![Metabase](https://user-images.githubusercontent.com/21223421/142780602-043f36c7-f279-4dc7-8853-99ddb31b452f.png)

## Stopping the demo

To stop all of the services run:

```
docker-compose down
```

## Helpful resources

* [`CREATE SOURCE: PostgreSQL`](https://materialize.com/docs/sql/create-source/postgres/)
* [`CREATE SOURCE`](https://materialize.com/docs/sql/create-source/)
* [`CREATE VIEWS`](https://materialize.com/docs/sql/create-views)
* [`SELECT`](https://materialize.com/docs/sql/select)