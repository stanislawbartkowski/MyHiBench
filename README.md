# MyHiBench

https://github.com/intel-hadoop/HiBench

HiBench, like any other benchmark, can be used to measure the performance of the Hadoop setup and also as a tool to evaluate the general health check of the installation because it touches and streches many aspects of Hadoop stack.<br>
Although the configuration of HiBench is simple and straighforward, at the beginning is not obvious and requires several lessons to be learnt.<br>
Below I'm describing a practical steps how to install and configure HiBench to push it into motion without much ado. The Hadoop installation under test is RedHat  HortonWorks 2.6.5 (https://docs.hortonworks.com/HDPDocuments/HDP2/HDP-2.6.5/bk_release-notes/content/ch_relnotes.html) but it applies to all HDP versions.

# Installation
## Prerequisities
* Download *maven* if not installed yet. (https://maven.apache.org/download.cgi).<br>
* Unpack in *opt* directory and make in */usr/local/bin* a link to *mvn* executable. 
  * mvn -> /opt/apache-maven-3.6.0/bin/mvn
* Verify
>mvn
```
Apache Maven 3.6.0 (97c98ec64a1fdfee7767ce5ffb20918da4f719f3; 2018-10-24T20:41:47+02:00)
Maven home: /opt/apache-maven-3.6.0
Java version: 1.8.0_201, vendor: Oracle Corporation, runtime: /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.201.b09-2.el7_6.x86_64/jre
Default locale: pl_PL, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-957.1.3.el7.x86_64", arch: "amd64", family: "unix"
```
## Download HiBench and build it
https://github.com/intel-hadoop/HiBench/blob/master/docs/build-hibench.md
<br>
> git clone https://github.com/intel-hadoop/HiBench.git<br>
> cd HiBench<br>
> mvn -Dspark=2.2 clean package<br>
## Enable for Spark 2.3
### vi sparkbench/pom.xml
After spark2.2 profile add spark2.3
```XML
  <profile>
      <id>spark2.3</id>
      <properties>
        <spark.version>2.3.0</spark.version>
        <spark.bin.version>2.3</spark.bin.version>
      </properties>
      <activation>
        <property>
          <name>spark</name>
          <value>2.3</value>
        </property>
      </activation>
    </profile>
```
### vi sparkbench/streaming/pom.xml
```XML
  <profile>
      <id>spark2.3</id>
      <dependencies>
        <dependency>
          <groupId>org.apache.spark</groupId>
          <artifactId>spark-streaming-kafka-0-8_2.11</artifactId>
          <version>2.3.0</version>
        </dependency>
      </dependencies>
      <activation>
        <property>
          <name>spark</name>
          <value>2.3</value>
        </property>
      </activation>
    </profile>
```
### vi sparkbench/structuredStreaming/pom.xml
```XML
 <profile>
      <id>spark2.3</id>
      <dependencies>
        <dependency>
          <groupId>org.apache.spark</groupId>
          <artifactId>spark-streaming-kafka-0-8_${scala.binary.version}</artifactId>
          <version>2.3.0</version>
        </dependency>
        <dependency>
          <groupId>org.apache.spark</groupId>
          <artifactId>spark-sql-kafka-0-10_${scala.binary.version}</artifactId>
          <version>2.3.0</version>
        </dependency>
      </dependencies>
      <activation>
        <property>
          <name>spark</name>
          <value>2.3</value>
        </property>
      </activation>
    </profile>
```
### Build
> mvn -Dspark=2.3 clean package

## Hadoop user
For user running the benchmark:
* create */user/{user}* directory
* if Ranger is enabled, give the user privileges : *submitjob* and *admin-queue*
## Configure
### conf/hibench.conf
https://github.com/intel-hadoop/HiBench/blob/master/docs/run-hadoopbench.md

As a minimum, modify the following parameters

| Parameter | Example value |
| -- | -- |
| hibench.masters.hostnames | hurds1.fyre.ibm.com,a1.fyre.ibm.com,aa1.fyre.ibm.com
| hibench.slaves.hostnames | hurds2.fyre.ibm.com,hurds3.fyre.ibm.com,hurds4.fyre.ibm.com,hurds5.fyre.ibm.com,hurds5.fyre.ibm.com
### conf/hadoop.conf
> cp conf/hadoop.conf.template conf/hadoop.conf<br>

| Parameter | Example value |
| --- | --- |
| hibench.hadoop.home | /usr/hdp/current/hadoop-client
| hibench.hdfs.master |  hdfs://a1.fyre.ibm.com:8020/tmp/hibench
| hibench.hadoop.release | hdp
### conf/spark.conf
https://github.com/intel-hadoop/HiBench/blob/master/docs/run-sparkbench.md

> cp conf/spark.conf.template conf/spark.conf
<br>

| Parameter | Example value |
| -- | -- |
| hibench.spark.home | /usr/hdp/current/spark2-client
| hibench.spark.master | yarn
### modify
In order to run the script in the background (by adding & sign), modify one of the scripts in two places.
> vi bin/functions/workload_functions.sh<br>

(add *eval* before $CMD)
```bash
function execute () {
    CMD="$@"
    echo -e "${BCyan}Executing: ${Cyan}${CMD}${Color_Off}"
    eval $CMD
}
..........
function execute_withlog () {
    CMD="$@"
    if [ -t 1 ] ; then          # Terminal, beautify the output.
        ${workload_func_bin}/execute_with_log.py ${WORKLOAD_RESULT_FOLDER}/bench.log $CMD
    else                        # pipe, do nothing.
        eval $CMD
    fi
}
```
### Test
Test Hadoop<br>
> bin/workloads/micro/wordcount/prepare/prepare.sh<br>
>bin/workloads/micro/wordcount/spark/run.sh<br>

Test Spark<br>
> bin/workloads/ml/als/prepare/prepare.sh<br>
> bin/workloads/ml/als/spark/run.sh<br>

### Test size
The test size is defined om *conf/hibnech.conf* file. 
```
# Data scale profile. Available value is tiny, small, large, huge, gigantic and bigdata.
# The definition of these profiles can be found in the workload's conf file i.e. conf/workloads/micro/wordcount.conf
hibench.scale.profile               large
```
### Running the *large* test
In my environment I experienced problem with *large* test and *websearch.nutchindexing* test. Firstly, the prepared data exceeded the 1T capacity of the installation. The size can be reduced by modifying the value
>  vi conf/workloads/websearch/nutchindexing.conf <br>
```
hibench.nutch.tiny.pages                        25000
hibench.nutch.small.pages                       1000000
hibench.nutch.large.pages                       100000000
hibench.nutch.huge.pages                        10000000000
hibench.nutch.gigantic.pages                    100000000000
hibench.nutch.bigdata.pages                     100000000000
```
Secondly, after reducing the size, the test failed because of memory issue in M/R execution.<br>
The memory problem also failed the *graph.nweight* workbench.
## Kafka streaming
### Configurtion
https://github.com/intel-hadoop/HiBench/blob/master/docs/run-streamingbench.md

> vi conf/hibench.conf<br>

| Parameter | Example value|
| --- | --- |
| hibench.streambench.kafka.home | /usr/hdp/current/kafka-broker |
| hibench.streambench.zkHost | a1.fyre.ibm.com:2181,aa1.fyre.ibm.com:2181,hurds1.fyre.ibm.com:2181
| hibench.streambench.kafka.brokerList | a1.fyre.ibm.com:6667

<br>
In *conf/spark.conf* replace *hibench.streambench.spark.checkpointPath* parameter with HDFS value accessible for the user running the test.<br>

> vi conf/spark.conf

| Parameter | Value |
| -- | -- |
| hibench.streambench.spark.checkpointPath | /tmp |

### vi bin/workloads/streaming/identity/prepare/dataGen.sh
Modify the script to get access to HDFS file system. Enhance *-cp* parameter with Hadoop client jars.
```
#JVM_OPTS="-Xmx1024M -server -XX:+UseCompressedOops -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -XX:+CMSScavengeBeforeRemark -XX:+DisableExplicitGC -Djava.awt.headless=true -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dkafka.logs.dir=bin/../logs -cp ${DATATOOLS}"

JVM_OPTS="-Xmx1024M -server -XX:+UseCompressedOops -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -XX:+CMSScavengeBeforeRemark -XX:+DisableExplicitGC -Djava.awt.headless=true -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dkafka.logs.dir=bin/../logs -cp $HADOOP_HOME/*:$HADOOP_HOME/client/*:${DATATOOLS}"
```
### Modify for Kerberos
>vi bin/workloads/streaming/identity/prepare/dataGen.sh<br>

(add */etc/hadoop/conf* to -cp)
```bash
.........
JVM_OPTS="-Xmx1024M -server -XX:+UseCompressedOops -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:+CMSClassUnloadingEnabled -XX:+CMSScavengeBeforeRemark -XX:+DisableExplicitGC -Djava.awt.headless=true -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dkafka.logs.dir=bin/../logs -cp ${DATATOOLS}:/etc/hadoop/conf"
..........

```
### Prepare data
> bin/workloads/streaming/identity/prepare/genSeedDataset.sh<br>
### Start producing stream of data
> bin/workloads/streaming/identity/prepare/dataGen.sh<br>
```
============ StreamBench Data Generator ============
 Interval Span       : 50 ms
 Record Per Interval : 5
 Record Length       : 200 bytes
 Producer Number     : 1
 Total Records        : -1 [Infinity]
 Total Rounds         : -1 [Infinity]
 Kafka Topic          : identity
====================================================
Estimated Speed :
    100 records/second
    0.02 Mb/second
====================================================
log4j:WARN No appenders could be found for logger (org.apache.kafka.clients.producer.ProducerConfig).
log4j:WARN Please initialize the log4j system properly.
log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
opening all parts in path: hdfs://mdp1.sb.com:8020/tmp/bench/HiBench/Streaming/Seed/uservisits, from offset: 0
pool-1-thread-1 - starting generate data ...
```
## Test using Kafka command line tools

**Kerberos**
If Kafka is protected by Kerberos then *keytab* file with Kerberos principal is necessary. <br>
Prepare *jaas* file, replace *keyTab* and *principal* parameter with valid values.
```
KafkaServer {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/home/bench/jaas/bench.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="kafka"
   principal="bench@FYRE.NET";
};
KafkaClient {
   com.sun.security.auth.module.Krb5LoginModule required
   useTicketCache=true
   renewTicket=true
   serviceName="kafka";
};
Client {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/home/bench/jaas/bench.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="zookeeper"
   principal="bench@FYRE.NET";
};
```
Before running command line toos, export bash variable pointing to place where *jaas* file is saved.
> export KAFKA_KERBEROS_PARAMS=-Djava.security.auth.login.config=/home/bench/jaas/kafka_jaas.conf
<br>

List topics<br>

> /usr/hdp/3.1.0.0-78/kafka/bin/kafka-topics.sh --zookeeper mdp1.sb.com:2181,mdp2.sb.com:2181,mdp3.sb.com:2181 --list<br>
```
SPARK_identity_1_5_50_1553699883942
SPARK_identity_1_5_50_1553700725899
SPARK_identity_1_5_50_1553700790191
SPARK_identity_1_5_50_1553719366371
SPARK_identity_1_5_50_1553719791840
SPARK_identity_1_5_50_1553720512069
SPARK_identity_1_5_50_1553720760253
__consumer_offsets
ambari_kafka_service_check
identity

```
>  /usr/hdp/3.1.0.0-78/kafka/bin/kafka-console-consumer.sh  --bootstrap-server  mdp1.sb.com:6667 --topic identity<br>

(stream of lines)<br>
```
39568	163.48.19.64,cnwtiocjooynqvcjwrplpnexcomgpybcrqriswbfcpnazkaqkwckuqitjfznhhgbbzzwphzmqfafqdkjxlafycm,1975-03-19,0.1348409,Mozilla/5.0 (Windows; U; Windows NT 5.2) AppleWebKit/525.13
39568	169.95.134.151,cnwtiocjooynqvcjwrplpnexcomgpybcrqriswbfcpnazkaqkwckuqitjfznhhgbbzzwphzmqfafqdkjxlafycm,2007-07-02,0.23636532,Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 6.0),ISL,I
39568	142.5.139.121,cnwtiocjooynqvcjwrplpnexcomgpybcrqriswbfcpnazkaqkwckuqitjfznhhgbbzzwphzmqfafqdkjxlafycm,1979-09-02,0.99028385,Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0),BEL,BE
39568	57.174.175.181,cnwtiocjooynqvcjwrplpnexcomgpybcrqriswbfcpnazkaqkwckuqitjfznhhgbbzzwphzmqfafqdkjxlafycm,1994-12-23,0.4787203,Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0),AUS,AU
39576	178.227.245.86,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,2003-05-25,0.44906622,Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1),PHL,PHL-EN,Sakha,6
39576	207.206.180.144,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,2011-07-07,0.989489,Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1),POL,POL-PL,sulphured,6
39576	119.181.133.206,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,1980-03-01,0.5762792,Pzheuxmiqiggwls/3.3,ARE,ARE-AR,inevitable,10
39576	90.190.160.33,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,1974-02-07,0.3832044,Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1),ISL,ISL-IS,firemen,2
39576	233.99.172.182,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,1989-10-19,0.7538247,Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 5.1),ROU,ROU-RO,paramedic,1
39576	217.33.120.134,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,1972-03-04,0.37720335,Wqzfuwmaqsleg/5.1,SVK,SVK-SK,knifes,8
39576	83.184.206.126,rbugnwgtufibpnzapyujlpkycickwftijhmfiaffhmxhvlgevubmxnmeoyrn,1997-03-19,0.76973563,Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.0),PRY,PRY-ES,Agustin,3


```
### In parallel, run the Spark streaming client
> bin/workloads/streaming/identity/spark/run.sh<br>
```
19/03/19 15:33:17 INFO Client:
         client token: N/A
         diagnostics: N/A
         ApplicationMaster host: 192.168.122.74
         ApplicationMaster RPC port: 0
         queue: default
         start time: 1553005993879
         final status: UNDEFINED
         tracking URL: http://mdp1.sb.com:8088/proxy/application_1552994510056_0051/
         user: bench
19/03/19 15:33:17 INFO YarnClientSchedulerBackend: Application application_1552994510056_0051 has started running.
19/03/19 15:33:17 INFO Utils: Successfully started service 'org.apache.spark.network.netty.NettyBlockTransferService' on port 43370.
19/03/19 15:33:17 INFO NettyBlockTransferService: Server created on mdp1.sb.com:43370
19/03/19 15:33:17 INFO BlockManager: Using org.apache.spark.storage.RandomBlockReplicationPolicy for block replication policy
19/03/19 15:33:17 INFO BlockManagerMaster: Registering BlockManager BlockManagerId(driver, mdp1.sb.com, 43370, None)
19/03/19 15:33:17 INFO BlockManagerMasterEndpoint: Registering block manager mdp1.sb.com:43370 with 673.5 MB RAM, BlockManagerId(driver, mdp1.sb.com, 43370, None)
19/03/19 15:33:17 INFO BlockManagerMaster: Registered BlockManager BlockManagerId(driver, mdp1.sb.com, 43370, None)
19/03/19 15:33:17 INFO BlockManager: Initialized BlockManager: BlockManagerId(driver, mdp1.sb.com, 43370, None)
19/03/19 15:33:17 INFO JettyUtils: Adding filter org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter to /metrics/json.
19/03/19 15:33:17 INFO ContextHandler: Started o.s.j.s.ServletContextHandler@1291aab5{/metrics/json,null,AVAILABLE,@Spark}
19/03/19 15:33:20 INFO YarnSchedulerBackend$YarnDriverEndpoint: Registered executor NettyRpcEndpointRef(spark-client://Executor) (192.168.122.82:36746) with ID 1
19/03/19 15:33:20 INFO BlockManagerMasterEndpoint: Registering block manager mdp4.sb.com:42709 with 673.5 MB RAM, BlockManagerId(1, mdp4.sb.com, 42709, None)
19/03/19 15:33:20 INFO YarnSchedulerBackend$YarnDriverEndpoint: Registered executor NettyRpcEndpointRef(spark-client://Executor) (192.168.122.105:41418) with ID 2
19/03/19 15:33:20 INFO YarnClientSchedulerBackend: SchedulerBackend is ready for scheduling beginning after reached minRegisteredResourcesRatio: 0.8
19/03/19 15:33:20 INFO BlockManagerMasterEndpoint: Registering block manager mdp2.sb.com:37615 with 673.5 MB RAM, BlockManagerId(2, mdp2.sb.com, 37615, None)
```
In tiny environment, the Spark client can fail.
```
19/03/27 22:02:06 INFO BlockManagerMasterEndpoint: Registering block manager mdp2.sb.com:33446 with 620.1 MB RAM, BlockManagerId(2, mdp2.sb.com, 33446, None)
java.util.concurrent.RejectedExecutionException: Task org.apache.spark.streaming.CheckpointWriter$CheckpointWriteHandler@2f33e0c8 rejected from java.util.concurrent.Threa...
        at java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2063)
        at java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:830)
        at java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1379)
        at org.apache.spark.streaming.CheckpointWriter.write(Checkpoint.scala:290)
        at org.apache.spark.streaming.scheduler.JobGenerator.doCheckpoint(JobGenerator.scala:297)
        at org.apache.spark.streaming.scheduler.JobGenerator.org$apache$spark$streaming$scheduler$JobGenerator$$processEvent(JobGenerator.scala:186)
        at org.apache.spark.streaming.scheduler.JobGenerator$$anon$1.onReceive(JobGenerator.scala:89)
        at org.apache.spark.streaming.scheduler.JobGenerator$$anon$1.onReceive(JobGenerator.scala:88)
        at org.apache.spark.util.EventLoop$$anon$1.run(EventLoop.scala:48)

```
The solution is to modify *hibench.streambench.spark.batchInterval* parameter in *conf/spark.conf*
```
#======================================================
# Spark Streaming
#======================================================
# Spark streaming Batchnterval in millisecond (default 100)
#hibench.streambench.spark.batchInterval          100
hibench.streambench.spark.batchInterval          200

```

### Get metrics from time to time
>  bin/workloads/streaming/identity/common/metrics_reader.sh
```
start MetricsReader bench
SPARK_identity_1_5_50_1552999658022
SPARK_identity_1_5_50_1553000124410
SPARK_identity_1_5_50_1553000236815
SPARK_identity_1_5_50_1553001022513
SPARK_identity_1_5_50_1553001112518
SPARK_identity_1_5_50_1553001343170
SPARK_identity_1_5_50_1553003441519
SPARK_identity_1_5_50_1553005025887
SPARK_identity_1_5_50_1553005572004
SPARK_identity_1_5_50_1553005923312
SPARK_identity_1_5_50_1553005989549
__consumer_offsets
ambari_kafka_service_check
identity
Please input the topic:
```
Enter the latest topic and get the result as csv file.

### HDP 3.1
Out of the box, HiBench will fail while running on HDP 3.1. Several manual changes are required.
## vi pom.xml
> <hadoop.mr2.version>2.4.0</hadoop.mr2.version><br>

with<br>
> <hadoop.mr2.version>3.1.0</hadoop.mr2.version><br>

## vi autogen/src/main/java/org/apache/hadoop/fs/dfsioe/TestDFSIO.java
Replace 
> import org.apache.commons.logging.*;<br>

with <br>

> import org.slf4j.*;<br>
```
import java.util.Date;
import java.util.StringTokenizer;

//import org.apache.commons.logging.*;
import org.slf4j.*;

import org.apache.hadoop.fs.*;
import org.apache.hadoop.mapred.*;
```

Replace
>  private static final Log LOG = FileInputFormat.LOG;<br>

with

>   private static final Logger LOG = FileInputFormat.LOG;<br>
```
  private static final String DEFAULT_RES_FILE_NAME = "TestDFSIO_results.log";
 
// private static final Log LOG = FileInputFormat.LOG;
  private static final Logger LOG = FileInputFormat.LOG;

  private static Configuration fsConfig = new Configuration();
  private static final long MEGA = 0x100000;
  private static String TEST_ROOT_DIR = System.getProperty("test.build.data","/benchmarks/TestDFSIO");
  private static Path CONTROL_DIR = new Path(TEST_ROOT_DIR, "io_control");
```
## vi autogen/src/main/java/org/apache/hadoop/fs/dfsioe/TestDFSIOEnh.java
At the end of **import** section, add
> import java.util.Arrays;<br>
```
import org.apache.hadoop.fs.*;
import org.apache.hadoop.fs.dfsioe.Analyzer._Mapper;
import org.apache.hadoop.fs.dfsioe.Analyzer._Reducer;

import java.util.Arrays;
```
In HDP 3.1, procedure *copyMerge* from package *FileUtil* is removed, so copy and paste the deprecated method body into the beginning of the Java code, before the first method and after: <br>

>   private static Configuration fsConfig = new Configuration(); <br>


```
	private static Path checkDest(String srcName, FileSystem dstFS, Path dst,
		      boolean overwrite) throws IOException {
		    if (dstFS.exists(dst)) {
		      FileStatus sdst = dstFS.getFileStatus(dst);
		      if (sdst.isDirectory()) {
		        if (null == srcName) {
		          throw new IOException("Target " + dst + " is a directory");
		        }
		        return checkDest(null, dstFS, new Path(dst, srcName), overwrite);
		      } else if (!overwrite) {
		        throw new IOException("Target " + dst + " already exists");
		      }
		    }
		    return dst;
		  }
	
	private static boolean copyMerge(FileSystem srcFS, Path srcDir, FileSystem dstFS, Path dstFile, boolean deleteSource,
			Configuration conf, String addString) throws IOException {
		dstFile = checkDest(srcDir.getName(), dstFS, dstFile, false);

		if (!srcFS.getFileStatus(srcDir).isDirectory())
			return false;

		OutputStream out = dstFS.create(dstFile);

		try {
			FileStatus contents[] = srcFS.listStatus(srcDir);
			Arrays.sort(contents);
			for (int i = 0; i < contents.length; i++) {
				if (contents[i].isFile()) {
					InputStream in = srcFS.open(contents[i].getPath());
					try {
						IOUtils.copyBytes(in, out, conf, false);
						if (addString != null)
							out.write(addString.getBytes("UTF-8"));

					} finally {
						in.close();
					}
				}
			}
		} finally {
			out.close();
		}

		if (deleteSource) {
			return srcFS.delete(srcDir, true);
		} else {
			return true;
		}
	}
```
Replace line:
> FileUtil.copyMerge(fs, DfsioeConfig.getInstance().getReportDir(fsConfig), fs, DfsioeConfig.getInstance().getReportTmp(fsConfig), false, fsConfig, null); <br>

with (remove *FileUtil* qualifier)
> copyMerge(fs, DfsioeConfig.getInstance().getReportDir(fsConfig), fs, DfsioeConfig.getInstance().getReportTmp(fsConfig), false, fsConfig, null);<br>

## Run
Configure, build and run *wordcount* test as described above. <br>
Then, before running the full benchmark, execute<br>

> bin/workloads/micro/dfsioe/prepare/prepare.sh<br>

## Hive 3.1
HiBench is running standalone Hive 0.17 which does not talk to Hive 3.1. The solution is to use cluster *hive* command line.
## Modify *Hive* configuarion
Ambari console->Hive->Advanced->Custom hive-site.xml<br>
Add property (pay attention to | as field delimiter)

| Property | Values |
| -- | -- |
| hive.security.authorization.sqlstd.confwhitelist.append | hive.input.format\|hive.stats.autogather\|mapreduce.job.reduces\|mapreduce.job.maps 
##  vi bin/workloads/sql/aggregation/hadoop/run.sh
Use *hive* command line directly.Replace:<br>
> CMD="$HIVE_HOME/bin/hive -f ${HIVEBENCH_SQL_FILE}"<br>

with

> CMD="hive -f ${HIVEBENCH_SQL_FILE}"<br>
```
#CMD="$HIVE_HOME/bin/hive -f ${HIVEBENCH_SQL_FILE}"
CMD="hive -f ${HIVEBENCH_SQL_FILE}"
```
## Test Hive 3.1
> bin/workloads/sql/aggregation/hadoop/run.sh
## Run all Hadoop benchmarks
> bin/run_all.sh<br>
