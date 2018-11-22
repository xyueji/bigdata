# Phoenix二级索引
### 为什么需要二级索引？
对于HBase而言，如果想精确地定位到某行记录，唯一的办法是通过rowkey来查询。如果不通过rowkey来查找数据，就必须逐行地比较每一列的值，即全表扫瞄。对于较大的表，全表扫描的代价是不可接受的。但是，很多情况下，需要从多个角度查询数据。
所以，需要secondary index（二级索引）来完成这件事。
### 配置Hbase支持Phoenix二级索引
如果要启用phoenix的二级索引功能，需要对HMaster以及每一个RegionServer上的hbase-site.xml进行额外的配置。首先，在每一个RegionServer的hbase-site.xml里加入如下属性：
```text
<property> 
  <name>hbase.regionserver.wal.codec</name> 
  <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value> 
</property>
 
<property> 
  <name>hbase.region.server.rpc.scheduler.factory.class</name>
  <value>org.apache.hadoop.hbase.ipc.PhoenixRpcSchedulerFactory</value> 
  <description>Factory to create the Phoenix RPC Scheduler that uses separate queues for index and metadata updates</description> 
</property>
 
<property>
  <name>hbase.rpc.controllerfactory.class</name>
  <value>org.apache.hadoop.hbase.ipc.controller.ServerRpcControllerFactory</value>
  <description>Factory to create the Phoenix RPC Scheduler that uses separate queues for index and metadata updates</description>
</property>
 
<property>
  <name>hbase.coprocessor.regionserver.classes</name>
  <value>org.apache.hadoop.hbase.regionserver.LocalIndexMerger</value> 
</property>
```
注：如果没有在每个regionserver上的hbase-site.xml里面配置如上属性，那么使用create index语句创建二级索引将会抛出如下异常：
```text
Error: ERROR 1029 (42Y88): Mutable secondary indexes must have the hbase.regionserver.wal.codec property set to org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec in the hbase-sites.xml of every region server tableName=TEST_INDEXES (state=42Y88,code=1029)

```
然后在每一个master的hbase-site.xml里加入如下属性：
```text
<property>
  <name>hbase.master.loadbalancer.class</name>                                     
  <value>org.apache.phoenix.hbase.index.balancer.IndexLoadBalancer</value>
</property>
 
<property>
  <name>hbase.coprocessor.master.classes</name>
  <value>org.apache.phoenix.hbase.index.master.IndexMasterObserver</value>
</property>
```
完成上述修改，重启HBase。
### 索引类型
#### 覆盖索引（Covered Indexes）
覆盖索引：只需要通过索引就能返回所要查询的数据，所以索引的列必须包含所需查询的列(SELECT的列和WHRER的列)
#### 函数索引（Functional Indexes）
从Phoeinx4.3以上就支持函数索引，其索引不局限于列，可以合适任意的表达式来创建索引，当在查询时用到了这些表达式时就直接返回表达式结果。
#### 全局索引（Global Indexes）
全局索引适用于多读少写的场景，在写操作上会给性能带来极大的开销，因为所有的更新和写操作（DELETE,UPSERT VALUES和UPSERT SELECT）都会引起索引的更新,在读数据时，Phoenix将通过索引表来达到快速查询的目的。
#### 本地索引（Local Indexes）
本地索引适用于写多读少，空间有限的场景，和全局索引一样，Phoneix在查询时会自动选择是否使用本地索引，使用本地索引，为避免进行写操作所带来的网络开销，索引数据和表数据都存放在相同的服务器中，当查询的字段不完全是索引字段时本地索引也会被使用，与全局索引不同的是，所有的本地索引都单独存储在同一张共享表中，由于无法预先确定region的位置，所以在读取数据时会检查每个region上的数据因而带来一定性能开销。
