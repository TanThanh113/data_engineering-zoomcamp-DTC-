# KESTRA 
---
## Introduction
Then you can read some information [here](https://kestra.io/)
So here I'll talk about the basic operation of Kestra. It's a tool that runs on Docker Compose, and once it's finished, you run it locally on your web browser.
[example](https://github.com/TanThanh113/data-engineering-zoomcamp/blob/main/module2/flows/run-kestra.md)
## Operation
Kestra operates directly on the web, so you can check it out; it has a detailed guide.
## ETL process
Here, I will share the code for the steps to create a simple ETL process.

I will base my analysis on the New York taxi data from Module 1.
### Kestra startup
I've already posted this above; you can check it out there.
### Things that can be in a process
- [Flow](https://go.kestra.io/de-zoomcamp/flow) - a container for tasks and their orchestration logic. 
- [Tasks](https://go.kestra.io/de-zoomcamp/tasks) - the steps within a flow.
- [Inputs](https://go.kestra.io/de-zoomcamp/inputs) - dynamic values passed to the flow at runtime.
- [Outputs](https://go.kestra.io/de-zoomcamp/outputs) - pass data between tasks and flows.
- [Triggers](https://go.kestra.io/de-zoomcamp/triggers) - mechanism that automatically starts the execution of a flow.
- [Execution](https://go.kestra.io/de-zoomcamp/execution) - a single run of a flow with a specific state.
- [Variables](https://go.kestra.io/de-zoomcamp/variables) - key–value pairs that let you reuse values across tasks.
- [Plugin Defaults](https://go.kestra.io/de-zoomcamp/plugin-defaults) - default values applied to every task of a given type within one or more flows.
- [Concurrency](https://go.kestra.io/de-zoomcamp/concurrency) - control how many executions of a flow can run at the same time.
### Build the inputs
```yaml
id: 04_postgres_taxi
namespace: zoomcamp
description: |
  The CSV Data used in the course: https://github.com/DataTalksClub/nyc-tlc-data/releases

inputs:
  - id: taxi
    type: SELECT
    displayName: Select taxi type
    values: [yellow, green]
    defaults: yellow

  - id: year
    type: SELECT
    displayName: Select year
    values: ["2019", "2020"]
    defaults: "2019"

  - id: month
    type: SELECT
    displayName: Select month
    values: ["01", "02", "03", "04", "05", "06", "07", "08", "09", "10", "11", "12"]
    defaults: "01"

```
  * Create IDs and namespaces for the flow.
  * Create input fields so that you can change the variables in the download link, increasing flexibility.
  * Use type select so the user can interact with it.
### Variables
```yaml
variables:
  file: "{{inputs.taxi}}_tripdata_{{inputs.year}}-{{inputs.month}}.csv"
  staging_table: "public.{{inputs.taxi}}_tripdata_staging"
  table: "public.{{inputs.taxi}}_tripdata"
  data: "{{outputs.extract.outputFiles[inputs.taxi ~ '_tripdata_' ~ inputs.year ~ '-' ~ inputs.month ~ '.csv']}}"
```
  * Create variables and give them names so they can be easily manipulated.
### Build tasks
**```Because the code is very long, I will only demo it on Yellow.```**
#### Extract
 I'll go to the link to download it (since it's a compressed file, we'll need to decompress it).
 ```yaml
 - id: set_label
    type: io.kestra.plugin.core.execution.Labels
    labels:
      file: "{{render(vars.file)}}"
      taxi: "{{inputs.taxi}}"

  - id: extract
    type: io.kestra.plugin.scripts.shell.Commands
    outputFiles:
      - "*.csv"
    taskRunner:
      type: io.kestra.plugin.core.runner.Process
    commands:
      - wget -qO- https://github.com/DataTalksClub/nyc-tlc-data/releases/download/{{inputs.taxi}}/{{render(vars.file)}}.gz | gunzip > {{render(vars.file)}}
```
 #### Transform
 I'll break this down into smaller steps.
 1. Recreate the table format, dividing it into two tables: a main table (used to store official data) and a secondary table (to load unprocessed data) (I'm only demonstrating the main table).
     ```yaml
     - id: yellow_create_table
        type: io.kestra.plugin.jdbc.postgresql.Queries
        sql: |
          CREATE TABLE IF NOT EXISTS {{render(vars.table)}} (
              unique_row_id          text,
              filename               text,
              VendorID               text,
              tpep_pickup_datetime   timestamp,
              tpep_dropoff_datetime  timestamp,
              passenger_count        integer,
              trip_distance          double precision,
              RatecodeID             text,
              store_and_fwd_flag     text,
              PULocationID           text,
              DOLocationID           text,
              payment_type           integer,
              fare_amount            double precision,
              extra                  double precision,
              mta_tax                double precision,
              tip_amount             double precision,
              tolls_amount           double precision,
              improvement_surcharge  double precision,
              total_amount           double precision,
              congestion_surcharge   double precision
          );
     ```
> **```Note```**:
> I will add two more columns, unique_row_id and filename, to manage the data.

2. Start by clearing all the data in the auxiliary table so that you can fill it with new data.
     ```yaml
     - id: yellow_truncate_staging_table
        type: io.kestra.plugin.jdbc.postgresql.Queries
        sql: |
          TRUNCATE TABLE {{render(vars.staging_table)}};

      - id: yellow_copy_in_to_staging_table
        type: io.kestra.plugin.jdbc.postgresql.CopyIn
        format: CSV
        from: "{{render(vars.data)}}"
        table: "{{render(vars.staging_table)}}"
        header: true
        columns: [VendorID,tpep_pickup_datetime,tpep_dropoff_datetime,passenger_count,trip_distance,RatecodeID,store_and_fwd_flag,PULocationID,DOLocationID,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,improvement_surcharge,total_amount,congestion_surcharge]
     ```
3. Create an identifier for each data point.
     ```yaml
      - id: green_add_unique_id_and_filename
        type: io.kestra.plugin.jdbc.postgresql.Queries
        sql: |
          UPDATE {{render(vars.staging_table)}}
          SET 
            unique_row_id = md5(
              COALESCE(CAST(VendorID AS text), '') ||
              COALESCE(CAST(lpep_pickup_datetime AS text), '') || 
              COALESCE(CAST(lpep_dropoff_datetime AS text), '') || 
              COALESCE(PULocationID, '') || 
              COALESCE(DOLocationID, '') || 
              COALESCE(CAST(fare_amount AS text), '') || 
              COALESCE(CAST(trip_distance AS text), '')      
            ),
            filename = '{{render(vars.file)}}';
     ```
4. Start loading filtered data into the main table.
     ```yaml
      - id: yellow_merge_data
        type: io.kestra.plugin.jdbc.postgresql.Queries
        sql: |
          MERGE INTO {{render(vars.table)}} AS T
          USING {{render(vars.staging_table)}} AS S
          ON T.unique_row_id = S.unique_row_id
          WHEN NOT MATCHED THEN
            INSERT (
              unique_row_id, filename, VendorID, tpep_pickup_datetime, tpep_dropoff_datetime,
              passenger_count, trip_distance, RatecodeID, store_and_fwd_flag, PULocationID,
              DOLocationID, payment_type, fare_amount, extra, mta_tax, tip_amount, tolls_amount,
              improvement_surcharge, total_amount, congestion_surcharge
            )
            VALUES (
              S.unique_row_id, S.filename, S.VendorID, S.tpep_pickup_datetime, S.tpep_dropoff_datetime,
              S.passenger_count, S.trip_distance, S.RatecodeID, S.store_and_fwd_flag, S.PULocationID,
              S.DOLocationID, S.payment_type, S.fare_amount, S.extra, S.mta_tax, S.tip_amount, S.tolls_amount,
              S.improvement_surcharge, S.total_amount, S.congestion_surcharge
            );
     ```
#### Remove
To avoid overloading memory, files should be deleted after execution is complete.
```yaml
 - id: purge_files
    type: io.kestra.plugin.core.storage.PurgeCurrentExecutionFiles
```
#### PluginDefault
Since we're using PostgreSQL, we need to provide the port, username, and password.
```yaml
pluginDefaults:
  - type: io.kestra.plugin.jdbc.postgresql
    values:
      url: jdbc:postgresql://pgdatabase:5432/ny_taxi
      username: root
      password: root
```
