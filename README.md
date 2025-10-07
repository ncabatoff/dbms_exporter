# DBMS Server Exporter

DBMS exporter for PostgresSQL, ODBC, and Sybase (FreeTDS) server metrics.

Supported Postgres versions: 9.2 and up.

Supported Sybase versions: 11 (and possibly up).  This is also the only database
I've tried with ODBC, via freetds.

In principle it should also work for SQL Server (using the driver name
freetds), but I haven't tried.

Similary, any DB driver supported by database/sql can be easily added if desired.
See [postgres.go](db/postgres.go) for an example.

## Quick Start
This package is available for Docker.  If you don't use Docker you'll need to
install some extra dependencies, see [Building](#Building) below for details.

```
# Build the dbms_exporter binary and the ncabatoff/dbms_exporter docker image
make build-with-docker

# Start an example database
docker run --net=host -it --rm -e POSTGRES_PASSWORD=password postgres

# Connect to it
docker run --rm -e DATA_SOURCE_NAME="postgresql://postgres:password@localhost:5432/?sslmode=disable" -p 9113:9113 -v`pwd`:/etc ncabatoff/dbms_exporter -queryfile /etc/postgres.yaml -driver postgres
```

The -driver argument allows working with engines than postgres; currently
the other options are odbc and freetds (for which sybase is an alias).

## Running

There are three required inputs: the driver, the queryfile, and the DSN.

```
DATA_SOURCE_NAME="<connection info>" ./dbms_exporter -driver <driver> -queryfile <path>
```

### PostgreSQL

```
DATA_SOURCE_NAME="postgres://username:password@hostname:port/?sslmode=disable&dbname=postgres&client_encoding=UTF8" \
  ./dbms_exporter -driver postgres -queryfile postgres.yaml 
```

See the [github.com/lib/pq](http://github.com/lib/pq) module for other ways to format the connection string.

### FreeTDS/Sybase

You can use "sybase" as an alias for the freetds driver; it behaves the same
except that metrics start with `sybase_` instead of `freetds_`.

```
DATA_SOURCE_NAME="compatibility_mode=sybase;user=myuser;pwd=mypassword;server=myhostname" \
  ./dbms_exporter -driver freetds -queryfile sybase.yaml 
```

### ODBC/FreeTDS/Sybase

```
DATA_SOURCE_NAME="driver=freetds;TDS_Version=5.0;server=hostname;uid=username;pwd=password;port=7100" \
  ./dbms_exporter -driver odbc -queryfile sybase-short.yaml 
```

### Flags

Name                   | Description
-----------------------|------------
driver                 | DB driver to use, one of odbc, postgres, freetds
dumpmaps               | Do not run, simply dump the queries read from queryfile.
persistent.connections | Only open a DB connection at startup and on failures.
queryfile              | Path to file containing the queries to run.
web.listen-address     | Address to listen on for web interface and telemetry.
web.telemetry-path     | Path under which to expose metrics.

## The metrics config file

The -queryfile command-line argument specifies a YAML file containing the
queries to run.  Some examples are provided in [postgres.yaml](postgres.yaml)
and [sybase.yaml](sybase.yaml).

The YAML file is a map from recipe name to recipe.  A recipe defines query(s)
to run and how to interpret the resulting columns as Prometheus metrics.  The
resulting metrics will be named `driverName_recipeName_columnName`.

Example recipe 1

```
  recipe1:
    query: SELECT lab1, COUNT(*) AS val1, SUM(val2) AS sumval2 
           FROM sometab 
           GROUP BY lab1
    metrics:
      lab1:
        usage: LABEL
      val1:
        usage: GAUGE
        description: help text for val1
      sumval2:
        usage: COUNTER
        description: help text for val2
```

Each row produced by the SELECT will yield metrics `driver_recipe1_val1`
and `driver_recipe1_sumval2` with labels based on the lab1 column.  For
example, given the following resultset returned by the SELECT:

lab1 | val1 | sumval2
-----|------|--------
ABC  |  1   |   2
DEF  |  3   |   4

we'll get these metrics assuming the postgres driver:

```
postgres_recipe1_val1{lab1=ABC} 1
postgres_recipe1_val1{lab1=DEF} 3
postgres_recipe1_sumval2{lab1=ABC} 2
postgres_recipe1_sumval2{lab1=DEF} 4
```

### Metric fields

We've already seen that metrics have usages and descriptions.  The possible
values for the usage field of each metric:

Usage    | Effect
---------|-------
DISCARD  | ignore column, do nothing with it
LABEL    | make column into a label
COUNTER  | create a counter metric from column
GAUGE    | create a gauge metric from column
DURATION | create a guage metric from column, interpreting it as a duration (PostgreSQL specific)
FIXED    | create a constant label based on the YAML config (not based on SQL results)

Description only need be provided for COUNTER, GAUGE, and DURATION, and becomes
the help text of the metric.

COUNTER and GAUGE metrics may provide a `regexp` attribute.  The regular
expression will be applied to the string value that came from the DB, and the
first capture group will then be interpreted as a number.

FIXED metrics must provide a `fixedval` attribute, which specifies the value
for the constant label.

### Multiple Resultsets

As seen above, the simplest case is that there is only a single resultset.  In
some cases there may be more.  For example, in Sybase sp_helpdb can be used to
get information about a DB's size, but it's returned in the form of two
different resultsets.  For this example we're only interested in the second
resultset, which looks like this:

device_fragments | size        | usage     | free kbytes
-----------------|-------------|-----------|------------
master           | 2.0 MB      | data only | 1248
tempdev          | 300.0 MB    | data only | 304816
templog          | 300.0 MB    | log only  | 306880

Example recipe 2

```
  dbsize:
    query: sp_helpdb tempdb
    resultsets:
      - discard:
      - metrics:
        - device_fragments:
            usage: LABEL
        - size:
            usage: GAUGE
            regexp: ^([\d.]+)\s+MB$
            description: size in MB
        - usage:
            usage: LABEL
        - free kbytes:
            usage: GAUGE
            description: free space in KB
```

The above recipe runs sp_helpdb and ignores the first resultset thanks to the
'discard'.  

### Rangeover Resultsets

What if you want to repeat the same query based on some list derived from DB
data?  Let's say for example we want to run sp_spaceused against every DB in
the system.  This requires that we do a USE beforehand.

Example recipe 3

```
  dbsize:
    rangeover: SELECT name AS db_name FROM master.dbo.sysdatabases
    queries: 
	  - USE {{.}}
	  - sp_spaceused
    resultsets:
      - discard:
      - metrics:
        - db_name:
            usage: "LABEL"
        - reserved:
            usage: "GAUGE"
            regexp: ^([\d.]+)\s+KB$
            description: "reserved space in kbytes"
        - data:
            usage: "GAUGE"
            regexp: ^([\d.]+)\s+KB$
            description: "data space in kbytes"
        - index_size:
            usage: "GAUGE"
            regexp: ^([\d.]+)\s+KB$
            description: "index space in kbytes"
        - unused:
            usage: "GAUGE"
            regexp: ^([\d.]+)\s+KB$
            description: "unused space in kbytes"
```

This recipe first runs the rangeover query, which should yield a single column.
For each value returned, the list of queries defined by 'queries:' is executed,
and {{.}} is replaced with the current value.  In addition, a column for the
iteration variable is appended to each result returned (db_name in the above
example.)  The regular sp_spacused output:

```
 database_name                  database_size
 ------------------------------ -------------
 tempdb                         602.0 MB     

 reserved        data            index_size      unused         
 --------------- --------------- --------------- ---------------
 3154 KB         2386 KB         54 KB           714 KB
```

## Building
The default make file behavior is to build the binary:
```
make
```

To build the dockerfile, run `make docker`. 

This will build the docker image as `ncabatoff/dbms_exporter:latest`. This 
is a minimal docker image containing *just* dbms_exporter. By default no SSL 
certificates are included, if you need to use SSL you should either bind-mount 
`/etc/ssl/certs/ca-certificates.crt` or derive a new image containing them.

### make arguments

To build specific drivers, run:
```
make DRIVERS="postgres freetds"
```

The default behaviour is as above, i.e. postgres and freetds drivers are built.

If building ODBC you'll need to disable static linking:
```
make DRIVERS="postgres odbc" LDFLAGS=
```

### FreeTDS

Sybase support depends on FreeTDS.  In Ubuntu the package you want is
freetds-dev, but I've had issues with pre-1.0 FreeTDS, so I recommend building
from source unless your distro has a >=1.0 version.  

Alternatively, if you have docker installed, you can do this by running:
```
make build-with-docker
```

This will build and run a docker image to build FreeTDS and then dbms_exporter,
statically linked.  It will also build a docker image named
ncabatoff:dbms_exporter containing the resulting binary, but the binary is
self-contained and can be used directly as well.

You must ensure that your [freetds.conf](freetds.conf) is configured correctly
and in the right place, e.g. /usr/local/etc/freetds.conf (build from source) or
/etc/freetds/freetds.conf (using Debian/Ubuntu package).  The
ncabatoff:dbms_exporter docker image puts ./freetds.conf to the appropriate
place in the image.  The main things you need to set are the TDS version (5.0)
and the port to connect to your Sybase instance.  For me it's 7100.

### Sybase via FreeTDS alone

This option uses gofreetds to call into FreeTDS directly, without going through
database/sql and ODBC.  This is the recommended option as it allows you to use
multiple resultsets and static linking.

### SQL Server via FreeTDS

It'll probably work but I haven't tried it.  Follow the instructions for Sybase
via FreeTDS above.  Make sure to customize the freetds.conf file to your needs
(e.g. you'll probably want TDS-Version > 5, and a different port than 7100).
Don't put `compatibility_mode=sybase` in your `DATA_SOURCE_NAME`.

### Sybase via ODBC+FreeTDS

This option is largely superceded by the non-ODBC FreeTDS option below.  Using
ODBC requires dynamic linking as far as I can tell, and it also means you're
going through database/sql, which doesn't allow for multiple resultsets.


If you're not running via docker you'll need to install both FreeTDS (discussed
above) and the Linux ODBC libraries:

```
apt-get install tdsodbc
```

You'll also need to configure ODBC using one of the following
commands:

```
# If you installed FreeTDS from source, run
odbcinst -i -d -f /usr/local/etc/odbcinst.ini
# If you installed via packages, run
odbcinst -i -d -f /usr/share/tdsodbc/odbcinst.ini
```

### ODBC with something other than Sybase

It'll probably work but I haven't tried it.  Follow the intructions above for
Sybase via ODBC+FreeTDS, then install your non-Sybase driver.  You'll have to
modify your odbcinst.ini file and `DATA_SOURCE_NAME` appropriately.

### PostgreSQL

#### Running as non-superuser

To be able to collect metrics from `pg_stat_activity` and `pg_stat_replication`
as non-superuser you have to create functions and views to do so.

```sql
CREATE USER postgres_exporter PASSWORD 'password';
ALTER USER postgres_exporter SET SEARCH_PATH TO postgres_exporter,pg_catalog;

CREATE SCHEMA postgres_exporter AUTHORIZATION postgres_exporter;

CREATE FUNCTION postgres_exporter.f_select_pg_stat_activity()
RETURNS setof pg_catalog.pg_stat_activity
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT * from pg_catalog.pg_stat_activity;
$$;

CREATE FUNCTION postgres_exporter.f_select_pg_stat_replication()
RETURNS setof pg_catalog.pg_stat_replication
LANGUAGE sql
SECURITY DEFINER
AS $$
  SELECT * from pg_catalog.pg_stat_replication;
$$;

CREATE VIEW postgres_exporter.pg_stat_replication
AS
  SELECT * FROM postgres_exporter.f_select_pg_stat_replication();

CREATE VIEW postgres_exporter.pg_stat_activity
AS
  SELECT * FROM postgres_exporter.f_select_pg_stat_activity();

GRANT SELECT ON postgres_exporter.pg_stat_replication TO postgres_exporter;
GRANT SELECT ON postgres_exporter.pg_stat_activity TO postgres_exporter;
```

### Pour tester le resultat en local 
curl http://localhost:9113/metrics
