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
> mvn -Dspark=2.1 -Dscala=2.11 clean package<br>
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
In tiny environm, the Spark client can fail.
```

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

At the end of **import** section, add
> import java.util.Arrays
```
import org.apache.hadoop.conf.*;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

import java.util.Arrays

/**
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
In HDP 3.1, procedure *copyMerge* from package *FileUtil* is removed, so copy and paste the deprecated method body at the beginning of the Java code, before the first method and after <br>
>  private static Configuration fsConfig = new Configuration();<br>

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

