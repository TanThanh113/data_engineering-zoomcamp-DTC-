# Data Ingestion
---
## Setting Up Jupyter
---
Install Jupyter:
```python
uv add --dev jupyter
```
Initialize a Jupyter notebook on the web:
```python
uv run jupyter notebook
```
## The NYC Taxi Dataset
---
We will use data from [NYC DATASET](https://www1.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

Specifically, we will use the [YELLOW TAXI TRIO](https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/yellow_tripdata_2021-01.csv.gz)

However, the data on the website is in parquet format, while we use CSV format, so please download from this link: [here](https://github.com/DataTalksClub/nyc-tlc-data)

## Explore Data
---
Create notebook and run:
```python
#Add library
import pandas as pd

#Add data
url = "https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/"
df = pd.read_csv(url + "yellow_tripdata_2021-01.csv.gz", nrows = 100)

#Check the top 5 rows of data.
df.head()

#Check the data type of the column.
df.dtypes

```
## Create new data types
---
Because of the column limit, Python has to guess the data type, but this won't be completely accurate.

Therefore, we need to convert the data type.
```python
dtype = {
    "VendorID": "Int64",
    "passenger_count": "Int64",
    "trip_distance": "float64",
    "RatecodeID": "Int64",
    "store_and_fwd_flag": "string",
    "PULocationID": "Int64",
    "DOLocationID": "Int64",
    "payment_type": "Int64",
    "fare_amount": "float64",
    "extra": "float64",
    "mta_tax": "float64",
    "tip_amount": "float64",
    "tolls_amount": "float64",
    "improvement_surcharge": "float64",
    "total_amount": "float64",
    "congestion_surcharge": "float64"
}

# Since dtype cannot convert datetime data, we separate it and use parse_dates.
parse_dates = [
    "tpep_pickup_datetime",
    "tpep_dropoff_datetime"
]
df = pd.read_csv(
    url + "yellow_tripdata_2021-01.csv.gz",
    nrows = 100,
    dtype = dtype,
    parse_dates = parse_dates
)
```
## Ingesting Data into Postgres
---
### Steps to follow:
1. Install SQLAlchemy
2. Create Database Connection
3. Read it in chunks with pandas
4. Insert data into PostgreSQL using SQLAlchemy
### Install SQLAlchemy
```python
uv add sqlalchemy "psycopg[binary,pool]"
```
### Create Database Connection
```python
#Get the create_engine command from sqlaichemy
from sqlalchemy import create_engine

#create a connection port
#'.....+psycopg://user:password@...:port/data'
engine = create_engine('postgresql+psycopg://thanh123:1234@localhost:5433/ny_taxi')
```
### Get DDL Schema
```python
#`pd.io.sql.get_schema` converts a Python variable to another data type (MySQL, PostgreSQL, etc.).
print(pd.io.sql.get_schema(df, name='yellow_taxi_data', con = engine))
```
### Output
```sql
CREATE TABLE yellow_taxi_data (
	"VendorID" BIGINT, 
	tpep_pickup_datetime TIMESTAMP WITHOUT TIME ZONE, 
	tpep_dropoff_datetime TIMESTAMP WITHOUT TIME ZONE, 
	passenger_count BIGINT, 
	trip_distance FLOAT(53), 
	"RatecodeID" BIGINT, 
	store_and_fwd_flag TEXT, 
	"PULocationID" BIGINT, 
	"DOLocationID" BIGINT, 
	payment_type BIGINT, 
	fare_amount FLOAT(53), 
	extra FLOAT(53), 
	mta_tax FLOAT(53), 
	tip_amount FLOAT(53), 
	tolls_amount FLOAT(53), 
	improvement_surcharge FLOAT(53), 
	total_amount FLOAT(53), 
	congestion_surcharge FLOAT(53)
)
```
### Create th table
```python
# head(0): retrieve data from column
# name: name of table
# if_exists='replace': Delete the old table if it exists, create a new table if it doesn't.
df.head(n=0).to_sql(name='yellow_taxi_data', con=engine, if_exists='replace')
```
### Ingesting Data In Chunks
Because there's so much data, nobody wants to download it all at once.
```python
df_iter = pd.read_csv(
    url + 'yellow_tripdata_2021-01.csv.gz',
    dtype=dtype,
    parse_dates=parse_dates,
    iterator=True,
    chunksize=100000
)
#We have two new things:
#iterator: reads in parts
#chunksize: only allows a limited amount of data to be entered at a time.
#This is an iterator.
```
### Output Test
```python
for df_chunk in df_iter:
    print(len(df_chunk))
```
### Inserting Data
```python
df_chunk.to_sql(name='yellow_taxi_data', con=engine, if_exists='append')
```
### Some examples of adding data
```python
first = True

for df_chunk in df_iter:

    if first:
        # Create table schema (no data)
        df_chunk.head(0).to_sql(
            name="yellow_taxi_data",
            con=engine,
            if_exists="replace"
        )
        first = False
        print("Table created")

    # Insert chunk
    df_chunk.to_sql(
        name="yellow_taxi_data",
        con=engine,
        if_exists="append"
    )

    print("Inserted:", len(df_chunk))
```
or
```python
first_chunk = next(df_iter)

first_chunk.head(0).to_sql(
    name="yellow_taxi_data",
    con=engine,
    if_exists="replace"
)

print("Table created")

first_chunk.to_sql(
    name="yellow_taxi_data",
    con=engine,
    if_exists="append"
)

print("Inserted first chunk:", len(first_chunk))

for df_chunk in df_iter:
    df_chunk.to_sql(
        name="yellow_taxi_data",
        con=engine,
        if_exists="append"
    )
    print("Inserted chunk:", len(df_chunk))
```
## Check Progress
----
Add tqdm to see progress:
```python
uv add tqdm
```
Continue
```python
from tqdm.auto import tqdm

for df_chunk in tqdm(df_iter):
    ...
```
## Verify the Data
---
Connect to it using pgcli:
```
uv run pgcli -h localhost -p 5433 -u thanh123 -d ny_taxi
```
---
# END
