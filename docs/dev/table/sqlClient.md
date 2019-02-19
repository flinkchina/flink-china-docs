---
title: "SQL客户端"
nav-parent_id: tableapi
nav-pos: 100
is_beta: true
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->


Flink的Table＆SQL API可以处理用SQL语言编写的查询，但这些查询需要嵌入用Java或Scala编写的表程序中。 而且，这些程序需要在提交到集群之前使用构建工具打包。 这或多或少地限制了Flink对Java / Scala程序员的使用。

*SQL Client*旨在提供一种简单的方法来编写，调试和提交表程序到Flink集群，而无需单行Java或Scala代码。 *SQL Client CLI*允许在命令行上检索和可视化运行的分布式应用程序的实时结果。

<a href="{{ site.baseurl }}/fig/sql_client_demo.gif"><img class="offset" src="{{ site.baseurl }}/fig/sql_client_demo.gif" alt="动态演示Flink SQL客户端CLI在集群上运行表程序" width="80%" /></a>

<span class="label label-danger">注意</span> The SQL Client is in an early development phase. Even though the application is not production-ready yet, it can be a quite useful tool for prototyping and playing around with Flink SQL. In the future, the community plans to extend its functionality by providing a REST-based [SQL Client Gateway](sqlClient.html#limitations--future).
SQL客户端处于早期开发阶段。 即使应用程序还没有生产就绪，但它可以是一个非常有用的工具，用于原型设计和使用Flink SQL。 在未来，社区计划通过提供基于REST的[SQL客户端网关](sqlClient.html#limitations--future)来扩展其功能。

* This will be replaced by the TOC
{:toc}

开始
---------------

This section describes how to setup and run your first Flink SQL program from the command-line.

The SQL Client is bundled in the regular Flink distribution and thus runnable out-of-the-box. It requires only a running Flink cluster where table programs can be executed. For more information about setting up a Flink cluster see the [Cluster & Deployment]({{ site.baseurl }}/ops/deployment/cluster_setup.html) part. If you simply want to try out the SQL Client, you can also start a local cluster with one worker using the following command:

{% highlight bash %}
./bin/start-cluster.sh
{% endhighlight %}

### 启动SQL客户端CLI

The SQL Client scripts are also located in the binary directory of Flink. [In the future](sqlClient.html#limitations--future), a user will have two possibilities of starting the SQL Client CLI either by starting an embedded standalone process or by connecting to a remote SQL Client Gateway. At the moment only the `embedded` mode is supported. You can start the CLI by calling:

{% highlight bash %}
./bin/sql-client.sh embedded
{% endhighlight %}

By default, the SQL Client will read its configuration from the environment file located in `./conf/sql-client-defaults.yaml`. See the [configuration part](sqlClient.html#environment-files) for more information about the structure of environment files.

### 运行SQL查询

Once the CLI has been started, you can use the `HELP` command to list all available SQL statements. For validating your setup and cluster connection, you can enter your first SQL query and press the `Enter` key to execute it:

{% highlight sql %}
SELECT 'Hello World';
{% endhighlight %}

This query requires no table source and produces a single row result. The CLI will retrieve results from the cluster and visualize them. You can close the result view by pressing the `Q` key.

The CLI supports **two modes** for maintaining and visualizing results.

The **table mode** materializes results in memory and visualizes them in a regular, paginated table representation. It can be enabled by executing the following command in the CLI:

{% highlight text %}
SET execution.result-mode=table;
{% endhighlight %}

The **changelog mode** does not materialize results and visualizes the result stream that is produced by a [continuous query](streaming/dynamic_tables.html#continuous-queries) consisting of insertions (`+`) and retractions (`-`).

{% highlight text %}
SET execution.result-mode=changelog;
{% endhighlight %}

You can use the following query to see both result modes in action:

{% highlight sql %}
SELECT name, COUNT(*) AS cnt FROM (VALUES ('Bob'), ('Alice'), ('Greg'), ('Bob')) AS NameTable(name) GROUP BY name;
{% endhighlight %}

This query performs a bounded word count example.

In *changelog mode*, the visualized changelog should be similar to:

{% highlight text %}
+ Bob, 1
+ Alice, 1
+ Greg, 1
- Bob, 1
+ Bob, 2
{% endhighlight %}

In *table mode*, the visualized result table is continuously updated until the table program ends with:

{% highlight text %}
Bob, 2
Alice, 1
Greg, 1
{% endhighlight %}

Both result modes can be useful during the prototyping of SQL queries. In both modes, results are stored in the Java heap memory of the SQL Client. In order to keep the CLI interface responsive, the changelog mode only shows the latest 1000 changes. The table mode allows for navigating through bigger results that are only limited by the available main memory and the configured [maximum number of rows](sqlClient.html#configuration) (`max-table-result-rows`).

<span class="label label-danger">Attention</span> Queries that are executed in a batch environment, can only be retrieved using the `table` result mode.

After a query is defined, it can be submitted to the cluster as a long-running, detached Flink job. For this, a target system that stores the results needs to be specified using the [INSERT INTO statement](sqlClient.html#detached-sql-queries). The [configuration section](sqlClient.html#configuration) explains how to declare table sources for reading data, how to declare table sinks for writing data, and how to configure other table program properties.

{% top %}

配置
-------------

The SQL Client can be started with the following optional CLI commands. They are discussed in detail in the subsequent paragraphs.

{% highlight text %}
./bin/sql-client.sh embedded --help

Mode "embedded" submits Flink jobs from the local machine.

  Syntax: embedded [OPTIONS]
  "embedded" mode options:
     -d,--defaults <environment file>      The environment properties with which
                                           every new session is initialized.
                                           Properties might be overwritten by
                                           session properties.
     -e,--environment <environment file>   The environment properties to be
                                           imported into the session. It might
                                           overwrite default environment
                                           properties.
     -h,--help                             Show the help message with
                                           descriptions of all options.
     -j,--jar <JAR file>                   A JAR file to be imported into the
                                           session. The file might contain
                                           user-defined classes needed for the
                                           execution of statements such as
                                           functions, table sources, or sinks.
                                           Can be used multiple times.
     -l,--library <JAR directory>          A JAR file directory with which every
                                           new session is initialized. The files
                                           might contain user-defined classes
                                           needed for the execution of
                                           statements such as functions, table
                                           sources, or sinks. Can be used
                                           multiple times.
     -s,--session <session identifier>     The identifier for a session.
                                           'default' is the default identifier.
{% endhighlight %}

{% top %}

### 环境文件

A SQL query needs a configuration environment in which it is executed. The so-called *environment files* define available table sources and sinks, external catalogs, user-defined functions, and other properties required for execution and deployment.

Every environment file is a regular [YAML file](http://yaml.org/). An example of such a file is presented below.

{% highlight yaml %}
# Define tables here such as sources, sinks, views, or temporal tables.

tables:
  - name: MyTableSource
    type: source-table
    update-mode: append
    connector:
      type: filesystem
      path: "/path/to/something.csv"
    format:
      type: csv
      fields:
        - name: MyField1
          type: INT
        - name: MyField2
          type: VARCHAR
      line-delimiter: "\n"
      comment-prefix: "#"
    schema:
      - name: MyField1
        type: INT
      - name: MyField2
        type: VARCHAR
  - name: MyCustomView
    type: view
    query: "SELECT MyField2 FROM MyTableSource"

# 在这里定义用户定义的函数。

functions:
  - name: myUDF
    from: class
    class: foo.bar.AggregateUDF
    constructor:
      - 7.6
      - false

# 执行允许更改表程序行为的属性。
execution:
  type: streaming                   # required: execution mode either 'batch' or 'streaming'
  result-mode: table                # required: either 'table' or 'changelog'
  max-table-result-rows: 1000000    # optional: maximum number of maintained rows in
                                    #   'table' mode (1000000 by default, smaller 1 means unlimited)
  time-characteristic: event-time   # optional: 'processing-time' or 'event-time' (default)
  parallelism: 1                    # optional: Flink's parallelism (1 by default)
  periodic-watermarks-interval: 200 # optional: interval for periodic watermarks (200 ms by default)
  max-parallelism: 16               # optional: Flink's maximum parallelism (128 by default)
  min-idle-state-retention: 0       # optional: table program's minimum idle state time
  max-idle-state-retention: 0       # optional: table program's maximum idle state time
  restart-strategy:                 # optional: restart strategy
    type: fallback                  #   "fallback" to global restart strategy by default

# 部署允许描述表程序提交到的集群的属性


deployment:
  response-timeout: 5000
{% endhighlight %}

This configuration:

- defines an environment with a table source `MyTableSource` that reads from a CSV file,
- defines a view `MyCustomView` that declares a virtual table using a SQL query,
- defines a user-defined function `myUDF` that can be instantiated using the class name and two constructor parameters,
- specifies a parallelism of 1 for queries executed in this streaming environment,
- specifies an event-time characteristic, and
- runs queries in the `table` result mode.

Depending on the use case, a configuration can be split into multiple files. Therefore, environment files can be created for general purposes (*defaults environment file* using `--defaults`) as well as on a per-session basis (*session environment file* using `--environment`). Every CLI session is initialized with the default properties followed by the session properties. For example, the defaults environment file could specify all table sources that should be available for querying in every session whereas the session environment file only declares a specific state retention time and parallelism. Both default and session environment files can be passed when starting the CLI application. If no default environment file has been specified, the SQL Client searches for `./conf/sql-client-defaults.yaml` in Flink's configuration directory.

<span class="label label-danger">Attention</span> Properties that have been set within a CLI session (e.g. using the `SET` command) have highest precedence:

{% highlight text %}
CLI commands > session environment file > defaults environment file
{% endhighlight %}

#### 重启策略

Restart strategies control how Flink jobs are restarted in case of a failure. Similar to [global restart strategies]({{ site.baseurl }}/dev/restart_strategies.html) for a Flink cluster, a more fine-grained restart configuration can be declared in an environment file.

The following strategies are supported:

{% highlight yaml %}
execution:
  # falls back to the global strategy defined in flink-conf.yaml
  restart-strategy:
    type: fallback

  # job fails directly and no restart is attempted
  restart-strategy:
    type: none

  # attempts a given number of times to restart the job
  restart-strategy:
    type: fixed-delay
    attempts: 3      # retries before job is declared as failed (default: Integer.MAX_VALUE)
    delay: 10000     # delay in ms between retries (default: 10 s)

  # attempts as long as the maximum number of failures per time interval is not exceeded
  restart-strategy:
    type: failure-rate
    max-failures-per-interval: 1   # retries in interval until failing (default: 1)
    failure-rate-interval: 60000   # measuring interval in ms for failure rate
    delay: 10000                   # delay in ms between retries (default: 10 s)
{% endhighlight %}

{% top %}

### 依赖

The SQL Client does not require to setup a Java project using Maven or SBT. Instead, you can pass the dependencies as regular JAR files that get submitted to the cluster. You can either specify each JAR file separately (using `--jar`) or define entire library directories (using `--library`). For connectors to external systems (such as Apache Kafka) and corresponding data formats (such as JSON), Flink provides **ready-to-use JAR bundles**. These JAR files can be downloaded for each release from the Maven central repository.

The full list of offered SQL JARs and documentation about how to use them can be found on the [connection to external systems page](connect.html).

The following example shows an environment file that defines a table source reading JSON data from Apache Kafka.

{% highlight yaml %}
tables:
  - name: TaxiRides
    type: source-table
    update-mode: append
    connector:
      property-version: 1
      type: kafka
      version: "0.11"
      topic: TaxiRides
      startup-mode: earliest-offset
      properties:
        - key: zookeeper.connect
          value: localhost:2181
        - key: bootstrap.servers
          value: localhost:9092
        - key: group.id
          value: testGroup
    format:
      property-version: 1
      type: json
      schema: "ROW<rideId LONG, lon FLOAT, lat FLOAT, rideTime TIMESTAMP>"
    schema:
      - name: rideId
        type: LONG
      - name: lon
        type: FLOAT
      - name: lat
        type: FLOAT
      - name: rowTime
        type: TIMESTAMP
        rowtime:
          timestamps:
            type: "from-field"
            from: "rideTime"
          watermarks:
            type: "periodic-bounded"
            delay: "60000"
      - name: procTime
        type: TIMESTAMP
        proctime: true
{% endhighlight %}

The resulting schema of the `TaxiRide` table contains most of the fields of the JSON schema. Furthermore, it adds a rowtime attribute `rowTime` and processing-time attribute `procTime`.

Both `connector` and `format` allow to define a property version (which is currently version `1`) for future backwards compatibility.

{% top %}

### 用户定义的函数

The SQL Client allows users to create custom, user-defined functions to be used in SQL queries. Currently, these functions are restricted to be defined programmatically in Java/Scala classes.

In order to provide a user-defined function, you need to first implement and compile a function class that extends `ScalarFunction`, `AggregateFunction` or `TableFunction` (see [User-defined Functions]({{ site.baseurl }}/dev/table/udfs.html)). One or more functions can then be packaged into a dependency JAR for the SQL Client.

All functions must be declared in an environment file before being called. For each item in the list of `functions`, one must specify

- a `name` under which the function is registered,
- the source of the function using `from` (restricted to be `class` for now),
- the `class` which indicates the fully qualified class name of the function and an optional list of `constructor` parameters for instantiation.

{% highlight yaml %}
functions:
  - name: ...               # required: name of the function
    from: class             # required: source of the function (can only be "class" for now)
    class: ...              # required: fully qualified class name of the function
    constructor:            # optimal: constructor parameters of the function class
      - ...                 # optimal: a literal parameter with implicit type
      - class: ...          # optimal: full class name of the parameter
        constructor:        # optimal: constructor parameters of the parameter's class
          - type: ...       # optimal: type of the literal parameter
            value: ...      # optimal: value of the literal parameter
{% endhighlight %}

Make sure that the order and types of the specified parameters strictly match one of the constructors of your function class.

#### 构造器参数

Depending on the user-defined function, it might be necessary to parameterize the implementation before using it in SQL statements.

As shown in the example before, when declaring a user-defined function, a class can be configured using constructor parameters in one of the following three ways:

**A literal value with implicit type:** The SQL Client will automatically derive the type according to the literal value itself. Currently, only values of `BOOLEAN`, `INT`, `DOUBLE` and `VARCHAR` are supported here.
If the automatic derivation does not work as expected (e.g., you need a VARCHAR `false`), use explicit types instead.

{% highlight yaml %}
- true         # -> BOOLEAN (case sensitive)
- 42           # -> INT
- 1234.222     # -> DOUBLE
- foo          # -> VARCHAR
{% endhighlight %}

**A literal value with explicit type:** Explicitly declare the parameter with `type` and `value` properties for type-safety.

{% highlight yaml %}
- type: DECIMAL
  value: 11111111111111111
{% endhighlight %}

The table below illustrates the supported Java parameter types and the corresponding SQL type strings.

| Java type               |  SQL type         |
| :---------------------- | :---------------- |
| `java.math.BigDecimal`  | `DECIMAL`         |
| `java.lang.Boolean`     | `BOOLEAN`         |
| `java.lang.Byte`        | `TINYINT`         |
| `java.lang.Double`      | `DOUBLE`          |
| `java.lang.Float`       | `REAL`, `FLOAT`   |
| `java.lang.Integer`     | `INTEGER`, `INT`  |
| `java.lang.Long`        | `BIGINT`          |
| `java.lang.Short`       | `SMALLINT`        |
| `java.lang.String`      | `VARCHAR`         |

More types (e.g., `TIMESTAMP` or `ARRAY`), primitive types, and `null` are not supported yet.

**A (nested) class instance:** Besides literal values, you can also create (nested) class instances for constructor parameters by specifying the `class` and `constructor` properties.
This process can be recursively performed until all the constructor parameters are represented with literal values.

{% highlight yaml %}
- class: foo.bar.paramClass
  constructor:
    - StarryName
    - class: java.lang.Integer
      constructor:
        - class: java.lang.String
          constructor:
            - type: VARCHAR
              value: 3
{% endhighlight %}

{% top %}

分离的SQL查询
--------------------

In order to define end-to-end SQL pipelines, SQL's `INSERT INTO` statement can be used for submitting long-running, detached queries to a Flink cluster. These queries produce their results into an external system instead of the SQL Client. This allows for dealing with higher parallelism and larger amounts of data. The CLI itself does not have any control over a detached query after submission.

{% highlight sql %}
INSERT INTO MyTableSink SELECT * FROM MyTableSource
{% endhighlight %}

The table sink `MyTableSink` has to be declared in the environment file. See the [connection page](connect.html) for more information about supported external systems and their configuration. An example for an Apache Kafka table sink is shown below.

{% highlight yaml %}
tables:
  - name: MyTableSink
    type: sink-table
    update-mode: append
    connector:
      property-version: 1
      type: kafka
      version: "0.11"
      topic: OutputTopic
      properties:
        - key: zookeeper.connect
          value: localhost:2181
        - key: bootstrap.servers
          value: localhost:9092
        - key: group.id
          value: testGroup
    format:
      property-version: 1
      type: json
      derive-schema: true
    schema:
      - name: rideId
        type: LONG
      - name: lon
        type: FLOAT
      - name: lat
        type: FLOAT
      - name: rideTime
        type: TIMESTAMP
{% endhighlight %}

The SQL Client makes sure that a statement is successfully submitted to the cluster. Once the query is submitted, the CLI will show information about the Flink job.

{% highlight text %}
[INFO] Table update statement has been successfully submitted to the cluster:
Cluster ID: StandaloneClusterId
Job ID: 6f922fe5cba87406ff23ae4a7bb79044
Web interface: http://localhost:8081
{% endhighlight %}

<span class="label label-danger">Attention</span> The SQL Client does not track the status of the running Flink job after submission. The CLI process can be shutdown after the submission without affecting the detached query. Flink's [restart strategy]({{ site.baseurl }}/dev/restart_strategies.html) takes care of the fault-tolerance. A query can be cancelled using Flink's web interface, command-line, or REST API.

{% top %}

SQL视图
---------

Views allow to define virtual tables from SQL queries. The view definition is parsed and validated immediately. However, the actual execution happens when the view is accessed during the submission of a general `INSERT INTO` or `SELECT` statement.

Views can either be defined in [environment files](sqlClient.html#environment-files) or within the CLI session.

The following example shows how to define multiple views in a file. The views are registered in the order in which they are defined in the environment file. Reference chains such as _view A depends on view B depends on view C_ are supported.

{% highlight yaml %}
tables:
  - name: MyTableSource
    # ...
  - name: MyRestrictedView
    type: view
    query: "SELECT MyField2 FROM MyTableSource"
  - name: MyComplexView
    type: view
    query: >
      SELECT MyField2 + 42, CAST(MyField1 AS VARCHAR)
      FROM MyTableSource
      WHERE MyField2 > 200
{% endhighlight %}

Similar to table sources and sinks, views defined in a session environment file have highest precedence.

Views can also be created within a CLI session using the `CREATE VIEW` statement:

{% highlight text %}
CREATE VIEW MyNewView AS SELECT MyField2 FROM MyTableSource;
{% endhighlight %}

Views created within a CLI session can also be removed again using the `DROP VIEW` statement:

{% highlight text %}
DROP VIEW MyNewView;
{% endhighlight %}

<span class="label label-danger">Attention</span> The definition of views in the CLI is limited to the mentioned syntax above. Defining a schema for views or escaping whitespaces in table names will be supported in future versions.

{% top %}

时态表
---------------

A [temporal table](./streaming/temporal_tables.html) allows for a (parameterized) view on a changing history table that returns the content of a table at a specific point in time. This is especially useful for joining a table with the content of another table at a particular timestamp. More information can be found in the [temporal table joins](./streaming/joins.html#join-with-a-temporal-table) page.

The following example shows how to define a temporal table `SourceTemporalTable`:

{% highlight yaml %}
tables:

  # Define the table source (or view) that contains updates to a temporal table
  - name: HistorySource
    type: source-table
    update-mode: append
    connector: # ...
    format: # ...
    schema:
      - name: integerField
        type: INT
      - name: stringField
        type: VARCHAR
      - name: rowtimeField
        type: TIMESTAMP
        rowtime:
          timestamps:
            type: from-field
            from: rowtimeField
          watermarks:
            type: from-source

  # Define a temporal table over the changing history table with time attribute and primary key
  - name: SourceTemporalTable
    type: temporal-table
    history-table: HistorySource
    primary-key: integerField
    time-attribute: rowtimeField  # could also be a proctime field
{% endhighlight %}

As shown in the example, definitions of table sources, views, and temporal tables can be mixed with each other. They are registered in the order in which they are defined in the environment file. For example, a temporal table can reference a view which can depend on another view or table source.

{% top %}

局限与未来
--------------------

The current SQL Client implementation is in a very early development stage and might change in the future as part of the bigger Flink Improvement Proposal 24 ([FLIP-24](https://cwiki.apache.org/confluence/display/FLINK/FLIP-24+-+SQL+Client)). Feel free to join the discussion and open issue about bugs and features that you find useful.

{% top %}
