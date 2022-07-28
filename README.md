## Real-time Data Warehouse

Real-time Data Warehouse using: [Flink & Kafka](https://github.com/izhangzhihao/Real-time-Data-Warehouse/tree/kafka) | [Flink & Hudi](https://github.com/izhangzhihao/Real-time-Data-Warehouse/tree/hudi) | [Spark & Delta](https://github.com/izhangzhihao/Real-time-Data-Warehouse/tree/spark) | [Flink & Hudi & E-commerce](https://github.com/izhangzhihao/Real-time-Data-Warehouse/tree/e-commerce)

<p align="center">
<img width="700" alt="demo_overview" src="https://user-images.githubusercontent.com/12044174/125452701-2717d438-c2e5-43f9-94c9-aaa804774699.png">
</p>

#### Getting the setup up and running

`docker compose build`

`docker compose up -d`

#### Check everything really up and running

`docker compose ps`

You should be able to access the Flink Web UI (http://localhost:8081), as well as Kibana (http://localhost:5601).

## Postgres

Start the Postgres client to have a look at the source tables and run some DML statements later:

```bash
docker compose exec postgres env PGOPTIONS="--search_path=claims" bash -c 'psql -U $POSTGRES_USER postgres'
```

#### What tables are we dealing with?

```sql
SELECT * FROM information_schema.tables WHERE table_schema = 'claims';
```

## Debezium

Start the [Debezium Postgres connector](https://debezium.io/documentation/reference/1.2/connectors/postgresql.html)
using the configuration provided in the `register-postgres.json` file:

```bash
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres.json
curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @register-postgres-members.json
```

Check that the connector is running:

```bash
curl http://localhost:8083/connectors/claims-connector/status  | jq
```

The first time it connects to a Postgres server, Debezium takes
a [consistent snapshot](https://debezium.io/documentation/reference/1.2/connectors/postgresql.html#postgresql-snapshots)
of all database schemas; so, you should see that the pre-existing records in the `accident_claims` table have already
been pushed into your Kafka topic:

```bash
docker compose exec kafka /kafka/bin/kafka-console-consumer.sh \
    --bootstrap-server kafka:9092 \
    --from-beginning \
    --property print.key=true \
    --topic pg_claims.claims.accident_claims
```

> ℹ️ Have a quick read about the structure of these events in the [Debezium documentation](https://debezium.io/documentation/reference/1.2/connectors/postgresql.html#postgresql-change-events-value).

### Is it working?

In the tab you used to start the Postgres client, you can now run some DML statements to see that the changes are
propagated all the way to your Kafka topic:

```sql
INSERT INTO accident_claims (claim_total, claim_total_receipt, claim_currency, member_id, accident_date, accident_type,accident_detail, claim_date, claim_status) VALUES (500, 'PharetraMagnaVestibulum.tiff', 'AUD', 321, '2020-08-01 06:43:03', 'Collision', 'Blue Ringed Octopus','2020-08-10 09:39:31', 'INITIAL');
```

```sql
UPDATE accident_claims
SET claim_total_receipt = 'CorrectReceipt.pdf'
WHERE claim_id = 1001;
```

```sql
DELETE
FROM accident_claims
WHERE claim_id = 1001;
```

In the output of your Kafka console consumer, you should now see three consecutive events with `op` values equal
to `c` (an _insert_ event), `u` (an _update_ event) and `d` (a _delete_ event).

## Flink connectors

https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/connectors/table/overview/
https://flink-packages.org/categories/connectors
https://github.com/knaufk/flink-faker/

## Datasource ingestion

Start the Flink SQL Client:

```bash
docker compose exec sql-client ./sql-client.sh
```


OR

```bash
docker compose exec sql-client ./sql-client-submit.sh
```

test

```sql
CREATE TABLE t1(
  uuid VARCHAR(20), -- you can use 'PRIMARY KEY NOT ENFORCED' syntax to mark the field as record key
  name VARCHAR(10),
  age INT,
  ts TIMESTAMP(3),
  `partition` VARCHAR(20)
)
PARTITIONED BY (`partition`)
WITH (
  'connector' = 'hudi',
  'path' = '/data/t1',
  'write.tasks' = '1', -- default is 4 ,required more resource
  'compaction.tasks' = '1', -- default is 10 ,required more resource
  'table.type' = 'COPY_ON_WRITE', -- this creates a MERGE_ON_READ table, by default is COPY_ON_WRITE
  'read.tasks' = '1', -- default is 4 ,required more resource
  'read.streaming.enabled' = 'true',  -- this option enable the streaming read
  'read.streaming.start-commit' = '20210712134429', -- specifies the start commit instant time
  'read.streaming.check-interval' = '4' -- specifies the check interval for finding new source commits, default 60s.
);

-- insert data using values
INSERT INTO t1 VALUES
  ('id1','Danny',23,TIMESTAMP '1970-01-01 00:00:01','par1'),
  ('id2','Stephen',33,TIMESTAMP '1970-01-01 00:00:02','par1'),
  ('id3','Julian',53,TIMESTAMP '1970-01-01 00:00:03','par2'),
  ('id4','Fabian',31,TIMESTAMP '1970-01-01 00:00:04','par2'),
  ('id5','Sophia',18,TIMESTAMP '1970-01-01 00:00:05','par3'),
  ('id6','Emma',20,TIMESTAMP '1970-01-01 00:00:06','par3'),
  ('id7','Bob',44,TIMESTAMP '1970-01-01 00:00:07','par4'),
  ('id8','Han',56,TIMESTAMP '1970-01-01 00:00:08','par4');

SELECT * FROM t1;
```

Register
a [Postgres catalog](https://ci.apache.org/projects/flink/flink-docs-stable/dev/table/connectors/jdbc.html#postgres-database-as-a-catalog)
, so you can access the metadata of the external tables over JDBC:

```sql
CREATE CATALOG datasource WITH (
    'type'='jdbc',
    'property-version'='1',
    'base-url'='jdbc:postgresql://postgres:5432/',
    'default-database'='postgres',
    'username'='postgres',
    'password'='postgres'
);
```

```sql
CREATE DATABASE IF NOT EXISTS datasource;
```

```sql
CREATE TABLE datasource.accident_claims WITH (
                                            'connector' = 'kafka',
                                            'topic' = 'pg_claims.claims.accident_claims',
                                            'properties.bootstrap.servers' = 'kafka:9092',
                                            'properties.group.id' = 'accident_claims-consumer-group',
                                            'format' = 'debezium-json',
                                            'scan.startup.mode' = 'earliest-offset'
                                            ) LIKE datasource.postgres.`claims.accident_claims` (EXCLUDING ALL);
```

OR generate data from datagen connector:

```sql
CREATE TABLE datasource.accident_claims(
    claim_id            BIGINT,
    claim_total         DOUBLE,
    claim_total_receipt VARCHAR(50),
    claim_currency      VARCHAR(3),
    member_id           INT,
    accident_date       DATE,
    accident_type       VARCHAR(20),
    accident_detail     VARCHAR(20),
    claim_date          DATE,
    claim_status        VARCHAR(10),
    ts_created          TIMESTAMP(3),
    ts_updated          TIMESTAMP(3)
                                          ) WITH (
                                            'connector' = 'datagen',
                                            'rows-per-second' = '100'
                                            );
```

and `members` table:

```sql
CREATE TABLE datasource.members WITH (
                                    'connector' = 'kafka',
                                    'topic' = 'pg_claims.claims.members',
                                    'properties.bootstrap.servers' = 'kafka:9092',
                                    'properties.group.id' = 'members-consumer-group',
                                    'format' = 'debezium-json',
                                    'scan.startup.mode' = 'earliest-offset'
                                    ) LIKE datasource.postgres.`claims.members` ( EXCLUDING OPTIONS);
```

OR generate data from datagen connector:

```sql
CREATE TABLE datasource.members(
    id                BIGINT,
    first_name        VARCHAR(50),
    last_name         VARCHAR(50),
    address           VARCHAR(50),
    address_city      VARCHAR(10),
    address_country   VARCHAR(10),
    insurance_company VARCHAR(25),
    insurance_number  VARCHAR(50),
    ts_created        TIMESTAMP(3),
    ts_updated        TIMESTAMP(3)
                                    ) WITH (
                                            'connector' = 'datagen',
                                            'rows-per-second' = '100'
                                            );
```

Check data:

```sql
SELECT * FROM datasource.accident_claims;
SELECT * FROM datasource.members;
```

## DWD

Create a database in DWD layer:

```sql
CREATE DATABASE IF NOT EXISTS dwd;
```

```sql
CREATE TABLE dwd.accident_claims
(
    claim_id            BIGINT,
    claim_total         DOUBLE,
    claim_total_receipt VARCHAR(50),
    claim_currency      VARCHAR(3),
    member_id           INT,
    accident_date       DATE,
    accident_type       VARCHAR(20),
    accident_detail     VARCHAR(20),
    claim_date          DATE,
    claim_status        VARCHAR(10),
    ts_created          TIMESTAMP(3),
    ts_updated          TIMESTAMP(3),
    ds                  DATE,
    PRIMARY KEY (claim_id) NOT ENFORCED
) PARTITIONED BY (ds) WITH (
  'connector'='hudi',
  'path' = '/data/dwd/accident_claims',
  'table.type' = 'MERGE_ON_READ',
  'read.streaming.enabled' = 'true',
  'write.batch.size' = '1',
  'write.tasks' = '1',
  'compaction.tasks' = '1',
  'compaction.delta_seconds' = '60',
  'write.precombine.field' = 'ts_updated',
  'read.tasks' = '1',
  'read.streaming.check-interval' = '5',
  'read.streaming.start-commit' = '20210712134429',
  'index.bootstrap.enabled' = 'true'
);
```

```sql
CREATE TABLE dwd.members
(
    id                BIGINT,
    first_name        VARCHAR(50),
    last_name         VARCHAR(50),
    address           VARCHAR(50),
    address_city      VARCHAR(10),
    address_country   VARCHAR(10),
    insurance_company VARCHAR(25),
    insurance_number  VARCHAR(50),
    ts_created        TIMESTAMP(3),
    ts_updated        TIMESTAMP(3),
    ds                DATE,
    PRIMARY KEY (id) NOT ENFORCED
) PARTITIONED BY (ds) WITH (
      'connector'='hudi',
      'path'='/data/dwd/members',
      'table.type' = 'MERGE_ON_READ',
      'read.streaming.enabled' = 'true',
      'write.batch.size' = '1',
      'write.tasks' = '1',
      'compaction.tasks' = '1',
      'compaction.delta_seconds' = '60',
      'write.precombine.field' = 'ts_updated',
      'read.tasks' = '1',
      'read.streaming.check-interval' = '5',
      'read.streaming.start-commit' = '20210712134429',
      'index.bootstrap.enabled' = 'true'
);
```

and submit a continuous query to the Flink cluster that will write the data from datasource into dwd table(ES):

```sql
INSERT INTO dwd.accident_claims
SELECT claim_id,
       claim_total,
       claim_total_receipt,
       claim_currency,
       member_id,
       CAST (accident_date as DATE),
       accident_type,
       accident_detail,
       CAST (claim_date as DATE),
       claim_status,
       CAST (ts_created as TIMESTAMP),
       CAST (ts_updated as TIMESTAMP),
       claim_date
       --CAST (SUBSTRING(claim_date, 0, 9) as DATE)
FROM datasource.accident_claims;
```

```sql
INSERT INTO dwd.members
SELECT id,
       first_name,
       last_name,
       address,
       address_city,
       address_country,
       insurance_company,
       insurance_number,
       CAST (ts_created as TIMESTAMP),
       CAST (ts_updated as TIMESTAMP),
       ts_created
       --CAST (SUBSTRING(ts_created, 0, 9) as DATE)
FROM datasource.members;
```

Check data:

```sql
SELECT * FROM dwd.accident_claims;
SELECT * FROM dwd.members;
```

## DWB

Create a database in DWB layer:

```sql
CREATE DATABASE IF NOT EXISTS dwb;
```

```sql
CREATE TABLE dwb.accident_claims
(
    claim_id            BIGINT,
    claim_total         DOUBLE,
    claim_total_receipt VARCHAR(50),
    claim_currency      VARCHAR(3),
    member_id           INT,
    accident_date       DATE,
    accident_type       VARCHAR(20),
    accident_detail     VARCHAR(20),
    claim_date          DATE,
    claim_status        VARCHAR(10),
    ts_created          TIMESTAMP(3),
    ts_updated          TIMESTAMP(3),
    ds                  DATE,
    PRIMARY KEY (claim_id) NOT ENFORCED
) PARTITIONED BY (ds) WITH (
  'connector'='hudi',
  'path' = '/data/dwb/accident_claims',
  'table.type' = 'MERGE_ON_READ',
  'read.streaming.enabled' = 'true',
  'write.batch.size' = '1',
  'write.tasks' = '1',
  'compaction.tasks' = '1',
  'compaction.delta_seconds' = '60',
  'write.precombine.field' = 'ts_updated',
  'read.tasks' = '1',
  'read.streaming.check-interval' = '5',
  'read.streaming.start-commit' = '20210712134429',
  'index.bootstrap.enabled' = 'true'
);
```

```sql
CREATE TABLE dwb.members
(
    id                BIGINT,
    first_name        VARCHAR(50),
    last_name         VARCHAR(50),
    address           VARCHAR(50),
    address_city      VARCHAR(10),
    address_country   VARCHAR(10),
    insurance_company VARCHAR(25),
    insurance_number  VARCHAR(50),
    ts_created        TIMESTAMP(3),
    ts_updated        TIMESTAMP(3),
    ds                DATE,
    PRIMARY KEY (id) NOT ENFORCED
) PARTITIONED BY (ds) WITH (
      'connector'='hudi',
      'path'='/data/dwb/members',
      'table.type' = 'MERGE_ON_READ',
      'read.streaming.enabled' = 'true',
      'write.batch.size' = '1',
      'write.tasks' = '1',
      'compaction.tasks' = '1',
      'compaction.delta_seconds' = '60',
      'write.precombine.field' = 'ts_updated',
      'read.tasks' = '1',
      'read.streaming.check-interval' = '5',
      'read.streaming.start-commit' = '20210712134429',
      'index.bootstrap.enabled' = 'true'
);
```

```sql
INSERT INTO dwb.accident_claims
SELECT claim_id,
       claim_total,
       claim_total_receipt,
       claim_currency,
       member_id,
       accident_date,
       accident_type,
       accident_detail,
       claim_date,
       claim_status,
       ts_created,
       ts_updated,
       ds
FROM dwd.accident_claims;
```

```sql
INSERT INTO dwb.members
SELECT id,
       first_name,
       last_name,
       address,
       address_city,
       address_country,
       insurance_company,
       insurance_number,
       ts_created,
       ts_updated,
       ds
FROM dwd.members;
```

Check data:

```sql
SELECT * FROM dwb.accident_claims;
SELECT * FROM dwb.members;
```

## DWS

Create a database in DWS layer:

```sql
CREATE DATABASE IF NOT EXISTS dws;
```

```sql
CREATE TABLE dws.insurance_costs
(
    es_key            STRING PRIMARY KEY NOT ENFORCED,
    insurance_company STRING,
    accident_detail   STRING,
    accident_agg_cost DOUBLE
) WITH (
      'connector' = 'elasticsearch-7', 'hosts' = 'http://elasticsearch:9200', 'index' = 'agg_insurance_costs'
      );
```

and submit a continuous query to the Flink cluster that will write the aggregated insurance costs
per `insurance_company`, bucketed by `accident_detail` (or, what animals are causing the most harm in terms of costs):

```sql
INSERT INTO dws.insurance_costs
SELECT UPPER(SUBSTRING(m.insurance_company, 0, 4) || '_' || SUBSTRING(ac.accident_detail, 0, 4)) es_key,
       m.insurance_company,
       ac.accident_detail,
       SUM(ac.claim_total) member_total
FROM dwb.accident_claims ac
         JOIN dwb.members m
              ON ac.member_id = m.id
WHERE ac.claim_status <> 'DENIED'
GROUP BY m.insurance_company, ac.accident_detail;
```

Finally, create a
simple [dashboard in Kibana](https://www.elastic.co/guide/en/kibana/current/dashboard-create-new-dashboard.html) 









## References

* [Flink SQL DDL](https://docs.google.com/document/d/1TTP-GCC8wSsibJaSUyFZ_5NBAHYEB1FVmPpP7RgDGBA/edit)
* [Flink SQL Client](https://ci.apache.org/projects/flink/flink-docs-release-1.13/docs/dev/table/sqlclient/)
* [Flink SQL Cookbook](https://github.com/ververica/flink-sql-cookbook)
* [Change Data Capture with Flink SQL and Debezium](https://noti.st/morsapaes/liQzgs/change-data-capture-with-flink-sql-and-debezium)
