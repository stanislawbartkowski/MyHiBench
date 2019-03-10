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




