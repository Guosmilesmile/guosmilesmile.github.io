---
title: Flink sql client 尝试和增加提交指定集群功能
date: 2020-05-04 18:19:12
tags:
categories:
	- Flink
---

笔者想要做一个实时计算平台，通过sql来减少开发，如果直接用jar包开发，是可以做精细化管理，但是开发速度就会受到限制。

目前有两种方式，一种是做成一个jar包，sql放入配置文件中，但是这样就没办法很好用上社区的功能，限制太大。另一个是使用flink-sql-client。


### 集群构建依赖

```
flink-connector-kafka_2.11-1.10.0.jar
flink-connector-kafka-base_2.11-1.10.0.jar
flink-csv-1.10.0.jar(需要json改json)
kafka-clients-2.2.0.jar
flink-sql-connector-kafka_2.11-1.10.0.jar
```


### 尝试

修改配置conf/flink-conf.yaml

```
jobmanager.rpc.address: flink-jobmanager.flink.svc.cluster.local

# The RPC port where the JobManager is reachable.

jobmanager.rpc.port: 6123

```

配置上对应集群的address和端口6123.


或者修改源码，将集群目标改为可配置。见后面的内容。（由于笔者的集群是flink on kubernetes，社区目前对flink on kubernetes 的native需要在1.11才会有比较好的功能。flink的sql gate way暂时也没有开发，得等社区）


配置对应的yaml文件

sql-client-defaults.yaml

```
################################################################################
#  Licensed to the Apache Software Foundation (ASF) under one
#  or more contributor license agreements.  See the NOTICE file
#  distributed with this work for additional information
#  regarding copyright ownership.  The ASF licenses this file
#  to you under the Apache License, Version 2.0 (the
#  "License"); you may not use this file except in compliance
#  with the License.  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
################################################################################


# This file defines the default environment for Flink's SQL Client.
# Defaults might be overwritten by a session specific environment.


# See the Table API & SQL documentation for details about supported properties.


#==============================================================================
# Tables
#==============================================================================

# Define tables here such as sources, sinks, views, or temporal tables.

#tables: [] # empty list
# A typical table source definition looks like:
# - name: ...
#   type: source-table
#   connector: ...
#   format: ...
#   schema: ...
tables :
  - name: MyTableSource
    type: "source"
    update-mode: append
    format:
      type: csv
    schema:
    - name: "index"
      type: "INT"
    - name: "procTime"
      type: "TIMESTAMP"
      proctime: true
    connector:
      type: kafka
      version: "universal"
      topic: test
      startup-mode: earliest-offset
      properties:
        bootstrap.servers: kafka-host:51600
        zookeeper.connect: zk-host:7072/chroot/kafka
  - name: TableSink
    type: sink-table
    update-mode: append
    format:
      type: csv
    schema:
    - name: "index"
      type: "INT"
    - name: "procTime"
      type: "TIMESTAMP"
    connector:
      type: kafka
      version: "universal"
      topic: test1
      sink-partitioner: "round-robin"
      properties:
        bootstrap.servers: kafka-host:51600
        zookeeper.connect: zk-host:7072/chroot/kafka


# A typical view definition looks like:
# - name: ...
#   type: view
#   query: "SELECT ..."

# A typical temporal table definition looks like:
# - name: ...
#   type: temporal-table
#   history-table: ...
#   time-attribute: ...
#   primary-key: ...


#==============================================================================
# User-defined functions
#==============================================================================

# Define scalar, aggregate, or table functions here.

functions: [] # empty list
# A typical function definition looks like:
# - name: ...
#   from: class
#   class: ...
#   constructor: ...


#==============================================================================
# Catalogs
#==============================================================================

# Define catalogs here.

catalogs: [] # empty list
# A typical catalog definition looks like:
#  - name: myhive
#    type: hive
#    hive-conf-dir: /opt/hive_conf/
#    default-database: ...

#==============================================================================
# Modules
#==============================================================================

# Define modules here.

#modules: # note the following modules will be of the order they are specified
#  - name: core
#    type: core

#==============================================================================
# Execution properties
#==============================================================================

# Properties that change the fundamental execution behavior of a table program.

execution:
  # select the implementation responsible for planning table programs
  # possible values are 'blink' (used by default) or 'old'
  planner: blink
  # 'batch' or 'streaming' execution
  type: streaming
  # allow 'event-time' or only 'processing-time' in sources
  time-characteristic: event-time
  # interval in ms for emitting periodic watermarks
  periodic-watermarks-interval: 200
  # 'changelog' or 'table' presentation of results
  result-mode: table
  # maximum number of maintained rows in 'table' presentation of results
  max-table-result-rows: 1000000
  # parallelism of the program
  parallelism: 1
  # maximum parallelism
  max-parallelism: 128
  # minimum idle state retention in ms
  min-idle-state-retention: 0
  # maximum idle state retention in ms
  max-idle-state-retention: 0
  # current catalog ('default_catalog' by default)
  current-catalog: default_catalog
  # current database of the current catalog (default database of the catalog by default)
  current-database: default_database
  # controls how table programs are restarted in case of a failures
  restart-strategy:
    # strategy type
    # possible values are "fixed-delay", "failure-rate", "none", or "fallback" (default)
    type: fallback

#==============================================================================
# Configuration options
#==============================================================================

# Configuration options for adjusting and tuning table programs.

# A full list of options and their default values can be found
# on the dedicated "Configuration" web page.

# A configuration can look like:
# configuration:
#   table.exec.spill-compression.enabled: true
#   table.exec.spill-compression.block-size: 128kb
#   table.optimizer.join-reorder-enabled: true

#==============================================================================
# Deployment properties
#==============================================================================

# Properties that describe the cluster to which table programs are submitted to.

deployment:
  # general cluster communication timeout in ms
  response-timeout: 5000
  # (optional) address from cluster to gateway
  gateway-address: ""
  # (optional) port from cluster to gateway
  gateway-port: 0

```


### 运行sql-client

```
./bin/sql-client.sh embedded -d sql-client-defaults.yaml
```

```
Flink SQL> show tables;
MyTableSource
TableSink

```

就可以通过各种sql运行了。



### flink sql 大状态的清理

```
min-idle-state-retention
max-idle-state-retention
```


这里需要注意一点，默认情况下 StreamQueryConfig 的设置并不是全局的。因此当设置了清理周期以后，需要在 StreamTableEnvironment 类调用 toAppendStream 或 toRetractStream 将 Table 转为 DataStream 时，显式传入这个 QueryConfig 对象作为参数，才可以令该功能生效。

```java
DataStream<Row> stream = tableEnv.toAppendStream(result, Row.class, qConfig);
```
### 直接提交sql，不等待提示

在CLI会话中设置的属性(例如使用set命令)具有最高的优先级:
```
CLI commands > session environment file > defaults environment file
```


```
./bin/sql-client.sh embedded -m 10.99.212.160:8081 -e /usr/local/flink/flink-1.10.0-sql/sql-client-defaults.yaml -u "insert into TableSink select * from MyTableSource;"

```

```
[root@sq-hdp-master-1 flink-1.10.0-sql]# ./bin/sql-client.sh embedded -m 10.104.155.82:8081 -e /usr/local/flink/flink-1.10.0-sql/sql-client-defaults.yaml -u "insert into TableSink select * from MyTableSource;"
No default environment specified.
Searching for '/usr/local/flink/flink-1.10.0-sql/conf/sql-client-defaults.yaml'...found.
Reading default environment from: file:/usr/local/flink/flink-1.10.0-sql/conf/sql-client-defaults.yaml
Reading session environment from: file:/usr/local/flink/flink-1.10.0-sql/sql-client-defaults.yaml

[INFO] Executing the following statement:
insert into TableSink select * from MyTableSource;
[INFO] Submitting SQL update statement to the cluster...
[INFO] Table update statement has been successfully submitted to the cluster:
Job ID: 420bbc382d32f473539eaa9703f3667e


Shutting down the session...
done.

```

### 修改flink-sql-client达到可以提交到远程集群


CliOptionsParser.java

```java

	public static CliOptions parseEmbeddedModeClient(String[] args) {
		try {
			DefaultParser parser = new DefaultParser();
			CommandLine line = parser.parse(EMBEDDED_MODE_CLIENT_OPTIONS, args, true);
			return new CliOptions(
				line.hasOption(CliOptionsParser.OPTION_HELP.getOpt()),
				checkSessionId(line),
				checkUrl(line, CliOptionsParser.OPTION_ENVIRONMENT),
				checkUrl(line, CliOptionsParser.OPTION_DEFAULTS),
				checkUrls(line, CliOptionsParser.OPTION_JAR),
				checkUrls(line, CliOptionsParser.OPTION_LIBRARY),
				line.getOptionValue(CliOptionsParser.OPTION_UPDATE.getOpt()),
				// 新增jobManager和port的读取
				line.getOptionValue(CliOptionsParser.OPTION_JOBMANAGER_AND_PORT.getOpt())
			);
		} catch (ParseException e) {
			throw new SqlClientException(e.getMessage());
		}
	}
```

CliOptions.java


```java

private String jobmanagerAndPort; // 新增属性

补充对应的构造函数和get set方法。



```


SqlClient.java


```java
private void start() {

.....

            // 修改executor的构造方法
            final Executor executor;
			if (StringUtils.isBlank(this.options.getJobmanagerAndPort())) {
				executor = new LocalExecutor(this.options.getDefaults(), jars, libDirs);
			} else {
				executor = new LocalExecutor(this.options.getDefaults(), jars, libDirs, this.options.getJobmanagerAndPort());
			}

....



}

```

LocalExecutor.java

window调试需要在系统变量增加FLINK_CONF_DIR，重启电脑。

```java
新增构造函数

public LocalExecutor(URL defaultEnv, List<URL> jars, List<URL> libraries, String jobmanagerAndPort) {
		// discover configuration
		final String flinkConfigDir;
		try {
			// find the configuration directory
			flinkConfigDir = CliFrontend.getConfigurationDirectoryFromEnv();

			// load the global configuration
			this.flinkConfig = GlobalConfiguration.loadConfiguration(flinkConfigDir);
			InetSocketAddress address = NetUtils.parseHostPortAddress(jobmanagerAndPort);
			// 主要是在配置文件覆盖jobManager 参数
			this.flinkConfig.setString(JobManagerOptions.ADDRESS, address.getHostString());
			this.flinkConfig.setInteger(JobManagerOptions.PORT, address.getPort());
			// initialize default file system
			FileSystem.initialize(flinkConfig, PluginUtils.createPluginManagerFromRootFolder(flinkConfig));

			// load command lines for deployment
			this.commandLines = CliFrontend.loadCustomCommandLines(flinkConfig, flinkConfigDir);
			this.commandLineOptions = collectCommandLineOptions(commandLines);
		} catch (Exception e) {
			throw new SqlClientException("Could not load Flink configuration.", e);
		}

		// try to find a default environment
		if (defaultEnv == null) {
			final String defaultFilePath = flinkConfigDir + "/" + DEFAULT_ENV_FILE;
			System.out.println("No default environment specified.");
			System.out.print("Searching for '" + defaultFilePath + "'...");
			final File file = new File(defaultFilePath);
			if (file.exists()) {
				System.out.println("found.");
				try {
					defaultEnv = Path.fromLocalFile(file).toUri().toURL();
				} catch (MalformedURLException e) {
					throw new SqlClientException(e);
				}
				LOG.info("Using default environment file: {}", defaultEnv);
			} else {
				System.out.println("not found.");
			}
		}

		// inform user
		if (defaultEnv != null) {
			System.out.println("Reading default environment from: " + defaultEnv);
			try {
				defaultEnvironment = Environment.parse(defaultEnv);
			} catch (IOException e) {
				throw new SqlClientException("Could not read default environment file at: " + defaultEnv, e);
			}
		} else {
			defaultEnvironment = new Environment();
		}
		this.contextMap = new ConcurrentHashMap<>();

		// discover dependencies
		dependencies = discoverDependencies(jars, libraries);

		// prepare result store
		resultStore = new ResultStore(flinkConfig);

		clusterClientServiceLoader = new DefaultClusterClientServiceLoader();
	}


```

然后打包flink-sql-client_2.11-1.10-SNAPSHOT.jar，替换opt下的该文件。搞定！
就可以通过-m提交作业。



### Reference
https://cloud.tencent.com/developer/article/1452854