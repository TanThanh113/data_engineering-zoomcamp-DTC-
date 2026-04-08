#  Transform And Tool
---
Now let's convert the notebook to Python script.
## Convert Notebook to Script
```
uv run jupyter nbconvert --to=script notebook.ipynb
mv notebook.py ingest_data.py
```
## Click Integration
Modifying lines of code is complicated and doesn't help with automation, so we'll use a Click tool to do it.
```python
import click

@click.command()
@click.option('--pg-user', default='thanh123', help = "PostgreSQL User")
@click.option('--pg-pass', default='1234', help = "PostgreSQL Password")
@click.option('--pg-host', default='localhost', help = "PostgreSQL Host")
@click.option('--pg-port', default='5433', help = "PostgreSQL Port")
@click.option('--pg-db', default='ny_taxi', help = "PostgreSQL Database Name")
@click.option('--target-table', default='yellow_taxi_date', help = "Target Table Name")
def run(pg_user, pg_pass, pg_host, pg_port, pg_db, target_table):
    #Logic code here
  pass
```
## Running the Script
The operation will be as follows (Window):
```
uv run python ingest_data.py `
  --pg-user=thanh123 `
  --pg-pass=1234 `
  --pg-host=localhost `
  --pg-port=5433 `
  --pg-db=ny_taxi `
  --target-table=yellow_taxi_trips
```
To learn how to input data correctly according to the structure created by others, we use the following method:
```
uv run python ingest_data.py --help
```
---
### END
