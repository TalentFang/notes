### 配置示例
```xml
<?xml version="1.0"?>

<!--

NOTE: User and query level settings are set up in "users.xml" file.

-->

<yandex>

<logger>

<!-- Possible levels: https://github.com/pocoproject/poco/blob/develop/Foundation/include/Poco/Logger.h#L105 -->

<level>warning</level>

<log>/var/log/clickhouse-server/clickhouse-server.log</log>

<errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>

<size>1000M</size>

<count>10</count>

<!-- <console>1</console> --> <!-- Default behavior is autodetection (log to console if not daemon mode and is tty) -->

</logger>

<!--display_name>production</display_name--> <!-- It is the name that will be shown in the client -->

<http_port>8123</http_port>

<tcp_port>9002</tcp_port>

<mysql_port>9003</mysql_port>

  

<!-- For HTTPS and SSL over native protocol. -->

<!--

<https_port>8443</https_port>

<tcp_port_secure>9440</tcp_port_secure>

-->

  

<!-- Used with https_port and tcp_port_secure. Full ssl options list: https://github.com/ClickHouse-Extras/poco/blob/master/NetSSL_OpenSSL/include/Poco/Net/SSLManager.h#L71 -->

<openSSL>

<server> <!-- Used for https server AND secure tcp port -->

<!-- openssl req -subj "/CN=localhost" -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/clickhouse-server/server.key -out /etc/clickhouse-server/server.crt -->

<certificateFile>/etc/clickhouse-server/server.crt</certificateFile>

<privateKeyFile>/etc/clickhouse-server/server.key</privateKeyFile>

<!-- openssl dhparam -out /etc/clickhouse-server/dhparam.pem 4096 -->

<dhParamsFile>/etc/clickhouse-server/dhparam.pem</dhParamsFile>

<verificationMode>none</verificationMode>

<loadDefaultCAFile>true</loadDefaultCAFile>

<cacheSessions>true</cacheSessions>

<disableProtocols>sslv2,sslv3</disableProtocols>

<preferServerCiphers>true</preferServerCiphers>

</server>

  

<client> <!-- Used for connecting to https dictionary source -->

<loadDefaultCAFile>true</loadDefaultCAFile>

<cacheSessions>true</cacheSessions>

<disableProtocols>sslv2,sslv3</disableProtocols>

<preferServerCiphers>true</preferServerCiphers>

<!-- Use for self-signed: <verificationMode>none</verificationMode> -->

<invalidCertificateHandler>

<!-- Use for self-signed: <name>AcceptCertificateHandler</name> -->

<name>RejectCertificateHandler</name>

</invalidCertificateHandler>

</client>

</openSSL>

  

<!-- Default root page on http[s] server. For example load UI from https://tabix.io/ when opening http://localhost:8123 -->

<http_server_default_response><![CDATA[<html ng-app="SMI2"><head><base href="http://ui.tabix.io/"></head><body><div ui-view="" class="content-ui"></div><script src="http://loader.tabix.io/master.js"></script></body></html>]]></http_server_default_response>

  

<!-- Port for communication between replicas. Used for data exchange. -->

<interserver_http_port>9009</interserver_http_port>

  

<!-- Hostname that is used by other replicas to request this server.

If not specified, than it is determined analoguous to 'hostname -f' command.

This setting could be used to switch replication to another network interface.

-->

<!--

<interserver_http_host>example.yandex.ru</interserver_http_host>

-->

  

<!-- Listen specified host. use :: (wildcard IPv6 address), if you want to accept connections both with IPv4 and IPv6 from everywhere. -->

<!-- <listen_host>::</listen_host> -->

<!-- Same for hosts with disabled ipv6: -->

<listen_host>0.0.0.0</listen_host>

  

<!-- Default values - try listen localhost on ipv4 and ipv6: -->

<!--

<listen_host>::1</listen_host>

<listen_host>127.0.0.1</listen_host>

-->

<!-- Don't exit if ipv6 or ipv4 unavailable, but listen_host with this protocol specified -->

<!-- <listen_try>0</listen_try> -->

  

<!-- Allow listen on same address:port -->

<!-- <listen_reuse_port>0</listen_reuse_port> -->

  

<!-- <listen_backlog>64</listen_backlog> -->

  

<max_connections>1024</max_connections>

<keep_alive_timeout>300</keep_alive_timeout>

  

<!-- Maximum number of concurrent queries. -->

<max_concurrent_queries>128</max_concurrent_queries>

  

<!-- Set limit on number of open files (default: maximum). This setting makes sense on Mac OS X because getrlimit() fails to retrieve

correct maximum value. -->

<!-- <max_open_files>262144</max_open_files> -->

  

<!-- Size of cache of uncompressed blocks of data, used in tables of MergeTree family.

In bytes. Cache is single for server. Memory is allocated only on demand.

Cache is used when 'use_uncompressed_cache' user setting turned on (off by default).

Uncompressed cache is advantageous only for very short queries and in rare cases.

-->

<uncompressed_cache_size>8589934592</uncompressed_cache_size>

  

<!-- Approximate size of mark cache, used in tables of MergeTree family.

In bytes. Cache is single for server. Memory is allocated only on demand.

You should not lower this value.

-->

<mark_cache_size>5368709120</mark_cache_size>

  
  

<!-- Path to data directory, with trailing slash. -->

<path>{{ CK_SERVER_DATA_PATH | default('/data') }}/comm/clickhouse/</path>

  

<!-- Path to temporary data for processing hard queries. -->

<tmp_path>{{ CK_SERVER_DATA_PATH | default('/data') }}/comm/clickhouse/tmp/</tmp_path>

  

<!-- Directory with user provided files that are accessible by 'file' table function. -->

<user_files_path>{{ CK_SERVER_DATA_PATH | default('/data') }}/comm/clickhouse/user_files/</user_files_path>

  

<!-- Path to configuration file with users, access rights, profiles of settings, quotas. -->

<users_config>users.xml</users_config>

  

<!-- Default profile of settings. -->

<default_profile>default</default_profile>

  

<!-- System profile of settings. This settings are used by internal processes (Buffer storage, Distibuted DDL worker and so on). -->

<!-- <system_profile>default</system_profile> -->

  

<!-- Default database. -->

<default_database>default</default_database>

  

<!-- Server time zone could be set here.

  

Time zone is used when converting between String and DateTime types,

when printing DateTime in text formats and parsing DateTime from text,

it is used in date and time related functions, if specific time zone was not passed as an argument.

  

Time zone is specified as identifier from IANA time zone database, like UTC or Africa/Abidjan.

If not specified, system time zone at server startup is used.

  

Please note, that server could display time zone alias instead of specified name.

Example: W-SU is an alias for Europe/Moscow and Zulu is an alias for UTC.

-->

<!-- <timezone>Europe/Moscow</timezone> -->

  

<!-- You can specify umask here (see "man umask"). Server will apply it on startup.

Number is always parsed as octal. Default umask is 027 (other users cannot read logs, data files, etc; group can only read).

-->

<!-- <umask>022</umask> -->

  

<!-- Perform mlockall after startup to lower first queries latency

and to prevent clickhouse executable from being paged out under high IO load.

Enabling this option is recommended but will lead to increased startup time for up to a few seconds.

-->

<mlock_executable>false</mlock_executable>

  

<!-- Configuration of clusters that could be used in Distributed tables.

https://clickhouse.yandex/docs/en/table_engines/distributed/

-->

<remote_servers incl="clickhouse_remote_servers" >

<!-- Test only shard config for testing distributed storage -->

<test_shard_localhost>

<shard>

<replica>

<host>localhost</host>

<port>9000</port>

</replica>

</shard>

</test_shard_localhost>

<test_cluster_two_shards_localhost>

<shard>

<replica>

<host>localhost</host>

<port>9000</port>

</replica>

</shard>

<shard>

<replica>

<host>localhost</host>

<port>9000</port>

</replica>

</shard>

</test_cluster_two_shards_localhost>

<test_cluster_two_shards>

<shard>

<replica>

<host>127.0.0.1</host>

<port>9000</port>

</replica>

</shard>

<shard>

<replica>

<host>127.0.0.2</host>

<port>9000</port>

</replica>

</shard>

</test_cluster_two_shards>

<test_shard_localhost_secure>

<shard>

<replica>

<host>localhost</host>

<port>9440</port>

<secure>1</secure>

</replica>

</shard>

</test_shard_localhost_secure>

<test_unavailable_shard>

<shard>

<replica>

<host>localhost</host>

<port>9000</port>

</replica>

</shard>

<shard>

<replica>

<host>localhost</host>

<port>1</port>

</replica>

</shard>

</test_unavailable_shard>

</remote_servers>

  

<include_from>/etc/clickhouse-server/metrika.xml</include_from>

  

<!-- If element has 'incl' attribute, then for it's value will be used corresponding substitution from another file.

By default, path to file with substitutions is /etc/metrika.xml. It could be changed in config in 'include_from' element.

Values for substitutions are specified in /yandex/name_of_substitution elements in that file.

-->

  

<!-- ZooKeeper is used to store metadata about replicas, when using Replicated tables.

Optional. If you don't use replicated tables, you could omit that.

  

See https://clickhouse.yandex/docs/en/table_engines/replication/

-->

  

<zookeeper incl="zookeeper-servers" optional="true" />

  

<!-- Substitutions for parameters of replicated tables.

Optional. If you don't use replicated tables, you could omit that.

  

See https://clickhouse.yandex/docs/en/table_engines/replication/#creating-replicated-tables

-->

<macros incl="macros" optional="true" />

  
  

<!-- Reloading interval for embedded dictionaries, in seconds. Default: 3600. -->

<builtin_dictionaries_reload_interval>3600</builtin_dictionaries_reload_interval>

  
  

<!-- Maximum session timeout, in seconds. Default: 3600. -->

<max_session_timeout>3600</max_session_timeout>

  

<!-- Default session timeout, in seconds. Default: 60. -->

<default_session_timeout>60</default_session_timeout>

  

<!-- Sending data to Graphite for monitoring. Several sections can be defined. -->

<!--

interval - send every X second

root_path - prefix for keys

hostname_in_path - append hostname to root_path (default = true)

metrics - send data from table system.metrics

events - send data from table system.events

asynchronous_metrics - send data from table system.asynchronous_metrics

-->

<!--

<graphite>

<host>localhost</host>

<port>42000</port>

<timeout>0.1</timeout>

<interval>60</interval>

<root_path>one_min</root_path>

<hostname_in_path>true</hostname_in_path>

  

<metrics>true</metrics>

<events>true</events>

<events_cumulative>false</events_cumulative>

<asynchronous_metrics>true</asynchronous_metrics>

</graphite>

<graphite>

<host>localhost</host>

<port>42000</port>

<timeout>0.1</timeout>

<interval>1</interval>

<root_path>one_sec</root_path>

  

<metrics>true</metrics>

<events>true</events>

<events_cumulative>false</events_cumulative>

<asynchronous_metrics>false</asynchronous_metrics>

</graphite>

-->

  
  

<!-- Query log. Used only for queries with setting log_queries = 1. -->

<query_log>

<!-- What table to insert data. If table is not exist, it will be created.

When query log structure is changed after system update,

then old table will be renamed and new table will be created automatically.

-->

<database>system</database>

<table>query_log</table>

<engine>

ENGINE = MergeTree

PARTITION BY toYYYYMM(event_time)

ORDER BY event_time

TTL event_time + INTERVAL 7 DAY DELETE

</engine>

<flush_interval_milliseconds>7500</flush_interval_milliseconds>

</query_log>

  

<!-- Trace log. Stores stack traces collected by query profilers.

See query_profiler_real_time_period_ns and query_profiler_cpu_time_period_ns settings.

<trace_log>

<database>system</database>

<table>trace_log</table>

<engine>

ENGINE = MergeTree

ORDER BY (event_time, event_time_microseconds)

TTL event_time + INTERVAL 4 DAY DELETE

</engine>

<flush_interval_milliseconds>7500</flush_interval_milliseconds>

</trace_log>

-->

  

<!-- Query thread log. Has information about all threads participated in query execution.

Used only for queries with setting log_query_threads = 1. -->

<query_thread_log>

<database>system</database>

<table>query_thread_log</table>

<partition_by>toYYYYMM(event_date)</partition_by>

<flush_interval_milliseconds>7500</flush_interval_milliseconds>

</query_thread_log>

  

<!-- Uncomment if use part log.

Part log contains information about all actions with parts in MergeTree tables (creation, deletion, merges, downloads).

<part_log>

<database>system</database>

<table>part_log</table>

<flush_interval_milliseconds>7500</flush_interval_milliseconds>

</part_log>

-->

  

<!-- Uncomment to write text log into table.

Text log contains all information from usual server log but stores it in structured and efficient way.

<text_log>

<database>system</database>

<table>text_log</table>

<flush_interval_milliseconds>7500</flush_interval_milliseconds>

</text_log>

-->

  

<!-- Uncomment to write metric log into table.

Metric log contains rows with current values of ProfileEvents, CurrentMetrics collected with "collect_interval_milliseconds" interval.

<metric_log>

<database>system</database>

<table>metric_log</table>

<flush_interval_milliseconds>7500</flush_interval_milliseconds>

<collect_interval_milliseconds>1000</collect_interval_milliseconds>

</metric_log>

-->

  

<!-- Parameters for embedded dictionaries, used in Yandex.Metrica.

See https://clickhouse.yandex/docs/en/dicts/internal_dicts/

-->

  

<!-- Path to file with region hierarchy. -->

<!-- <path_to_regions_hierarchy_file>/opt/geo/regions_hierarchy.txt</path_to_regions_hierarchy_file> -->

  

<!-- Path to directory with files containing names of regions -->

<!-- <path_to_regions_names_files>/opt/geo/</path_to_regions_names_files> -->

  
  

<!-- Configuration of external dictionaries. See:

https://clickhouse.yandex/docs/en/dicts/external_dicts/

-->

<dictionaries_config>*_dictionary.xml</dictionaries_config>

  

<!-- Uncomment if you want data to be compressed 30-100% better.

Don't do that if you just started using ClickHouse.

-->

<compression incl="clickhouse_compression">

<!--

<!- - Set of variants. Checked in order. Last matching case wins. If nothing matches, lz4 will be used. - ->

<case>

  

<!- - Conditions. All must be satisfied. Some conditions may be omitted. - ->

<min_part_size>10000000000</min_part_size> <!- - Min part size in bytes. - ->

<min_part_size_ratio>0.01</min_part_size_ratio> <!- - Min size of part relative to whole table size. - ->

  

<!- - What compression method to use. - ->

<method>zstd</method>

</case>

-->

</compression>

  

<!-- Allow to execute distributed DDL queries (CREATE, DROP, ALTER, RENAME) on cluster.

Works only if ZooKeeper is enabled. Comment it if such functionality isn't required. -->

<distributed_ddl>

<!-- Path in ZooKeeper to queue with DDL queries -->

<path>/clickhouse/task_queue/ddl</path>

  

<!-- Settings from this profile will be used to execute DDL queries -->

<!-- <profile>default</profile> -->

</distributed_ddl>

  

<!-- Settings to fine tune MergeTree tables. See documentation in source code, in MergeTreeSettings.h -->

<!--

<merge_tree>

<max_suspicious_broken_parts>5</max_suspicious_broken_parts>

</merge_tree>

-->

  

<!-- Protection from accidental DROP.

If size of a MergeTree table is greater than max_table_size_to_drop (in bytes) than table could not be dropped with any DROP query.

If you want do delete one table and don't want to restart clickhouse-server, you could create special file <clickhouse-path>/flags/force_drop_table and make DROP once.

By default max_table_size_to_drop is 50GB; max_table_size_to_drop=0 allows to DROP any tables.

The same for max_partition_size_to_drop.

Uncomment to disable protection.

-->

<!-- <max_table_size_to_drop>0</max_table_size_to_drop> -->

<!-- <max_partition_size_to_drop>0</max_partition_size_to_drop> -->

  

<!-- Example of parameters for GraphiteMergeTree table engine -->

<graphite_rollup_example>

<pattern>

<regexp>click_cost</regexp>

<function>any</function>

<retention>

<age>0</age>

<precision>3600</precision>

</retention>

<retention>

<age>86400</age>

<precision>60</precision>

</retention>

</pattern>

<default>

<function>max</function>

<retention>

<age>0</age>

<precision>60</precision>

</retention>

<retention>

<age>3600</age>

<precision>300</precision>

</retention>

<retention>

<age>86400</age>

<precision>3600</precision>

</retention>

</default>

</graphite_rollup_example>

  

<!-- Directory in <clickhouse-path> containing schema files for various input formats.

The directory will be created if it doesn't exist.

-->

<format_schema_path>/var/lib/clickhouse/format_schemas/</format_schema_path>

  
  

<!-- Uncomment to use query masking rules.

name - name for the rule (optional)

regexp - RE2 compatible regular expression (mandatory)

replace - substitution string for sensitive data (optional, by default - six asterisks)

<query_masking_rules>

<rule>

<name>hide SSN</name>

<regexp>\b\d{3}-\d{2}-\d{4}\b</regexp>

<replace>000-00-0000</replace>

</rule>

</query_masking_rules>

-->

  

<!-- Uncomment to disable ClickHouse internal DNS caching. -->

<!-- <disable_internal_dns_cache>1</disable_internal_dns_cache> -->

<shutdown_wait_unfinished>60</shutdown_wait_unfinished>

<shutdown_wait_unfinished_queries>false</shutdown_wait_unfinished_queries>

<shutdown_wait_backups_and_restores>false</shutdown_wait_backups_and_restores>

<async_insert_queue_flush_on_shutdown>false</async_insert_queue_flush_on_shutdown>

<prometheus>

<endpoint>/metrics</endpoint>

<port>9363</port>

<metrics>true</metrics>

<events>true</events>

<asynchronous_metrics>true</asynchronous_metrics>

<status_info>true</status_info>

</prometheus>

</yandex>
```


这份是 ClickHouse Server 的主配置文件，我们快速抓住核心：
### 🔧 基础服务端口
- **`<http_port>8123</http_port>`**：HTTP 接口，用于 REST API 或 UI 工具连接。
- **`<tcp_port>9002</tcp_port>`**：**原生 TCP 端口**，这是 `clickhouse-client` 等主要客户端使用的端口（被改为了 9002，非默认的 9000）。
- **`<mysql_port>9003</mysql_port>`**：MySQL 兼容协议端口，可用 MySQL 客户端连接。
- **`<interserver_http_port>9009</interserver_http_port>`**：副本间内部数据交换与同步的专用端口。

### 📂 数据与日志路径
- **`<path>`**：数据存储的主目录。
- **`<tmp_path>`**：临时数据目录，用于处理复杂查询的中间数据。
- **`<user_files_path>`**：`file()` 表函数可访问的文件目录。
- **`<logger>`**：日志级别 (`warning`)、日志文件路径和滚动策略。

### 🌐 集群与协调配置
- **`<remote_servers>`**：定义了分布式表可用的集群拓扑。里面有几个测试集群示例（如 `test_shard_localhost`）。
- **`<include_from>`**：指定外部配置文件 `metrika.xml`，通常用于存放需要多节点同步的集群、压缩等配置。
- **`<zookeeper>`**：使用了 `incl` 属性，指向外部文件中的 `zookeeper-servers` 配置，这是**副本同步所必需**的协调服务地址。
- **`<macros>`**：同样通过 `incl` 引用外部配置，用于在复制表路径中替换 `{shard}` 和 `{replica}` 占位符。

### ⚙️ 性能与管控
- **`<max_connections>`**：最大客户端连接数 (1024)。
- **`<max_concurrent_queries>`**：最大并发查询数 (128)。
- **`<uncompressed_cache_size>`** 和 **`<mark_cache_size>`**：缓存大小配置。
- **`<distributed_ddl>`**：启用分布式 DDL 执行，需依赖 ZooKeeper/Keeper。
- **`<prometheus>`**：在 9363 端口暴露 Prometheus 格式的监控指标。

### ⚠️ 特别说明
配置中存在 `localhost` 的测试集群和 `metrika.xml` 引用，这表明该配置**很可能是一个由配置管理工具（如 Ansible）渲染的模板**，实际部署时会由外部变量动态填充具体配置。另外，`<listen_host>0.0.0.0</listen_host>` 允许服务监听所有网络接口。