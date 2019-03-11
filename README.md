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
For user running the benchmark:hibench.hadoop.release
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
The memory problem also failed the *graph.nweight*
