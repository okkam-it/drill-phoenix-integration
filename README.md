# Apache Drill - Phoenix-integration
This git repository has the goal to show how to connect (via JDBC) from **Apache Drill 1.12.0** and **Apache Phoenix 4.13.2** (on Cloudera **CDH 5.11.2**).

At the moment the vanilla release of Apache Drill uses HBase 1.1.2 and the drill-storage-hbase storage plugin is not compatible with Cloudera version (because of Scan.setCaching() that has a different signature). So, in order to make it work Apache Drill HBase plugin must be compiled against the right version of HBase (client actually). 

For this reason, a compatible version of **drill-storage-hbase** has been produced under jars directory. This jar must replace the *vanilla* one. 

In order to make Apache Drill compatible with Cloudera CDH 5.11.2 do the following commands (from the root Drill directory, let's say /home/user/apache-drill-1.12.0):
```
 cd /home/user/apache-drill-1.12.0
 rm jars/3rdparty/hbase-*
 rm jars/3rdparty/hadoop-*
 rm jars/3rdparty/htrace-*
 rm jars/3rdparty/hive-*
 rm jars/drill-storage-hbase-1.12.0.jar
```
And copy the jars of this repo (cloned for example at /home/user/git/drill-phoenix-integration) into Apache Drill:
```
cd /home/user/git/drill-phoenix-integration
cp -r jars /home/user/apache-drill-1.12.0
```

Finally, make hbase-site.xml available to Apache Drill. There are 2 options:
1. export HBASE_CONF_DIR into conf/drill-env.sh
2. copy the hbase-site.xml into the drill conf folder

Now you can start playing with both HBase tables and Phoenix.
Create a Phoenix JDBC storage plugin named "phoenix" (from http://localhost:8047/storage for example) using something like the following configuration (**using the right Phoenix connection URL**):
```
{
  "type": "jdbc",
  "driver": "org.apache.phoenix.jdbc.PhoenixDriver",
  "url": "jdbc:phoenix:myhost1,myhost2:2181/hbase",
  "enabled": true
}
```
The raw HBase connection could be created as well. Create for example a storage plugin named "hbase" using the following config (**using the right connection parameters**):
```
{
  "type": "hbase",
  "config": {
    "hbase.zookeeper.quorum": "myhost1,myhost2",
    "zookeeper.znode.parent": "/hbase",
    "hbase.zookeeper.property.clientPort": "2181"
  },
  "size.calculator.enabled": false,
  "enabled": true
}
```

Now you can query your Phoenix table with both storage plugins (obviously the raw hbase connection will dispay the Phoenix table in a raw format).

Query the CATALOG table via Phoenix plugin:
```
select * from phoenix.catalog LIMIT 10
```
Query the CATALOG table via "raw" HBase plugin (if using namespaces):
```
select * from hbase.`SYSTEM:CATALOG` LIMIT 10
```


## Known issues
At the moment there are a few issues with this solution.

 -  Sometimes, when query a table via Phoenix **without setting a LIMIT clause** or setting this LIMIT "too HIGH" can lead to *IndexOutOfBoundsException* or *Error: DATA_READ ERROR: Failure while attempting to read from database* exceptions. 

For example:

 **select * from hbase.\`TEST:MYTABLE\` LIMIT 1000;**
```
Error: SYSTEM ERROR: IndexOutOfBoundsException: index: 0, length: 1 (expected: range(0, 0))




Caused by: org.apache.drill.common.exceptions.UserException$Builder.build(UserException.java:586) ~[drill-common-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.work.fragment.FragmentExecutor.sendFinalState(FragmentExecutor.java:298) [drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.work.fragment.FragmentExecutor.cleanup(FragmentExecutor.java:160) [drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.work.fragment.FragmentExecutor.run(FragmentExecutor.java:267) [drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.common.SelfCleaningRunnable.run(SelfCleaningRunnable.java:38) [drill-common-1.12.0.jar:1.12.0]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_101]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_101]
	at java.lang.Thread.run(Thread.java:745) [na:1.8.0_101]
Caused by: java.lang.IndexOutOfBoundsException: index: 0, length: 1 (expected: range(0, 0))
	at io.netty.buffer.DrillBuf.checkIndexD(DrillBuf.java:122) ~[drill-memory-base-1.12.0.jar:4.0.48.Final]
	at io.netty.buffer.DrillBuf.chk(DrillBuf.java:146) ~[drill-memory-base-1.12.0.jar:4.0.48.Final]
	at io.netty.buffer.DrillBuf.getByte(DrillBuf.java:797) ~[drill-memory-base-1.12.0.jar:4.0.48.Final]
	at org.apache.drill.exec.vector.UInt1Vector.copyFrom(UInt1Vector.java:372) ~[vector-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.vector.UInt1Vector.copyFromSafe(UInt1Vector.java:382) ~[vector-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.vector.NullableVarBinaryVector.copyFromSafe(NullableVarBinaryVector.java:375) ~[vector-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.vector.NullableVarBinaryVector$TransferImpl.copyValueSafe(NullableVarBinaryVector.java:335) ~[vector-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.vector.complex.MapVector$MapTransferPair.copyValueSafe(MapVector.java:225) ~[vector-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.vector.complex.MapVector.copyFromSafe(MapVector.java:82) ~[vector-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.test.generated.CopierGen1.doEval(CopierTemplate2.java:24) ~[na:na]
	at org.apache.drill.exec.test.generated.CopierGen1.copyRecords(CopierTemplate2.java:59) ~[na:na]
	at org.apache.drill.exec.physical.impl.svremover.RemovingRecordBatch.doWork(RemovingRecordBatch.java:101) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.record.AbstractSingleRecordBatch.innerNext(AbstractSingleRecordBatch.java:97) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.physical.impl.svremover.RemovingRecordBatch.innerNext(RemovingRecordBatch.java:93) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.record.AbstractRecordBatch.next(AbstractRecordBatch.java:164) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.record.AbstractRecordBatch.next(AbstractRecordBatch.java:119) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.record.AbstractRecordBatch.next(AbstractRecordBatch.java:109) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.record.AbstractSingleRecordBatch.innerNext(AbstractSingleRecordBatch.java:51) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.physical.impl.project.ProjectRecordBatch.innerNext(ProjectRecordBatch.java:134) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.record.AbstractRecordBatch.next(AbstractRecordBatch.java:164) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.physical.impl.BaseRootExec.next(BaseRootExec.java:105) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.physical.impl.ScreenCreator$ScreenRoot.innerNext(ScreenCreator.java:79) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.physical.impl.BaseRootExec.next(BaseRootExec.java:95) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.work.fragment.FragmentExecutor$1.run(FragmentExecutor.java:234) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at org.apache.drill.exec.work.fragment.FragmentExecutor$1.run(FragmentExecutor.java:227) ~[drill-java-exec-1.12.0.jar:1.12.0]
	at java.security.AccessController.doPrivileged(Native Method) ~[na:1.8.0_101]
	at javax.security.auth.Subject.doAs(Subject.java:422) ~[na:1.8.0_101]
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1920) ~[hadoop-common-2.6.0-cdh5.11.2.jar:na]
	at org.apache.drill.exec.work.fragment.FragmentExecutor.run(FragmentExecutor.java:227) [drill-java-exec-1.12.0.jar:1.12.0]
	... 4 common frames omitted
```

**select * from phoenix.catalog**
```
Error: DATA_READ ERROR: Failure while attempting to read from database.

sql SELECT *
FROM "SYSTEM"."CATALOG"
plugin phoenix
Fragment 0:0

[Error Id: 59d032cf-d12f-4d54-b8c0-1f804d2d3cee on host1:31011] (state=,code=0)





Caused by: java.sql.SQLException: ERROR 1101 (XCL01): ResultSet is closed.
	at org.apache.phoenix.exception.SQLExceptionCode$Factory$1.newException(SQLExceptionCode.java:488) ~[phoenix-core-4.13.2-cdh5.11.2.jar:na]
	at org.apache.phoenix.exception.SQLExceptionInfo.buildException(SQLExceptionInfo.java:150) ~[phoenix-core-4.13.2-cdh5.11.2.jar:na]
	at org.apache.phoenix.jdbc.PhoenixResultSet.checkOpen(PhoenixResultSet.java:216) ~[phoenix-core-4.13.2-cdh5.11.2.jar:na]
	at org.apache.phoenix.jdbc.PhoenixResultSet.next(PhoenixResultSet.java:773) ~[phoenix-core-4.13.2-cdh5.11.2.jar:na]
	at org.apache.commons.dbcp.DelegatingResultSet.next(DelegatingResultSet.java:207) ~[commons-dbcp-1.4.jar:1.4]
	at org.apache.commons.dbcp.DelegatingResultSet.next(DelegatingResultSet.java:207) ~[commons-dbcp-1.4.jar:1.4]
	at org.apache.drill.exec.store.jdbc.JdbcRecordReader.next(JdbcRecordReader.java:237) [drill-jdbc-storage-1.12.0.jar:1.12.0]
	... 19 common frames omitted


```

