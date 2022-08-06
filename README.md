# JDBC Prometheus Exporter

Prometheus exporter for IBM i and other databases. This provides an interface for passive metrics collection. That is, Prometheus can scrape this exporter for metrics.

Any JDBC driver may be used as a data source for Prometheus. The metrics are customizable. 
Any numeric metric available through SQL can be monitored using this client!

This exporter was built for and tested on IBM i as a way to monitor IBM i system and application status through SQL.
As such, its out-of-box-experience is centered around the IBM i use case.

# Installation and Startup

1. Download the latest `prom-client-ibmi.jar` file from
[the releases page](https://github.com/ThePrez/Prom-client-IBMi/releases).
Place this file the filesystem somewhere. 
1. From a command line, `cd` to the directory where you placed
`prom-client-ibmi.jar` and run:
```bash
java -jar prom-client-ibmi.jar
```
Create a default configuration file by responding `y` to the
following prompt:
```bash
Configuration file config.json not found. Would you like to initialize one with defaults? [y] 
```
You Should see a series of messages about collectors being registered. If you see the
following message, the client is running successfully:
```
==============================================================
Successfully started Prometheus client on port 9853
==============================================================
```

If you're not running on IBM i, you'll want to kill the program and modify
`config.json` to have reasonable values for your needs. 

## Using another JDBC driver

The IBM i JDBC driver is bundled with this exporter. 
If you'd like to use a different JDBC driver, you will need to specify the necessary options
in the configuration (namely, `driver_class` and `driver_uri`). You will also need to
explicitly add that driver to the class path. For instance:

```
java -cp prom-client-ibmi.jar:myjdbcdriver.jar com.ibm.jesseg.prometheus.MainApp
```

## Using the `nohup` utility
If you would like to run the program in the background so that you can exit
your shell and keep the Prometheus client running, you can use the `nohup` utility:
```bash
nohup java -jar prom-client-ibmi.jar > prom-client.log 2>&1
```

# Running on a different port

The Prometheus client port can be customized in several ways. The port
is determined by the following, in order of precedence:
- The `PORT` environment variable
- The `promclient.port` Java system property
- The `port` value of the JSON configuration file
- The default value of 9853

# Prometheus Configuration

To configure Prometheus, just add a target to the `scrape_configs` as done
in the following sample configuration:

```yaml
scrape_configs:
  - job_name: 'prometheusibmi'
    static_configs:
    - targets: ['1.2.3.4:9853/metrics']
```

# JSON Configuration

See [config.json](./config.json) for an example JSON file, which contains the
following:
```json
{
  "port": 9853,

  "queries": [{
      "name": "System Statistics",
      "interval": 60,
      "enabled": true,
      "prefix": "STATS",
      "sql": "SELECT * FROM TABLE(QSYS2.SYSTEM_STATUS(RESET_STATISTICS=>'YES',DETAILED_INFO=>'ALL')) X"
    },
    {
      "name": "System Activity",
      "interval": 20,
      "prefix": "SYSACT",
      "include_hostname": true,
      "enabled": false,
      "sql": "SELECT * FROM TABLE(QSYS2.SYSTEM_ACTIVITY_INFO())"
    },
    {
      "name": "number of remote connections",
      "interval": 60,
      "sql": "select COUNT(REMOTE_ADDRESS) as REMOTE_CONNECTIONS from qsys2.netstat_info where TCP_STATE = 'ESTABLISHED' AND REMOTE_ADDRESS != '::1' AND REMOTE_ADDRESS != '127.0.0.1'"
    },
    {
      "name": "Memory Pool Info",
      "interval": 100,
      "multi_row": true,
      "prefix": "MEMPOOL",
      "sql": "SELECT POOL_NAME,CURRENT_SIZE,DEFINED_SIZE,MAXIMUM_ACTIVE_THREADS,CURRENT_THREADS,RESERVED_SIZE FROM TABLE(QSYS2.MEMORY_POOL(RESET_STATISTICS=>'YES')) X"
    }
  ]
}
```

Notes about the JSON configuration file:
- The program will create the default version for you if you don't create one yourself
- The location of the configuration file can be customized by the `promclient.config` Java system property
- The default configuration gathers metrics with just two queries. You can customize your metrics collection with any SQL query you'd like to monitor with prometheus.
- For each query, the `interval` value represents the interval between data collection attempts
for that query, in seconds.

## Valid values for JSON configuration

| Key name           | Type     | required? | Description                                      |
| ------------------ | -------- | ----------| -------------------------------------------------|
| `driver_class`     | String   | no        | The JDBC driver class (default: "com.ibm.as400.access.AS400JDBCDriver") |
| `driver_uri`       | String   | no        | The JDBC connection string (default: "jdbc:as400://localhost")    | 
| `hostname`         | String   | no        | The hostname of the system to connect to (default: localhost)      | 
| `username`         | String   | no        | Username to be used for the connection | 
| `password`         | String   | no        | Password for the connection. **NOT SECURE** |
| `queries`          | array    | yes       | Array of elemnts specifying which SQL queries to run |  

**NOTE: You may be prompted for any needed values (for instance, a password) at the command line if not specified in the JSON configuration**


### `queries` element

The `queries` element contains an array. Each element in the array can have the following values:
| Key name           | Type     | required? | Description                                      |
| ------------------ | -------- | ----------| -------------------------------------------------|
| `sql`              | String   | yes       | The SQL query                                    |
| `name`             | String   | no        | A human-readable name for the query              | 
| `interval`         | Integer  | yes       | The interval to wait between queries             | 
| `prefix`           | String   | no        | A prefix to be used in the Prometheus gauge name | 
| `include_hostname` | boolean  | no        | Whether to include the hostname in the Prometheus gauge name (default: true) |
| `enabled`          | boolean  | no        | Whether this SQL query is enabled (default: true) |
| `multi_row`        | boolean  | no        | Whether to enable multi-row mode (default: false) |



**IMPORTANT NOTES ABOUT COLLECTED METRICS**
- Only numeric values will be collected
- The values will be reported to Prometheus in the format
```
hostname__prefix_identifier
```
(where `hostname` is the IBM i self-resolved hostname and `column` is the SQL column)
- You can tailor the metric name in prometheus by changing the column name via the SQL query (using the SELECT `AS XXXX` syntax)
- The `prefix` is only included if specified in the configuration
- The `hostname` can be excluded via configuration
- The `identifier` is the column name when running in single-row mode

# Collection modes

## Single-row mode

The default behavior for processing query output is single-row mode. If feasible, this is the recommended
way to collect metrics. 
In single-row mode, only the first row of results are processed, but the gauge names can be computed up front,
since the identifier for the gauge is simply the column name. 


## Multi-row mode

Multi-row mode allows you to collect metrics from multiple rows of a JDBC query. When using multi-row mode,
the value of the first column in each result is used to formulate a gauge name each time the query is run. 
This has negative implications if the result set data does not have a consistent value in the first column.

# Managing with Service Commander (IBM i only)

First, install Service Commander (package name `service-commander`)
version 1.5.1 or later. 

Then, from a command line, `cd` to the directory where you placed
`prom-client-ibmi.jar` and run:
```bash
java -jar prom-client-ibmi.jar sc
```
Follow the on-screen instructions. A `prometheus.yml` file will be created.
Do not delete this file.

The default configuration adds a `prometheus` to the `autostart` group
to be launched automatically at IPL.


# Installation and Startup (off IBM i)

This Prometheus client can be run on a different platform and connect remotely to
IBM i to gather statistics. Currently, however, only one remote system at a time
is supported. 

To enable this, populate `username`, `hostname`, and (optionally) `password` in
the `config.json` file that is generated upon initial startup.

For instance:

```json
  "username": "myuser",
  "hostname": "systemname",
  "password": "mypassword"
```

Note, however, that putting your password in a plaintext file is not recommended. 
Instead, configure the username and hostname. You will be prompted for the password
at runtime. 

For instance:

```json
  "username": "myuser",
  "hostname": "systemname"
```


(documentation forthcoming)

# Metrics gathered with default config (IBM i)

- TOTAL_JOBS_IN_SYSTEM
- MAXIMUM_JOBS_IN_SYSTEM
- ACTIVE_JOBS_IN_SYSTEM
- INTERACTIVE_JOBS_IN_SYSTEM
- ELAPSED_TIME
- ELAPSED_CPU_USED
- ELAPSED_CPU_SHARED
- ELAPSED_CPU_UNCAPPED_CAPACITY
- CONFIGURED_CPUS
- CURRENT_CPU_CAPACITY
- AVERAGE_CPU_RATE
- AVERAGE_CPU_UTILIZATION
- AVERAGE_CPU_RATE_REAL
- AVERAGE_CPU_UTILIZATION_REAL
- MINIMUM_CPU_UTILIZATION
- MAXIMUM_CPU_UTILIZATION
- SQL_CPU_UTILIZATION
- MAIN_STORAGE_SIZE
- SYSTEM_ASP_STORAGE
- TOTAL_AUXILIARY_STORAGE
- SYSTEM_ASP_USED
- CURRENT_TEMPORARY_STORAGE
- MAXIMUM_TEMPORARY_STORAGE_USED
- PERMANENT_ADDRESS_RATE
- TEMPORARY_ADDRESS_RATE
- TEMPORARY_256MB_SEGMENTS
- TEMPORARY_4GB_SEGMENTS
- PERMANENT_256MB_SEGMENTS
- PERMANENT_4GB_SEGMENTS
- TEMPORARY_JOB_STRUCTURES_AVAILABLE
- PERMANENT_JOB_STRUCTURES_AVAILABLE
- TOTAL_JOB_TABLE_ENTRIES
- AVAILABLE_JOB_TABLE_ENTRIES
- IN_USE_JOB_TABLE_ENTRIES
- ACTIVE_JOB_TABLE_ENTRIES
- JOBQ_JOB_TABLE_ENTRIES
- OUTQ_JOB_TABLE_ENTRIES
- JOBLOG_PENDING_JOB_TABLE_ENTRIES
- PARTITION_ID
- NUMBER_OF_PARTITIONS
- ACTIVE_THREADS_IN_SYSTEM
- PARTITION_GROUP_ID
- SHARED_PROCESSOR_POOL_ID
- DEFINED_MEMORY
- MINIMUM_MEMORY
- MAXIMUM_MEMORY
- MEMORY_INCREMENT
- PHYSICAL_PROCESSORS
- PHYSICAL_PROCESSORS_SHARED_POOL
- MAXIMUM_PHYSICAL_PROCESSORS
- DEFINED_VIRTUAL_PROCESSORS
- VIRTUAL_PROCESSORS
- MINIMUM_VIRTUAL_PROCESSORS
- MAXIMUM_VIRTUAL_PROCESSORS
- DEFINED_PROCESSING_CAPACITY
- PROCESSING_CAPACITY
- UNALLOCATED_PROCESSING_CAPACITY
- MINIMUM_REQUIRED_PROCESSING_CAPACITY
- MAXIMUM_LICENSED_PROCESSING_CAPACITY
- MINIMUM_PROCESSING_CAPACITY
- MAXIMUM_PROCESSING_CAPACITY
- PROCESSING_CAPACITY_INCREMENT
- DEFINED_INTERACTIVE_CAPACITY
- INTERACTIVE_CAPACITY
- INTERACTIVE_THRESHOLD
- UNALLOCATED_INTERACTIVE_CAPACITY
- MINIMUM_INTERACTIVE_CAPACITY
- MAXIMUM_INTERACTIVE_CAPACITY
- DEFINED_VARIABLE_CAPACITY_WEIGHT
- VARIABLE_CAPACITY_WEIGHT
- UNALLOCATED_VARIABLE_CAPACITY_WEIGHT
- THREADS_PER_PROCESSOR
- DISPATCH_LATENCY
- DISPATCH_WHEEL_ROTATION_TIME
- TOTAL_CPU_TIME
- INTERACTIVE_CPU_TIME
- INTERACTIVE_CPU_TIME_ABOVE_THRESHOLD
- UNUSED_CPU_TIME_SHARED_POOL
- JOURNAL_RECOVERY_COUNT
- JOURNAL_CACHE_WAIT_TIME
- REMOTE_CONNECTIONS

# Sample screenshot (visualization w/Grafana)

![image](https://user-images.githubusercontent.com/17914061/180038306-30724eae-83b2-42c3-b6d5-da2e9b239a25.png)
