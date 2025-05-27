# iDSC 2025 Workshop: Stream Processing with FlinkSQL

Welcome to the iDSC 2025 workshop! In this session, we will explore FlinkSQL using a Redpanda-backed environment. We'll start with simple queries and gradually move to more complex tasks.

---

## Prerequisites
Before we begin, ensure the following:
1. The Flink/Redpanda environment is running.
2. You have access to the Flink SQL Client (`sql-client`).

---

## 1. Getting Started with Flink SQL

### 1.1. Connect to the Flink SQL Client
Run the following command to start the Flink SQL Client:
```bash
docker compose -p idsc2025 run sql-client
```

### 1.2 Create a dynamic table

Create a dynamic table that connects to the redpanda sls_monitoring topic: 
```sql
CREATE TABLE sls_monitoring (
  run_id STRING,
  batch INT,
  phase STRING,
  event_time TIMESTAMP(3),
  step INT,
  temperature DOUBLE,
  agitation DOUBLE,
  pressure DOUBLE
) WITH (
  'connector' = 'kafka',
  'topic' = 'sls_monitoring',
  'properties.bootstrap.servers' = 'redpanda:9092',
  'properties.group.id' = 'sls-monitoring-group',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'json',
  'json.timestamp-format.standard' = 'ISO-8601'
);
```

### 1.3 Verify the table

```sql
SHOW TABLES;
```

### 1.4 Real-time monitoring of sensor readings

Ensure that the simulator is running and sending objects to the Redpanda topic.
You should see a result table that gets continuously updated when a new message arrives at the topic. 

```sql
SELECT * FROM sls_monitoring;
```

## 2. Basic Queries

### 2.1 Filter data by phase

```sql
SELECT * FROM sls_monitoring WHERE phase = 'Reaction Hold';
```

### 2.2 Filter data exceeding a temperature threshold

```sql
SELECT * FROM sls_monitoring WHERE phase = 'Reaction Hold' AND temperature > 85.5;
```

## 3. Aggregations

### 3.1 Count overall sensor readings

```sql
SELECT COUNT(*) AS total_readings FROM sls_monitoring;
```

### 3.2 Monitor average temperature in the reactor during Reaction Hold phase

 ```sql
SELECT AVG(temperature) AS avg_temperature FROM sls_monitoring WHERE phase='Reaction Hold';
 ```

## 4. Grouped Aggregations

### 4.1 Monitor average temperature in the reactor during Reaction Hold phase per batch

```sql
SELECT 
  run_id,
  batch,
  phase,
  AVG(temperature) AS avg_temperature
FROM sls_monitoring
GROUP BY run_id, batch, phase;
```

### 4.2 Monitor average temperature by current running phase

```sql
SELECT 
  run_id,
  batch,
  phase,
  AVG(temperature) AS avg_temperature
FROM sls_monitoring
WHERE event_time = (
  SELECT MAX(event_time) FROM sls_monitoring
)
GROUP BY run_id, batch, phase;
```

### 4.3 Monitor average sensor readings during Main Heating

```sql
SELECT 
  run_id,
  batch,
  phase,
  MIN(event_time) AS phase_start,
  MAX(event_time) AS phase_end,
  AVG(temperature) AS avg_temperature,
  MIN(temperature) AS min_temperature,
  MAX(temperature) AS max_temperature
FROM sls_monitoring
WHERE phase = 'Main Heating'
GROUP BY run_id, batch, phase;
```

## 5. Time-Based Queries

### 5.1 Create a new table 

Watermarks are used in event-time processing to handle late or out-of-order events
This tells Flink to treat records as "on time" if they arrive within 5 seconds of their event time, allowing for out-of-order data.

```sql
CREATE TABLE sls_monitoring_time (
  run_id STRING,
  batch INT,
  phase STRING,
  event_time TIMESTAMP(3),
  step INT,
  temperature DOUBLE,
  agitation DOUBLE,
  pressure DOUBLE,
  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'sls_monitoring',
  'properties.bootstrap.servers' = 'redpanda:9092',
  'properties.group.id' = 'sls-monitoring-group',
  'scan.startup.mode' = 'earliest-offset',
  'format' = 'json',
  'json.timestamp-format.standard' = 'ISO-8601'
);
```

### 5.2 Minimum and maximum temperature per Batch in 30-Second tumbling Windows

This query uses a tumbling window. It tracks the temperature range in fixed, non-overlapping time windows of 15 seconds.
Queries like that can help to e.g. observe trends or summarize values during a given time period. 
In the example case we could use it to monitor process stability.

```sql
SELECT
  TUMBLE_START(event_time, INTERVAL '15' SECOND) AS window_start,
  TUMBLE_END(event_time, INTERVAL '15' SECOND) AS window_end,
  batch,
  MIN(temperature) AS min_temperature,
  MAX(temperature) AS max_temperature
FROM sls_monitoring_time
GROUP BY
  TUMBLE(event_time, INTERVAL '15' SECOND),
  batch;
```

### 5.2 Count of Events per phase in 15-Second Tumbling Windows

Note that this is independent of the phase.

```sql
SELECT
  TUMBLE_START(event_time, INTERVAL '15' SECOND) AS window_start,
  TUMBLE_END(event_time, INTERVAL '15' SECOND) AS window_end,
  phase,
  COUNT(*) AS event_count
FROM sls_monitoring_time
GROUP BY
  TUMBLE(event_time, INTERVAL '15' SECOND),
  phase;
```

### 5.3 Average temperature per sensor in 15-second Tumbling Windows
```sql
SELECT
  TUMBLE_START(event_time, INTERVAL '15' SECOND) AS window_start,
  TUMBLE_END(event_time, INTERVAL '15' SECOND) AS window_end,
  run_id,
  batch,
  AVG(temperature) AS avg_temperature
FROM sls_monitoring_time
GROUP BY
  TUMBLE(event_time, INTERVAL '15' SECOND),
  run_id,
  batch;
```

## 6. Other examples

### 6.1 Find duplicates 

```sql
SELECT 
  run_id,
  event_time,
  COUNT(*) AS event_count
FROM sls_monitoring
GROUP BY run_id, event_time
HAVING COUNT(*) > 1;
```