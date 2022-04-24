# postgres-ha-timescaledb

This repo adds the TimescaleDB extension to Fly's [highly available Postgres cluster](https://github.com/fly-apps/postgres-ha).

## Motivation

If you need to store any kind of metrics, it's important to store that data efficiently and quickly query it. The [TimescaleDB](https://github.com/timescale/timescaledb) PostgreSQL extension is ideal for this. It provides automatic partitioning yet retains the standard PostgreSQL interface. Plus it includes additional functions for time-series analysis that are not present in vanilla PostgreSQL (such as the `time_bucket` function).

Since Fly's Postgres clusters are just regular Fly applications, in theory you _could_ launch a new app directly from this repo. However then _you_ would have to do a lot of the work that Fly does for you (create a volume for its data, possibly create a replica for HA, set all the secrets needed ...). So we tell Fly to create the database app, using its provided commands. Then we'll deploy _our_ fork which will install the TimescaleDB extension.

## Deploy

If you haven't already done so, [install the Fly CLI](https://fly.io/docs/getting-started/installing-flyctl/) and then [log in to Fly](https://fly.io/docs/getting-started/log-in-to-fly/).

1. Clone this repo
2. Edit the `fly.toml` in two places: the `app` name, and the value of `PRIMARY_REGION` as the [region](https://fly.io/docs/reference/regions/#fly-io-regions) you plan to use for your database:

    ```toml
    app = "your-pg-name"

    PRIMARY_REGION = "lhr"
    ```
3. Run `fly pg create` to create a new database app. You can choose its region and whether to use HA. It will take a minute to configure and will then show you your credentials.
4. (optional) If you would like to create a multi-region database app, [add read-replicas](https://fly.io/docs/getting-started/multi-region-databases/#create-a-postgresql-cluster).
4. Run `fly deploy` to apply the modifications to install the TimescaleDB extension. It may take a few minutes.
5. Enable the TimescaleDB extension by using a stolon update (stolon controls the cluster):
    ```sh
    fly ssh console
    export $(cat /data/.env | xargs)
    stolonctl update --patch '{"pgParameters": { "shared_preload_libraries": "timescaledb"}}'
    ```
    You might like to confirm that worked by running `stolonctl clusterdata read` and checking you see `"shared_preload_libraries":"timescaledb"` within that large block of JSON.
6. Run `fly restart your-database-name-here` to apply that stolon update to all VMs in the cluster. It may take a minute to complete.
7. (optional) Run  `fly checks list` to confirm all is well: you should see all the checks pass.
8. (optional) Confirm the TimescaleDB extension has been successfully installed by connecting to the database's VM (run `fly ssh console`) and running these two commands:
    ```
    cat /data/postgres/postgresql.conf
    ```
    That should contain this line:
    ```
    shared_preload_libraries = 'timescaledb'`
    ```
    ... and ...
    ```
    dpkg -l | grep -E 'timescale'
    ```
    That should list the TimescaleDB extension:
    ```
    ...
    timescaledb-2-loader-postgresql-14 2.6.1~debian11                 amd64        The loader for TimescaleDB to load individual versions.
    timescaledb-2-postgresql-14        2.6.1~debian11                 amd64        An open-source time-series database based on PostgreSQL, as an extension.
    timescaledb-tools                  0.12.0~debian11                amd64        A suite of tools that can be used with TimescaleDB.
    ```

## Add TimescaleDB to a database

Usually you would attach a Fly app to the PostgreSQL app (by using the `fly pg attach --postgres-app your-database-name-here` command). That creates a new database (with the same name as the app) and provides the app with a secret `DATABASE_URL` which it can use to securely connect to it.

Or you can manually create a database (by using `CREATE DATABASE database-name-here;`).

Either way, once you have a database you will need to [add the TimescaleDB extension](https://docs.timescale.com/install/latest/self-hosted/installation-debian/#set-up-the-timescaledb-extension) to it. For example (using the credentials provided by Fly):

```sh
$ psql postgres://postgres:your-pg-password-here@top1.nearest.of.your-pg-app-name-here.internal:5432
\c your-database-name-here;
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

You can confirm that has worked by using the `\dx` command with psql. You should see something like:

```
                                      List of installed extensions
    Name     | Version |   Schema   |                            Description
-------------+---------+------------+-------------------------------------------------------------------
 plpgsql     | 1.0     | pg_catalog | PL/pgSQL procedural language
 timescaledb | 2.6.1   | public     | Enables scalable inserts and complex queries for time-series data
```

## Using TimescaleDB

You can use standard SQL but can now also create and query hypertables (source: [The Timescale repo](https://github.com/timescale/timescaledb#creating-a-hypertable))

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

If you would like to make your own, we changed these files in the [repo](https://github.com/fly-apps/postgres-ha):

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

### Connecting to Postgres from your local machine (without Wireguard)

1. Postgres needs to be installed on your local machine.

2. If you don't have [WireGuard](https://fly.io/docs/reference/private-networking/#install-your-wireguard-app) installed and configured, forward the server port to your local system with [`flyctl proxy`](https://fly.io/docs/flyctl/proxy/):

```
flyctl proxy 5432 -a <postgres-app-name>
```

3. Use psql to connect to your Postgres instance on the forwarded port using localhost:

```
psql postgres://postgres:<operator_password>@localhost:5432
```

### Connecting to Postgres from your local machine (with Wireguard)

1. Postgres needs to be installed on your local machine.

2. If you do have [WireGuard](https://fly.io/docs/reference/private-networking/#install-your-wireguard-app) installed and configured, you can simply use psql to connect to your Postgres instance using the `.internal` hostname (as if you were an app within the private network):

```
psql postgres://postgres:your-pg-password-here@top1.nearest.of.your-pg-app-name-here.internal:5432
```

## Having trouble?

Create an issue or ask a question here: https://community.fly.io/

