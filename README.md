# Description
MySQL -> embulk -> S3 로 Change Data Capture(CDC)를 S3로 저장하는 예제

![embulk-cdc-from-mysql-to-s3](embulk-cdc-from-mysql-to-s3.svg)

# Embulk Installation
## Linux & Mac & BSD
emublk 버전별로 embulk를 실행할 수 있는 Java 버전이 다르기 때문에 반드시 [Embulk Quick Start](https://github.com/embulk/embulk#quick-start)를 참고해서 필요한 Java SE Runtime Environment (JRE)를 embulk 설치 전에 미리 설치해야 한다.

```
curl --create-dirs -o ~/.embulk/bin/embulk -L "https://dl.embulk.org/embulk-latest.jar"
chmod +x ~/.embulk/bin/embulk
echo 'export PATH="$HOME/.embulk/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Running example
emublk가 잘 설치 되었는지를 아래와 같은 간단한 예제를 실행해서 확인한다.
```
embulk example ./try1
embulk guess   ./try1/seed.yml -o config.yml
embulk preview config.yml
embulk run     config.yml
```

# Incremental Loading from MySQL to S3 using Embulk
## Preparation for sample database
테스트에 사용할 MySQL 데이터베이스와 테이블을 생성하고, 샘플 데이터를 적재한다.

```
$ mycli -u<user_id> -p'<password>' -h <rds host name>
Version: 1.8.1
Chat: https://gitter.im/dbcli/mycli
Mail: https://groups.google.com/forum/#!forum/mycli-users
Home: http://mycli.net
Thanks to the contributor - Ted Pennings
mysql>
mysql> create database test;
mysql> use test;
mysql>

CREATE TABLE pet (
  pkid INT NOT NULL AUTO_INCREMENT,
  name VARCHAR(20),
  owner VARCHAR(20),
  species VARCHAR(20),
  sex CHAR(1),
  birth DATE,
  death DATE,
  c_time DATETIME DEFAULT CURRENT_TIMESTAMP,
  m_time DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

  PRIMARY KEY (pkid),
  KEY (c_time),
  KEY (m_time)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8 COLLATE=utf8_bin;

mysql>
INSERT INTO test.pet (name, `owner`, species, sex, birth, death) VALUES
("Fluffy", "Harold", "cat", "f", "1993-02-04", NULL),
("Claws", "Gwen", "cat", "m", "1904-03-17", NULL),
("Buffy", "Harold", "dog", "f", "1989-05-13", NULL),
("Fang", "Benny", "dog", "m", "1990-08-27", NULL),
("Bowser", "Diane", "dog", "m", "1979-08-31", "1995-07-29"),
("Chirpy", "Gwen", "bird", "f", "1998-09-11", NULL),
("Whistler", "Gwen", "bird", NULL, "1997-12-09", NULL),
("Slim", "Benny", "snake", "m", "1996-04-29", NULL);
```

## Running embulk
### MySQL Input Plugin Installation
[Input plugin](https://plugins.embulk.org/#input) 목록에서 MySQL input-plugin을 찾아서 설치한다.
`embulk gem install <name>`  명령어로 설치하고, `embulk gem list` 명령어로 정상적으로 설치되었는지 확인한다.
```
$ embulk gem install embulk-input-mysql
$ embulk gem list
```

### Out: RDS -> Console
(1) config.yml 파일 작성 (myconfig/config.out-console.yml)
```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  where: "c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'"
out: {type: stdout}
```

(2) run
```
$ embulk run config.yml
2020-06-08 00:53:20.994 +0000: Embulk v0.9.23
2020-06-08 00:53:21.809 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-08 00:53:23.851 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-08 00:53:24.541 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-08 00:53:24.675 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-08 00:53:24.714 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-08 00:53:24.720 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-08 00:53:24.722 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 00:53:24.954 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-08 00:53:24.955 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-08 00:53:24.955 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-08 00:53:24.955 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-08 00:53:25.011 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-08 00:53:25.016 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-08 00:53:25.068 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-08 00:53:25.068 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 00:53:25.078 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth FROM `pet` WHERE c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'
2020-06-10 00:53:25.081 +0000 [INFO] (0015:task-0000): > 0.00 seconds
Fluffy,Harold,cat,f,1993-02-04
Claws,Gwen,cat,m,1904-03-17
2020-06-08 00:53:25.086 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-08 00:53:25.089 +0000 [INFO] (main): Committed.
2020-06-08 00:53:25.089 +0000 [INFO] (main): Next config diff: {"in":{},"out":{}}
```

### Out: RDS -> file
(1) config.yml 파일 작성 (myconfig/config.out-file.yml)
```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  where: "c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'"
out:
  type: file
  path_prefix: ./logs/sample_output_
  file_ext: csv
  formatter:
    type: csv
```

- **주의 사항\!**
  - **out** 섹션에서 **type**은 반드시 필요하고, 없을 경우, 실행시 에러가 발생한다.
  - **file** 로 출력할 경우, **path_prefix** 에 포함되어 있는 디렉터리는 미리 생성해야 한다. 그렇지 않을 경우, 에러가 발생한다.
- **out** 섹션에서 **type** 속성이 없는 경우 발생하는 에러 사례
```
$ embulk run config.yml
2020-06-07 14:00:45.676 +0000: Embulk v0.9.23
2020-06-07 14:00:46.497 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-07 14:00:48.503 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-07 14:00:49.208 +0000 [INFO] (main): Started Embulk v0.9.23
org.embulk.config.ConfigException: Attribute type is required but not set
    at org.embulk.config.DataSourceImpl.get(DataSourceImpl.java:67)
    at org.embulk.exec.BulkLoader$ProcessPluginSet.<init>(BulkLoader.java:425)
    at org.embulk.exec.BulkLoader.doRun(BulkLoader.java:502)
    at org.embulk.exec.BulkLoader.access$000(BulkLoader.java:35)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:353)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:350)
    at org.embulk.spi.Exec.doWith(Exec.java:22)
    at org.embulk.exec.BulkLoader.run(BulkLoader.java:350)
    at org.embulk.EmbulkEmbed.run(EmbulkEmbed.java:242)
    at org.embulk.EmbulkRunner.runInternal(EmbulkRunner.java:291)
    at org.embulk.EmbulkRunner.run(EmbulkRunner.java:155)
    at org.embulk.cli.EmbulkRun.runSubcommand(EmbulkRun.java:431)
    at org.embulk.cli.EmbulkRun.run(EmbulkRun.java:90)
    at org.embulk.cli.Main.main(Main.java:64)

Error: Attribute type is required but not set
```
- **out** 섹션에서 **type:file** 일 때, **path_prefix**에 명시된 출력 디렉터리가 없는 경우 발생하는 에러 사례
```
$ embulk run config.yml
2020-06-07 14:17:44.825 +0000: Embulk v0.9.23
2020-06-07 14:17:45.621 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-07 14:17:47.640 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-07 14:17:48.277 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-07 14:17:48.448 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-07 14:17:48.485 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-07 14:17:48.491 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-07 14:17:48.493 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-07 14:17:48.695 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-07 14:17:48.696 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-07 14:17:48.696 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-07 14:17:48.696 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-07 14:17:48.761 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-07 14:17:48.785 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-07 14:17:48.821 +0000 [INFO] (0015:task-0000): Writing local file './logs/sample_output_000.00.csv'
2020-06-07 14:17:48.821 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
org.embulk.exec.PartialExecutionException: java.lang.RuntimeException: java.io.FileNotFoundException: ./logs/sample_output_000.00.csv (No such file or directory)
    at org.embulk.exec.BulkLoader$LoaderState.buildPartialExecuteException(BulkLoader.java:340)
    at org.embulk.exec.BulkLoader.doRun(BulkLoader.java:566)
    at org.embulk.exec.BulkLoader.access$000(BulkLoader.java:35)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:353)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:350)
    at org.embulk.spi.Exec.doWith(Exec.java:22)
    at org.embulk.exec.BulkLoader.run(BulkLoader.java:350)
    at org.embulk.EmbulkEmbed.run(EmbulkEmbed.java:242)
    at org.embulk.EmbulkRunner.runInternal(EmbulkRunner.java:291)
    at org.embulk.EmbulkRunner.run(EmbulkRunner.java:155)
    at org.embulk.cli.EmbulkRun.runSubcommand(EmbulkRun.java:431)
    at org.embulk.cli.EmbulkRun.run(EmbulkRun.java:90)
    at org.embulk.cli.Main.main(Main.java:64)
Caused by: java.lang.RuntimeException: java.io.FileNotFoundException: ./logs/sample_output_000.00.csv (No such file or directory)
    at org.embulk.standards.LocalFileOutputPlugin$1.nextFile(LocalFileOutputPlugin.java:88)
    at org.embulk.spi.util.FileOutputOutputStream.nextFile(FileOutputOutputStream.java:30)
    at org.embulk.spi.util.LineEncoder.nextFile(LineEncoder.java:84)
    at org.embulk.standards.CsvFormatterPlugin.open(CsvFormatterPlugin.java:104)
    at org.embulk.spi.FileOutputRunner.open(FileOutputRunner.java:127)
    at org.embulk.exec.LocalExecutorPlugin$ScatterTransactionalPageOutput.openOutputs(LocalExecutorPlugin.java:402)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor.runInputTask(LocalExecutorPlugin.java:256)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor.access$100(LocalExecutorPlugin.java:194)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor$1.call(LocalExecutorPlugin.java:233)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor$1.call(LocalExecutorPlugin.java:230)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
Caused by: java.io.FileNotFoundException: ./logs/sample_output_000.00.csv (No such file or directory)
    at java.io.FileOutputStream.open0(Native Method)
    at java.io.FileOutputStream.open(FileOutputStream.java:270)
    at java.io.FileOutputStream.<init>(FileOutputStream.java:213)
    at java.io.FileOutputStream.<init>(FileOutputStream.java:162)
    at org.embulk.standards.LocalFileOutputPlugin$1.nextFile(LocalFileOutputPlugin.java:86)
    ... 13 more

Error: java.lang.RuntimeException: java.io.FileNotFoundException: ./logs/sample_output_000.00.csv (No such file or directory)
```
(2) run

```
$ embulk run config.yml
2020-06-10 01:23:54.157 +0000: Embulk v0.9.23
2020-06-10 01:23:54.989 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-10 01:23:57.063 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-10 01:23:57.759 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-10 01:23:57.893 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-10 01:23:57.933 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-10 01:23:57.940 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 01:23:57.941 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 01:23:58.217 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-10 01:23:58.217 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-10 01:23:58.217 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-10 01:23:58.217 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-10 01:23:58.285 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-10 01:23:58.307 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-10 01:23:58.344 +0000 [INFO] (0015:task-0000): Writing local file './logs/sample_output_000.00.csv'
2020-06-10 01:23:58.347 +0000 [INFO] (0015:task-0000): Writing local file './logs/sample_output_001.00.csv'
2020-06-10 01:23:58.349 +0000 [INFO] (0015:task-0000): Writing local file './logs/sample_output_002.00.csv'
2020-06-10 01:23:58.350 +0000 [INFO] (0015:task-0000): Writing local file './logs/sample_output_003.00.csv'
2020-06-10 01:23:58.364 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 01:23:58.364 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 01:23:58.378 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth FROM `pet` WHERE c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'
2020-06-10 01:23:58.381 +0000 [INFO] (0015:task-0000): > 0.00 seconds
2020-06-10 01:23:58.385 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-10 01:23:58.389 +0000 [INFO] (main): Committed.
2020-06-10 01:23:58.389 +0000 [INFO] (main): Next config diff: {"in":{},"out":{}}
```

(3) 결과 확인

```
$ ls -1 logs/
sample_output_000.00.csv
sample_output_001.00.csv
sample_output_002.00.csv
```

- csv 포맷으로 파일에 저장할 경우, MySQL 테이블의 컬럼 이름이 맨 첫줄에 나타난다.
```
$ cat logs/sample_output_00*.csv
name,owner,species,sex,birth
Fluffy,Harold,cat,f,1993-02-04 00:00:00.000000 +0000
Claws,Gwen,cat,m,1904-03-17 00:00:00.000000 +0000
name,owner,species,sex,birth
name,owner,species,sex,birth
name,owner,species,sex,birth
```

### Out: RDS -> file (json format)
(1) File formatter plugin 설치
json 포맷으로 file에 저장하기 위해서는 json file formatter가 필요하다.
[File formatter plugin](https://plugins.embulk.org/#file-formatter)에서 jsonl 을 찾아서 설치한다.

```
$ embulk gem install embulk-formatter-jsonl
$ embulk gem list
```

(2) config.yml 작성 (myconfig/config.out-json-file.yml)
```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  where: "c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'"
out:
  type: file
  path_prefix: ./logs-json/sample_output_
  file_ext: json
  formatter:
    type: jsonl
    timezone: "UTC"
    date_format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
```

(3) run
```
$ embulk run config.yml
2020-06-10 01:38:33.524 +0000: Embulk v0.9.23
2020-06-10 01:38:34.324 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-10 01:38:36.320 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-10 01:38:37.019 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-10 01:38:37.139 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-10 01:38:37.175 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-10 01:38:37.180 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 01:38:37.182 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 01:38:37.398 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-10 01:38:37.399 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-10 01:38:37.399 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-10 01:38:37.399 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-10 01:38:37.462 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-10 01:38:37.504 +0000 [INFO] (0001:transaction): Loaded plugin embulk-formatter-jsonl (0.1.4)
2020-06-10 01:38:37.534 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-10 01:38:37.625 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 01:38:37.625 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 01:38:37.640 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth FROM `pet` WHERE c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'
2020-06-10 01:38:37.643 +0000 [INFO] (0015:task-0000): > 0.00 seconds
2020-06-10 01:38:37.657 +0000 [INFO] (embulk-output-executor-0): Writing local file './logs-json/sample_output_000.00.json'
2020-06-10 01:38:37.674 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-10 01:38:37.678 +0000 [INFO] (main): Committed.
2020-06-10 01:38:37.678 +0000 [INFO] (main): Next config diff: {"in":{},"out":{}}
```

(3) 결과 확인

```
$ ls logs-json/
sample_output_000.00.json

$ cat logs-json/sample_output_000.00.json
{"name":"Fluffy","owner":"Harold","species":"cat","sex":"f","birth":"1993-02-04T00:00:00.000+0000"}
{"name":"Claws","owner":"Gwen","species":"cat","sex":"m","birth":"1904-03-17T00:00:00.000+0000"}
```
- json 포맷으로 저장할 경우, embulk가 여러 개의 threads를 이용해서 parallel하게 mysql에서 데이터를 덤프할 때, 덤프한 데이터가 없는 경우, 출력하지 않는다.

### Out: RDS -> S3 (csv format)
(1) Output plugin 설치
S3에 저장하기 위해서 [Output plugin](https://plugins.embulk.org/#output)에서 S3 output plugin 을 찾아서 설치한다.

```
$ embulk gem install embulk-output-s3
$ embulk gem list
```

(2) config.yml 작성 (myconfig/config.out-s3.yml)

```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  where: "c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'"
out:
  type: s3
  path_prefix: logs/out
  file_ext: .csv
  bucket: embulk-demo-output-use1
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  tmp_path: /tmp
  formatter:
    type: csv
```

(3) run

```
$ embulk run config.yml
2020-06-07 15:03:45.710 +0000: Embulk v0.9.23
2020-06-07 15:03:46.537 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-07 15:03:48.559 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-07 15:03:49.239 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-07 15:03:49.370 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-07 15:03:49.405 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-07 15:03:49.438 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-07 15:03:49.444 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-07 15:03:49.445 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-07 15:03:49.670 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-07 15:03:49.670 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-07 15:03:49.670 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-07 15:03:49.670 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-07 15:03:49.729 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-07 15:03:49.756 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-07 15:03:50.046 +0000 [INFO] (0015:task-0000): Writing S3 file 'logs/out.000.00.csv'
2020-06-07 15:03:50.051 +0000 [INFO] (0015:task-0000): Writing S3 file 'logs/out.001.00.csv'
2020-06-07 15:03:50.053 +0000 [INFO] (0015:task-0000): Writing S3 file 'logs/out.002.00.csv'
2020-06-07 15:03:50.055 +0000 [INFO] (0015:task-0000): Writing S3 file 'logs/out.003.00.csv'
2020-06-07 15:03:50.069 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-07 15:03:50.070 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-07 15:03:50.083 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth FROM `pet` WHERE c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'
2020-06-07 15:03:50.086 +0000 [INFO] (0015:task-0000): > 0.00 seconds
2020-06-07 15:03:50.557 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-07 15:03:50.562 +0000 [INFO] (main): Committed.
2020-06-07 15:03:50.562 +0000 [INFO] (main): Next config diff: {"in":{},"out":{}}
```

(3) 결과 확인
```
$ aws s3 ls s3://embulk-demo-output-use1/logs/
2020-06-07 15:18:42        135 out.000.00.csv
2020-06-07 15:18:42         30 out.001.00.csv
2020-06-07 15:18:42         30 out.002.00.csv
2020-06-07 15:18:42         30 out.003.00.csv
```

### Out: RDS -> S3 (json format)
(1) Output / File Formatter plugin 설치
- S3에 저장하기 위해서 [Output plugin](https://plugins.embulk.org/#output)에서 S3 output plugin 을 찾아서 설치한다.
- json 포맷으로 file에 저장하기 위해서 [File formatter plugin](https://plugins.embulk.org/#file-formatter)에서 jsonl 을 찾아서 설치한다.

```
$ embulk gem install embulk-output-s3
$ embulk gem install embulk-formatter-jsonl
$ embulk gem list
```

(2) config.yml 작성 (myconfig/config.out-s3-json.yml)
```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  where: "c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'"
out:
  type: s3
  path_prefix: logs-json/out
  file_ext: .json
  bucket: embulk-demo-output-use1
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  formatter:
    type: jsonl
    timezone: "UTC"
    date_format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ”
```

(3) run

```
$ embulk run config.yml
2020-06-10 01:57:19.801 +0000: Embulk v0.9.23
2020-06-10 01:57:20.615 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-10 01:57:22.652 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-10 01:57:23.329 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-10 01:57:23.452 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-10 01:57:23.487 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-10 01:57:23.521 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-10 01:57:23.526 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 01:57:23.527 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 01:57:23.747 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-10 01:57:23.747 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-10 01:57:23.747 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-10 01:57:23.747 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-10 01:57:23.810 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-10 01:57:23.856 +0000 [INFO] (0001:transaction): Loaded plugin embulk-formatter-jsonl (0.1.4)
2020-06-10 01:57:23.894 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-10 01:57:24.246 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 01:57:24.246 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 01:57:24.270 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth FROM `pet` WHERE c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'
2020-06-10 01:57:24.276 +0000 [INFO] (0015:task-0000): > 0.00 seconds
2020-06-10 01:57:24.297 +0000 [INFO] (embulk-output-executor-0): Writing S3 file 'logs-json/out.000.00.json'
2020-06-10 01:57:24.668 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-10 01:57:24.672 +0000 [INFO] (main): Committed.
2020-06-10 01:57:24.672 +0000 [INFO] (main): Next config diff: {"in":{},"out":{}}
```

(4) 결과 확인
```
$ aws s3 ls s3://embulk-demo-output-use1/logs-json/
2020-06-08 13:04:14       1184 out.000.00.json
```

### Out: RDS -> S3 (json format을 gzip으로 압축해서 저장)
(1) Output / File Formatter plugin 설치
- S3에 저장하기 위해서 [Output plugin](https://plugins.embulk.org/#output)에서 S3 output plugin 을 찾아서 설치한다.
- json 포맷으로 file에 저장하기 위해서 [File formatter plugin](https://plugins.embulk.org/#file-formatter)에서 jsonl 을 찾아서 설치한다.

```
$ embulk gem install embulk-output-s3
$ embulk gem install embulk-formatter-jsonl
$ embulk gem list
```

(2) config.yml 작성 (myconfig/config.out-s3-json-gzip.yml)
```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  where: "c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'"
out:
  type: s3
  path_prefix: logs-json-gzip/out
  file_ext: .json.gz
  bucket: embulk-demo-output-use1
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  formatter:
    type: jsonl
    timezone: "UTC"
    date_format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
  encoders:
  - type: gzip
    level: 1
```
- 올바른 gzip 파일로 저장하기 위해서 **out:file_ext** 에 `.gz` 확장자를 추가해야 한다
- 출력 파일을 압축하기 위해서 **out** 섹션에 **encoders** 속성을 추가해야 한다.
  ```
  encoders:
  - type: gzip
    level: 1
  ```

(3) run
```
$ embulk run config.yml
2020-06-10 02:05:51.600 +0000: Embulk v0.9.23
2020-06-10 02:05:52.389 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-10 02:05:54.411 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-10 02:05:55.111 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-10 02:05:55.256 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-10 02:05:55.287 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-10 02:05:55.320 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-10 02:05:55.327 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 02:05:55.328 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 02:05:55.533 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-10 02:05:55.533 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-10 02:05:55.533 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-10 02:05:55.533 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-10 02:05:55.586 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-10 02:05:55.629 +0000 [INFO] (0001:transaction): Loaded plugin embulk-formatter-jsonl (0.1.4)
2020-06-10 02:05:55.676 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-10 02:05:56.061 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-10 02:05:56.061 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-10 02:05:56.083 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth FROM `pet` WHERE c_time >= '2020-06-06 13:23:00' AND c_time < '2020-06-06 13:24:00'
2020-06-10 02:05:56.086 +0000 [INFO] (0015:task-0000): > 0.00 seconds
2020-06-10 02:05:56.108 +0000 [INFO] (embulk-output-executor-0): Writing S3 file 'logs-json-gzip/out.000.00.json.gz'
2020-06-10 02:05:56.392 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-10 02:05:56.397 +0000 [INFO] (main): Committed.
2020-06-10 02:05:56.397 +0000 [INFO] (main): Next config diff: {"in":{},"out":{}}
```

(4) 결과 확인
```
$ aws s3 ls s3://embulk-demo-output-use1/logs-json-gzip/
2020-06-10 02:05:57        136 out.000.00.json.gz
```

### Out: Incrementally RDS -> S3 (json format을 gzip으로 압축해서 저장)
RDS의 CDC 데이터를 증분(incrementally)으로 덤프해서 S3에 json 포맷의 파일을 gzip으로 압축해서 저장하기

(1) Output / File Formatter plugin 설치
- S3에 저장하기 위해서 [Output plugin](https://plugins.embulk.org/#output)에서 S3 output plugin 을 찾아서 설치한다.
- json 포맷으로 file에 저장하기 위해서 [File formatter plugin](https://plugins.embulk.org/#file-formatter)에서 jsonl 을 찾아서 설치한다.

```
$ embulk gem install embulk-output-s3
$ embulk gem install embulk-formatter-jsonl
$ embulk gem list
```

(2) config.yml 작성 (myconfig/config.out-s3-incr-json-gzip.yml)

```
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth, c_time, pkid"
  incremental: true
  incremental_columns: [c_time, pkid]
  last_record: ['2020-06-06T00:00:00.000', 1]
out:
  type: s3
  path_prefix: logs-json/out
  file_ext: .json
  bucket: embulk-demo-output-use1
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  formatter:
    type: jsonl
    timezone: "UTC"
    date_format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ"
  encoders:
  - type: gzip
    level: 1
```
- `incremental: true` : 증분 덤프을 하도록 설정한다.
- `incremental_columns: [c_time, pkid]` : 증분 덤프를 위한 컬럼 이름들을 array 형식으로 추가한다.
  - **select** 속성에, 반드시 `incremental_columns`에 명시된 컬럼 이름들이 있어야 한다.
- `last_record: ['2020-06-06T00:00:00.000', 1]` : 최초 덤프 조건
  - `incremental_columns`에 `datetime` 또는 `timestamp` 데이터 타입의 컬럼이 있는 경우, `last_record`에 timestamp format을 `%Y-%m-%dT%H:%M:%S.%N` (java 기준) 으로 표시해야 한다.

(3) run
- (a) emubulk 실행 시, `-c <config-diff.yml>` 를 이용해서 `last_record` 변경 사항을 저장한 후, 다음 embulk 실행 시 사용한다.
```
$ embulk run config.yml -c config-diff.yml
2020-06-08 13:04:08.313 +0000: Embulk v0.9.23
2020-06-08 13:04:09.136 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-08 13:04:11.156 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-08 13:04:11.899 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-08 13:04:12.027 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-08 13:04:12.059 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-08 13:04:12.094 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-08 13:04:12.104 +0000 [INFO] (0001:transaction): Fetch size is 2. Using server-side prepared statement.
2020-06-08 13:04:12.105 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 13:04:12.303 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-08 13:04:12.304 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-08 13:04:12.304 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-08 13:04:12.304 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-08 13:04:12.365 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-08 13:04:12.407 +0000 [INFO] (0001:transaction): Loaded plugin embulk-formatter-jsonl (0.1.4)
2020-06-08 13:04:12.447 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-08 13:04:12.815 +0000 [INFO] (0015:task-0000): Fetch size is 2. Using server-side prepared statement.
2020-06-08 13:04:12.816 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 13:04:12.869 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth, c_time, pkid FROM `pet` WHERE ((`c_time` > ?) OR (`c_time` = ? AND `pkid` > ?)) ORDER BY `c_time`, `pkid`
2020-06-08 13:04:12.870 +0000 [INFO] (0015:task-0000): Parameters: ["2020-06-06T00:00:00.000", "2020-06-06T00:00:00.000", 1]
2020-06-08 13:04:12.890 +0000 [INFO] (0015:task-0000): > 0.00 seconds
2020-06-08 13:04:12.918 +0000 [INFO] (embulk-output-executor-0): Writing S3 file 'logs-json/out.000.00.json'
2020-06-08 13:04:13.208 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-08 13:04:13.214 +0000 [INFO] (main): Committed.
2020-06-08 13:04:13.214 +0000 [INFO] (main): Next config diff: {"in":{"last_record":["2020-06-06T13:26:42.000000",8]},"out":{}}
ubuntu@ip-172-31-8-230:~/workspace/embulk-demo$ cat config-diff.yml
in:
  last_record: ['2020-06-06T13:26:42.000000', 8]
out: {}
```
- (b) `last_record` 이후 변경된 CDC 데이터를 embulk를 이용해서 s3에 저장한다.
```
ubuntu@ip-172-31-8-230:~/workspace/embulk-demo$ embulk run config.yml -c config-diff.yml
2020-06-08 13:05:28.600 +0000: Embulk v0.9.23
2020-06-08 13:05:29.415 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-08 13:05:31.481 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-08 13:05:32.189 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-08 13:05:32.325 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-08 13:05:32.359 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-08 13:05:32.394 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-08 13:05:32.400 +0000 [INFO] (0001:transaction): Fetch size is 2. Using server-side prepared statement.
2020-06-08 13:05:32.401 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 13:05:32.621 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-08 13:05:32.621 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-08 13:05:32.621 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-08 13:05:32.621 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-08 13:05:32.683 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-08 13:05:32.729 +0000 [INFO] (0001:transaction): Loaded plugin embulk-formatter-jsonl (0.1.4)
2020-06-08 13:05:32.764 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-08 13:05:33.151 +0000 [INFO] (0015:task-0000): Fetch size is 2. Using server-side prepared statement.
2020-06-08 13:05:33.151 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 13:05:33.172 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth, c_time, pkid FROM `pet` WHERE ((`c_time` > ?) OR (`c_time` = ? AND `pkid` > ?)) ORDER BY `c_time`, `pkid`
2020-06-08 13:05:33.173 +0000 [INFO] (0015:task-0000): Parameters: ["2020-06-06T13:26:42.000000", "2020-06-06T13:26:42.000000", 8]
2020-06-08 13:05:33.226 +0000 [INFO] (0015:task-0000): > 0.02 seconds
2020-06-08 13:05:33.228 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
2020-06-08 13:05:33.232 +0000 [INFO] (main): Committed.
2020-06-08 13:05:33.232 +0000 [INFO] (main): Next config diff: {"in":{"last_record":["2020-06-06T13:26:42.000000",8]},"out":{}}
```

(4) Scheduling loading by cron
- 매일 증분 덤프를 할 경우, 다음과 같이 cronjob에 등록한다.
```
0 * * * * embulk run /path/to/config.yml -c /path/to/diff.yml
```

(5) Troubling shooting
(a) `incremental_columns`에 있는 컬럼(예를 들어, c_time)이 **select** 속성에 없는 경우, `Error: org.embulk.config.ConfigException: Column name 'c_time' is in incremental_columns option does not exist`  에러가 발생함

```
$ cat config.yml
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth"
  incremental: true
  incremental_columns: [c_time, pkid]
  last_record: ['2020-06-06 00:00:00', 1]
out:
  type: s3
  path_prefix: logs-json/out
  file_ext: .json
  bucket: embulk-demo-output-use1
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  tmp_path: /tmp
  formatter:
    type: jsonl
    timezone: "UTC"
    date_format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ”
  encoders:
  - type: gzip
    level: 1

$ embulk run config.yml
2020-06-08 12:49:52.342 +0000: Embulk v0.9.23
2020-06-08 12:49:53.164 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-08 12:49:55.165 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-08 12:49:55.889 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-08 12:49:56.021 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-08 12:49:56.055 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-08 12:49:56.089 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-08 12:49:56.094 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-08 12:49:56.095 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 12:49:56.298 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-08 12:49:56.298 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
  1 in:
2020-06-08 12:49:56.298 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-08 12:49:56.298 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
org.embulk.exec.PartialExecutionException: org.embulk.config.ConfigException: Column name 'c_time' is in incremental_columns option does not exist
    at org.embulk.exec.BulkLoader$LoaderState.buildPartialExecuteException(BulkLoader.java:340)
    at org.embulk.exec.BulkLoader.doRun(BulkLoader.java:566)
    at org.embulk.exec.BulkLoader.access$000(BulkLoader.java:35)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:353)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:350)
    at org.embulk.spi.Exec.doWith(Exec.java:22)
    at org.embulk.exec.BulkLoader.run(BulkLoader.java:350)
    at org.embulk.EmbulkEmbed.run(EmbulkEmbed.java:242)
    at org.embulk.EmbulkRunner.runInternal(EmbulkRunner.java:291)
    at org.embulk.EmbulkRunner.run(EmbulkRunner.java:155)
    at org.embulk.cli.EmbulkRun.runSubcommand(EmbulkRun.java:431)
    at org.embulk.cli.EmbulkRun.run(EmbulkRun.java:90)
    at org.embulk.cli.Main.main(Main.java:64)
    Suppressed: java.lang.NullPointerException
        at org.embulk.exec.BulkLoader.doCleanup(BulkLoader.java:463)
        at org.embulk.exec.BulkLoader$3.run(BulkLoader.java:397)
        at org.embulk.exec.BulkLoader$3.run(BulkLoader.java:394)
        at org.embulk.spi.Exec.doWith(Exec.java:22)
        at org.embulk.exec.BulkLoader.cleanup(BulkLoader.java:394)
        at org.embulk.EmbulkEmbed.run(EmbulkEmbed.java:245)
        ... 5 more
Caused by: org.embulk.config.ConfigException: Column name 'c_time' is in incremental_columns option does not exist
    at org.embulk.input.jdbc.AbstractJdbcInputPlugin.findIncrementalColumnIndexes(AbstractJdbcInputPlugin.java:360)
    at org.embulk.input.jdbc.AbstractJdbcInputPlugin.setupTask(AbstractJdbcInputPlugin.java:280)
    at org.embulk.input.jdbc.AbstractJdbcInputPlugin.transaction(AbstractJdbcInputPlugin.java:214)
    at org.embulk.exec.BulkLoader.doRun(BulkLoader.java:507)
    ... 11 more

Error: org.embulk.config.ConfigException: Column name 'c_time' is in incremental_columns option does not exist
```
(b) `last_record`에 명시한 `datetime` 또는 `timestamp` 컬럼의 포맷이 잘못된 경우, `Error: org.embulk.spi.time.TimestampParseException: Cannot parse '2020-06-06 00:00:00' by '%Y-%m-%dT%H:%M:%S.%N’`  에러가 발생함
```
$ cat config.yml
in:
  type: mysql
  host: <db_host>
  user: "<db user>"
  password: "<db password>"
  database: test
  table: pet
  select: "name, owner, species, sex, birth, c_time, pkid"
  incremental: true
  incremental_columns: [c_time, pkid]
  last_record: ['2020-06-06 00:00:00', 1]
out:
  type: s3
  path_prefix: logs-json/out
  file_ext: .json
  bucket: embulk-demo-output-use1
  access_key_id: AKIAIOSFODNN7EXAMPLE
  secret_access_key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
  tmp_path: /tmp
  formatter:
    type: jsonl
    timezone: "UTC"
    date_format: "yyyy-MM-dd'T'HH:mm:ss.SSSZ”
  encoders:
  - type: gzip
    level: 1

$  embulk run config.yml
2020-06-08 12:51:02.883 +0000: Embulk v0.9.23
2020-06-08 12:51:03.704 +0000 [WARN] (main): DEPRECATION: JRuby org.jruby.embed.ScriptingContainer is directly injected.
2020-06-08 12:51:05.714 +0000 [INFO] (main): Gem's home and path are set by default: "/home/ubuntu/.embulk/lib/gems"
2020-06-08 12:51:06.404 +0000 [INFO] (main): Started Embulk v0.9.23
2020-06-08 12:51:06.530 +0000 [INFO] (0001:transaction): Loaded plugin embulk-input-mysql (0.10.1)
2020-06-08 12:51:06.565 +0000 [INFO] (0001:transaction): Loaded plugin embulk-output-s3 (1.5.0)
2020-06-08 12:51:06.600 +0000 [INFO] (0001:transaction): JDBC Driver = /home/ubuntu/.embulk/lib/gems/gems/embulk-input-mysql-0.10.1/default_jdbc_driver/mysql-connector-java-5.1.44.jar
2020-06-08 12:51:06.605 +0000 [INFO] (0001:transaction): Fetch size is 10000. Using server-side prepared statement.
2020-06-08 12:51:06.606 +0000 [INFO] (0001:transaction): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 12:51:06.802 +0000 [INFO] (0001:transaction): Using JDBC Driver mysql-connector-java-5.1.44 ( Revision: b3cda4f864902ffdde495b9df93937c3e20009be )
2020-06-08 12:51:06.802 +0000 [WARN] (0001:transaction): embulk-input-mysql 0.9.0 upgraded the bundled MySQL Connector/J version from 5.1.34 to 5.1.44 .
2020-06-08 12:51:06.802 +0000 [WARN] (0001:transaction): And set useLegacyDatetimeCode=false by default in order to get correct datetime value when the server timezone and the client timezone are different.
2020-06-08 12:51:06.802 +0000 [WARN] (0001:transaction): Set useLegacyDatetimeCode=true if you need to get datetime value same as older embulk-input-mysql.
2020-06-08 12:51:06.860 +0000 [INFO] (0001:transaction): Using local thread executor with max_threads=8 / output tasks 4 = input tasks 1 * 4
2020-06-08 12:51:06.913 +0000 [INFO] (0001:transaction): Loaded plugin embulk-formatter-jsonl (0.1.4)
2020-06-08 12:51:06.951 +0000 [INFO] (0001:transaction): {done:  0 / 1, running: 0}
2020-06-08 12:51:07.347 +0000 [INFO] (0015:task-0000): Fetch size is 10000. Using server-side prepared statement.
2020-06-08 12:51:07.347 +0000 [INFO] (0015:task-0000): Connecting to jdbc:mysql://cdc-test.cluster-ro-alqvd4onbu3w.us-east-1.rds.amazonaws.com:3306/test options {useCompression=true, socketTimeout=1800000, useSSL=false, user=admin, useLegacyDatetimeCode=false, tcpKeepAlive=true, useCursorFetch=true, connectTimeout=300000, password=***, zeroDateTimeBehavior=convertToNull}
2020-06-08 12:51:07.360 +0000 [INFO] (0015:task-0000): SQL: SELECT name, owner, species, sex, birth, c_time, pkid FROM `pet` WHERE ((`c_time` > ?) OR (`c_time` = ? AND `pkid` > ?)) ORDER BY `c_time`, `pkid`
2020-06-08 12:51:07.363 +0000 [INFO] (0015:task-0000): Parameters: ["2020-06-06 00:00:00", "2020-06-06 00:00:00", 1]
2020-06-08 12:51:07.380 +0000 [INFO] (0001:transaction): {done:  1 / 1, running: 0}
org.embulk.exec.PartialExecutionException: org.embulk.spi.time.TimestampParseException: Cannot parse '2020-06-06 00:00:00' by '%Y-%m-%dT%H:%M:%S.%N'
    at org.embulk.exec.BulkLoader$LoaderState.buildPartialExecuteException(BulkLoader.java:340)
    at org.embulk.exec.BulkLoader.doRun(BulkLoader.java:566)
    at org.embulk.exec.BulkLoader.access$000(BulkLoader.java:35)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:353)
    at org.embulk.exec.BulkLoader$1.run(BulkLoader.java:350)
    at org.embulk.spi.Exec.doWith(Exec.java:22)
    at org.embulk.exec.BulkLoader.run(BulkLoader.java:350)
    at org.embulk.EmbulkEmbed.run(EmbulkEmbed.java:242)
    at org.embulk.EmbulkRunner.runInternal(EmbulkRunner.java:291)
    at org.embulk.EmbulkRunner.run(EmbulkRunner.java:155)
    at org.embulk.cli.EmbulkRun.runSubcommand(EmbulkRun.java:431)
    at org.embulk.cli.EmbulkRun.run(EmbulkRun.java:90)
    at org.embulk.cli.Main.main(Main.java:64)
Caused by: org.embulk.spi.time.TimestampParseException: Cannot parse '2020-06-06 00:00:00' by '%Y-%m-%dT%H:%M:%S.%N'
    at org.embulk.spi.time.TimestampParserLegacy.parseInternal(org/embulk/spi/time/TimestampParserLegacy.java:79)
    at org.embulk.spi.time.TimestampParser.parseInternal(org/embulk/spi/time/TimestampParser.java:224)
    at org.embulk.spi.time.TimestampParser.parse(org/embulk/spi/time/TimestampParser.java:229)
    at org.embulk.input.mysql.getter.AbstractMySQLTimestampIncrementalHandler.decodeFromJsonTo(org/embulk/input/mysql/getter/AbstractMySQLTimestampIncrementalHandler.java:83)
    at org.embulk.input.jdbc.JdbcInputConnection.prepareParameters(org/embulk/input/jdbc/JdbcInputConnection.java:159)
    at org.embulk.input.mysql.MySQLInputConnection.newBatchSelect(org/embulk/input/mysql/MySQLInputConnection.java:36)
    at org.embulk.input.jdbc.JdbcInputConnection.newSelectCursor(org/embulk/input/jdbc/JdbcInputConnection.java:130)
    at org.embulk.input.jdbc.AbstractJdbcInputPlugin.run(org/embulk/input/jdbc/AbstractJdbcInputPlugin.java:495)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor.runInputTask(org/embulk/exec/LocalExecutorPlugin.java:269)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor.access$100(org/embulk/exec/LocalExecutorPlugin.java:194)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor$1.call(org/embulk/exec/LocalExecutorPlugin.java:233)
    at org.embulk.exec.LocalExecutorPlugin$ScatterExecutor$1.call(org/embulk/exec/LocalExecutorPlugin.java:230)
    at java.util.concurrent.FutureTask.run(java/util/concurrent/FutureTask.java:266)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(java/util/concurrent/ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(java/util/concurrent/ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(java/lang/Thread.java:748)

Error: org.embulk.spi.time.TimestampParseException: Cannot parse '2020-06-06 00:00:00' by '%Y-%m-%dT%H:%M:%S.%N’
```

# References
* [embulk](https://www.embulk.org/docs/) - a open-source bulk data loader that helps data transfer between various databases, storages, file formats, and cloud services.
* [mycli](https://www.mycli.net/) - a command line interface for MySQL, MariaDB, and Percona with auto-completion and syntax highlighting
* [Loading CDC from MySQL to Amazon Kinesis Data Streams](https://github.com/ksmin23/lambda-cdc-to-kinesis)
