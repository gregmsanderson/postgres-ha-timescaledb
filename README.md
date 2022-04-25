# postgres-ha-timescaledb

This repo adds the TimescaleDB extension to Fly's [highly available Postgres cluster](https://github.com/fly-apps/postgres-ha).

## Motivation

If you need to store any kind of metrics, it's important to store that data efficiently and quickly query it. The [TimescaleDB](https://github.com/timescale/timescaledb) PostgreSQL extension is ideal for this. It provides automatic partitioning yet retains the standard PostgreSQL interface. Plus it includes additional functions for time-series analysis that are not present in vanilla PostgreSQL (such as the `time_bucket` function).

Since Fly's Postgres clusters are just regular Fly applications, in theory you _could_ launch a new app directly from this repo. However then _you_ would have to do a lot of the work that Fly does for you (create a volume for its data, possibly create a replica for HA, set all the secrets needed ...). So we tell Fly to create the database app, using its provided commands. Then we'll deploy _our_ fork which will install the TimescaleDB extension.

## Deploy

If you haven't already done so, [install the Fly CLI](https://fly.io/docs/getting-started/installing-flyctl/) and then [log in to Fly](https://fly.io/docs/getting-started/log-in-to-fly/).

1. Clone this repo
2. Run `fly pg create` to create a new database app. Give it a name, choose its region, and whether to use HA. It may take a few minutes to deploy.
3. (optional) If you would like to create a multi-region database app, [add read-replicas](https://fly.io/docs/getting-started/multi-region-databases/#create-a-postgresql-cluster) to it.
4. Edit the `fly.toml` in two places to match the values you chose for the database app's name and primary [region](https://fly.io/docs/reference/regions/#fly-io-regions):

    ```toml
    app = "your-pg-name"

    PRIMARY_REGION = "lhr"
    ```
4. Run `fly deploy` to apply the modifications to **install** the TimescaleDB extension. It may take a few minutes as new VMs will need to be created.
5. When complete, **enable** the TimescaleDB extension by using a stolon update (stolon controls the cluster):
    ```sh
    fly ssh console
    export $(cat /data/.env | xargs)
    stolonctl update --patch '{"pgParameters": { "shared_preload_libraries": "timescaledb"}}'
    exit
    ```
6. Run `fly restart your-database-name-here` to apply that stolon update to all VMs in the cluster.
7. Every minute or so, try `fly checks list` to confirm all is well. Once the restart completes you should see all the checks have a status of `passing` (there are usually three checks _per_ VM).
8. (optional) Confirm the TimescaleDB extension has been successfully installed by connecting to the database app (`fly ssh console`) and running these two commands:
    ```
    cat /data/postgres/postgresql.conf
    ```
    You should see that file now contains this line:
    ```
    shared_preload_libraries = 'timescaledb'`
    ```
    ... and ...
    ```
    dpkg -l | grep -E 'timescale'
    ```
    You should see the TimescaleDB extension listed:
    ```
    timescaledb-2-loader-postgresql-14 2.6.1~debian11                 amd64        The loader for TimescaleDB to load individual versions.
    timescaledb-2-postgresql-14        2.6.1~debian11                 amd64        An open-source time-series database based on PostgreSQL, as an extension.
    timescaledb-tools                  0.12.0~debian11                amd64        A suite of tools that can be used with TimescaleDB.
    ```

## Add TimescaleDB to a database

Usually you would attach a Fly app to the PostgreSQL app (by using the `fly pg attach --postgres-app your-database-name-here` command). That creates a new database (with the same name as the app) and provides the app with a secret `DATABASE_URL` which it can use to securely connect to it.

Or you can manually create a database (by using `CREATE DATABASE its_name;`) as we demonstrate below.

Armed with a database, the final step is to [add the TimescaleDB extension](https://docs.timescale.com/install/latest/self-hosted/installation-debian/#set-up-the-timescaledb-extension) _to_ that database. Since we have Wireguard and PostgreSQL installed locally, we can do that right now:

```
$ psql postgres://postgres:your-password-here@your-pg-app.internal:5432
psql (14.2)
Type "help" for help.

postgres=# create database delete_me;
CREATE DATABASE

postgres=# \c delete_me;
You are now connected to database "delete_me" as user "postgres".

delete_me=# CREATE EXTENSION IF NOT EXISTS timescaledb;
WELCOME TO
 _____ _                               _     ____________
|_   _(_)                             | |    |  _  \ ___ \
  | |  _ _ __ ___   ___  ___  ___ __ _| | ___| | | | |_/ /
  | | | |  _ ` _ \ / _ \/ __|/ __/ _` | |/ _ \ | | | ___ \
  | | | | | | | | |  __/\__ \ (_| (_| | |  __/ |/ /| |_/ /
  |_| |_|_| |_| |_|\___||___/\___\__,_|_|\___|___/ \____/
               Running version 2.6.1
For more information on TimescaleDB, please visit the following links:

 1. Getting started: https://docs.timescale.com/timescaledb/latest/getting-started
 2. API reference documentation: https://docs.timescale.com/api/latest
 3. How TimescaleDB is designed: https://docs.timescale.com/timescaledb/latest/overview/core-concepts

Note: TimescaleDB collects anonymous reports to better understand and assist our users.
For more information and how to disable, please see our docs https://docs.timescale.com/timescaledb/latest/how-to-guides/configuration/telemetry.

CREATE EXTENSION
delete_me=# \dx;
                                      List of installed extensions
    Name     | Version |   Schema   |                            Description
-------------+---------+------------+-------------------------------------------------------------------
 plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
 timescaledb | 2.6.1   | public     | Enables scalable inserts and complex queries for time-series data
(2 rows)

delete_me=# \q
```

## Using TimescaleDB

You use standard SQL but can now _also_ create and query hypertables (this example is from [the TimescaleDB repo](https://github.com/timescale/timescaledb#creating-a-hypertable))

```sql
-- Do not forget to create timescaledb extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- We start by creating a regular SQL table
CREATE TABLE conditions (
  time        TIMESTAMPTZ       NOT NULL,
  location    TEXT              NOT NULL,
  temperature DOUBLE PRECISION  NULL,
  humidity    DOUBLE PRECISION  NULL
);

-- Then we convert it into a hypertable that is partitioned by time
SELECT create_hypertable('conditions', 'time');

-- Insert data into the table
INSERT INTO conditions(time, location, temperature, humidity)
  VALUES (NOW(), 'office', 70.0, 50.0);

-- Query data in the table
SELECT * FROM conditions ORDER BY time DESC LIMIT 100;

-- Query data in the table
SELECT time_bucket('15 minutes', time) AS fifteen_min,
    location, COUNT(*),
    MAX(temperature) AS max_temp,
    MAX(humidity) AS max_hum
  FROM conditions
  WHERE time > NOW() - interval '3 hours'
  GROUP BY fifteen_min, location
  ORDER BY fifteen_min DESC, max_temp DESC;
```

There are lots more tutorials within the [TimescaleDB docs](https://docs.timescale.com/timescaledb/latest/getting-started/create-hypertable/#chunks-and-hypertables).

## Modified files

If you would like to make your own fork, we changed these files in the [repo](https://github.com/fly-apps/postgres-ha):

#### fly.toml

As mentioned above, we updated the app's name and the value of `PRIMARY_REGION`:

```toml
app = "postgres-ha-timescaledb"

PRIMARY_REGION = "lhr"
```

#### Dockerfile

We installed the TimescaleDB extension based on the [Timescale docs](https://docs.timescale.com/install/latest/self-hosted/installation-debian/#install-self-hosted-timescaledb-on-debian-based-systems):

```sh
# timescale https://docs.timescale.com/install/latest/self-hosted/installation-debian/#install-self-hosted-timescaledb-on-debian-based-systems
RUN apt-get install --no-install-recommends -y gnupg postgresql-common apt-transport-https lsb-release wget curl
RUN echo "deb https://packagecloud.io/timescale/timescaledb/debian/ $(lsb_release -c -s) main" > /etc/apt/sources.list.d/timescaledb.list && \
    wget --quiet -O - https://packagecloud.io/timescale/timescaledb/gpgkey | apt-key add - && \
    apt-get update && \
    apt-get install -y timescaledb-2-postgresql-$PG_MAJOR timescaledb-tools
```

#### pkg/flypg/config.go

The config values in this file _appear_ to only apply during the launch (since then a spec file does not exist and one needs creating, based on these values). As such, this change is likely not needed if you have already run `fly pg create`. However in case a database has not yet been created, we added the TimescaleDB extension here:

```go
PGParameters: map[string]string{
    ...
    "shared_preload_libraries":        "timescaledb",
}
```

## Connecting

Fly apps within the same organization can connect to your Postgres using the following URI:

```
postgres://postgres:<operator_password>@<postgres-app-name>.internal:5432/<database-name>
```

### Connecting from your local machine (without Wireguard)

1. Postgres needs to be installed on your local machine.

2. Forward the server port to your local system with [`flyctl proxy`](https://fly.io/docs/flyctl/proxy/):

```
flyctl proxy 5432 -a <postgres-app-name>
```

3. Use psql to connect to your Postgres instance on the forwarded port using localhost:

```
psql postgres://postgres:<operator_password>@localhost:5432
```

### Connecting from your local machine (with Wireguard)

1. Postgres needs to be installed on your local machine.

2. With [WireGuard](https://fly.io/docs/reference/private-networking/#install-your-wireguard-app) installed and configured, you can simply use psql to connect to your Postgres instance using its `.internal` hostname (as if you were a Fly app within the private network):

```
psql postgres://postgres:your-pg-password-here@your-pg-app-name-here.internal:5432
```

## Having trouble?

Create an issue or ask a question here: https://community.fly.io/

