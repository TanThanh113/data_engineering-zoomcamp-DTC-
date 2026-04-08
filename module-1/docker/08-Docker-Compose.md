# Docker Compose
---
```docker-compose```: Instead of running multiple complex commands to run a container, we can use ```docker-compose``` to run several containers simultaneously.
Docker-compose uses a YAML file to run.
## Database
Run database
```python
services:
  pgdatabase:
    image: postgres:18
    environment:
      POSTGRES_USER: "thanh123"
      POSTGRES_PASSWORD: "1234"
      POSTGRES_DB: "ny_taxi"
    volumes:
      - "./ny_taxi_db_data:/var/lib/postgresql/data"
    ports:
      - "5433:5432"
```
## Explanation:
* We don't have to specify a network because docker compose takes care of it: every single container (or "service", as the file states) will run
within the same network and will be able to find each other according to their names (pgdatabase and pgadmin in this example).
* All other details from the docker run commands (environment variables, volumes and ports) are mentioned accordingly in the file following YAML syntax.

## Python
With this, we can run Python directly without needing a Docker build and run tool.

Run python
```python
ingest_data:
    build: .
    image: taxi_ingest:v001
    depends_on:
      pgdatabase:
        condition: service_healthy
    command: [
      "--pg-user=thanh123",
      "--pg-pass=1234",
      "--pg-host=pgdatabase",
      "--pg-port=5432",
      "--pg-db=ny_taxi",
      "--target-table=yellow_taxi_date_trip"
    ]
```
### Note
* Because running this can easily cause conflicts, as PostgreSQL might not have finished building while Python is already running, leading to a failed build and crash.

We need to add another block of code to check if the database has finished running.
```python
healthcheck:
      test: ["CMD-SHELL", "pg_isready -U thanh123 -d ny_taxi"]
      interval: 5s    # Check every 5 seconds.
      timeout: 5s     # If there is no response within 5 seconds, it is considered an error.
      retries: 5      # Try again up to 5 times.
      start_period: 10s
```
#### Explanation:
```test: ["CMD-SHELL", "pg_isready -U thanh123 -d ny_taxi"]```:
   * ```CMD-SHELL```: run the command in the terminal.
   * ```pg_isready```: A PostgreSQL command that checks instead of running.
   * ```-U thanh123 -d ny_taxi```: user and database
## Run Docker Compose
```
docker-compose up --build
```
## Stop Docker Compose
```
docker-compose down
```
## Other Useful Commands
```
# View logs
docker-compose logs

# Stop and remove volumes
docker-compose down -v

# running in the background
docker-compose up -d
```
## Benefits of Docker Compose
* Single command to start all services
* Automatic network creation
* Easy configuration management
* Declarative infrastructure
## Full Code Example
```python
services:
  pgdatabase:
    image: postgres:18
    environment:
      POSTGRES_USER: "thanh123"
      POSTGRES_PASSWORD: "1234"
      POSTGRES_DB: "ny_taxi"
    volumes:
      - "./ny_taxi_db_data:/var/lib/postgresql"
    ports:
      - "5433:5432"
      
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U thanh123 -d ny_taxi"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s

  ingest_data:
    build: .
    image: taxi_ingest:v001
    depends_on:
      pgdatabase:
        condition: service_healthy
    command: [
      "--pg-user=thanh123",
      "--pg-pass=1234",
      "--pg-host=pgdatabase",
      "--pg-port=5432",
      "--pg-db=ny_taxi",
      "--target-table=yellow_taxi_date_trip"
    ]
```
---
### END
