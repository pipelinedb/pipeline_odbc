# pipeline_odbc

PipelineDB ODBC extension, mainly for PipelineDB `<->` Vertica integration

# Building

You'll need to be able to build PipelineDB extensions, so you'll need to have PipelineDB installed on your system. Building the `odbc_fdw` extension utlimately uses `pg_config`, so PipelineDB's binary directory is on your path. Next, build and install the extension:

    make
    make install

# Quickstart

First created a DSN for connecting to Vertica from `odbc_fdw`. For example:

```
cat <<- EOF > /etc/odbcinst.ini
[VerticaDriver]
Description = HP Vertica ODBC Driver
Driver = /opt/vertica/lib64/libverticaodbc.so
EOF
```

```
cat <<- EOF > /etc/odbc.ini
[ODBC Data Sources]
Vertica = vertica database on HP Vertica

[Vertica]
Description = vertica database on HP Vertica
Driver = VerticaDriver
Database = vertica
Servername = localhost
UID = dbadmin
PWD = vertica
Port = 5433
Locale = en_GB

[ODBC]
Threading = 1
EOF
```

Next, create a table in Vertica to read from within PipelineDB:

    vertica=> CREATE TABLE vertica_test (x integer, y integer, z integer);
    CREATE TABLE
    vertica=> INSERT INTO vertica_test (x, y, z) VALUES (0, 0, 0);
    OUTPUT 
    --------
      1
    (1 row)

Now connect to PipelineDB with `pipeline`. First we'll need to create the `odbc_fdw` extension:

    pipeline=# CREATE EXTENSION odbc_fdw;
    CREATE EXTENSION
    
Next, point PipelineDB to the Vertica cluster via `odbc_fdw` (note that this uses the DSN from `/etc/odbc.ini`):
    
    pipeline=# CREATE SERVER vertica FOREIGN DATA WRAPPER odbc_fdw OPTIONS (dsn 'Vertica');
    CREATE SERVER
    
Now we can map the Vertica table to a PipelineDB table via the Vertica foreign server:

    pipeline=# CREATE FOREIGN TABLE vertica_test (x integer, y integer, z integer) SERVER vertica \
    OPTIONS (database 'vertica', table 'vertica_test');
    

The final thing we need to do is map the PipelineDB user to a Vertica user:

    pipeline=# CREATE USER MAPPING FOR <pipelinedb user> SERVER vertica \
    OPTIONS (username '<vertica user>', password '<vertica password>');
    
We can now interact with Vertica via PipelineDB as if the Vertica table were a native PipelineDB relation:

    pipeline=# SELECT * FROM vertica_test;
     x | y | z 
    ---+---+---
     0 | 0 | 0
    (1 row)

This means that the Vertica table can be used in a stream-table join with a continuous view:

    pipeline=# CREATE CONTINUOUS VIEW v0 AS SELECT x::integer, COUNT(*) \
    stream s JOIN vertica_test t ON s.x = t.x GROUP BY s.z;
    
Insert some rows that will join the row we previously inserted into Vertica:

    pipeline=# INSERT INTO stream (x) SELECT 0 AS x FROM generate_series(1, 1000);
    INSERT 1000
    
And verify that the continuous view in PipelineDB properly joined the Vertica relation on the stream:

    pipeline=# SELECT * FROM v0;
     x | count 
    ---+------
     0 | 1000
    (1 row)



    
