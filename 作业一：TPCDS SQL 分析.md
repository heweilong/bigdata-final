# 作业一：TPCDS SQL 分析

## 执行语句： q1.sql

**过程中用到SQL优化规则：**ColumnPruning，PullupCorrelatedPredicates，PushDownPredicates，CollapseProject，ConstantFolding，SimplifyCasts，InferFiltersFromConstraints，DecimalAggregates，ReorderJoin，RewriteCorrelatedScalarSubquery，EliminateOuterJoin，InferFiltersFromConstraints

**谓词下推（PushDownPredicates）：**

 谓词下推原理是将条件限制语句下推至数据源，先对原表数据进行条件过滤，进行一层数据筛选之后通过减少数据来源数据规模，然后再进行表关联，这样一来表之间相互关联时需要相互关联的数据量会大大降低，以当前执行SQL为例，其中：D_YEAR = 2000 条件可以在实际执行计划中看到如下语句：

```
!            +- Filter isnotnull(d_date_sk#329)                                                                                                                                                                                                                                                                                                                                                                                                       +- Project [d_date_sk#329]
!               +- Project [d_date_sk#329]                                                                                                                                                                                                                                                                                                                                                                                                               +- Filter ((isnotnull(d_year#335) AND (d_year#335 = 2000)) AND isnotnull(d_date_sk#329))
!                  +- Filter (isnotnull(d_year#335) AND (d_year#335 = 2000))                                                                                                                                                                                                                                                                                                                                                                                +- Relation[d_date_sk#329,d_date_id#330,d_date#331,d_month_seq#332,d_week_seq#333,d_quarter_seq#334,d_year#335,d_dow#336,d_moy#337,d_dom#338,d_qoy#339,d_fy_year#340,d_fy_quarter_seq#341,d_fy_week_seq#342,d_day_name#343,d_quarter_name#344,d_holiday#345,d_weekend#346,d_following_holiday#347,d_first_dom#348,d_last_dom#349,d_same_day_ly#350,d_same_day_lq#351,d_current_day#352,... 4 more fields] parquet
!                     +- Relatio


```

其中： **+- Filter (isnotnull(d_year#335) AND (d_year#335 = 2000))**  project在执行关联之前先执行了一个filter针对DATE_DIM 表中数据过滤d_year字段为2000的值，接着才执行的Relation进行的表关联，谓词下推是一种基础常见的优化手段，也是优化过程中有效提升执行效率的方式。

**列剪枝（ColumnPruning）：**列剪枝本质就是查询过程中减少查询列数，SQL优化器会解析SQL语句提取出实际使用到列，在执行过程中只查询使用到的列，这样的好处是可以减少数据查询量，降低网络IO及数据内存消耗等好处，

```
=== Applying Rule org.apache.spark.sql.catalyst.optimizer.ColumnPruning ===
 Aggregate [count(1) AS count#227L]                                                                                                                                              Aggregate [count(1) AS count#227L]
!+- Relation[c_customer_sk#154,c_customer_id#155,c_current_cdemo_sk#156,c_current_hdemo_sk#157,c_current_addr_sk#158,c_first_shipto_date_sk#159,c_first_sales_date_sk#160,c_salutation#161,c_first_name#162,c_last_name#163,c_preferred_cust_flag#164,c_birth_day#165,c_birth_month#166,c_birth_year#167,c_birth_country#168,c_login#169,c_email_address#170,c_last_review_date#171] 

```



# 作业二：Lambda  架构设计  



另附图片

 Lambda架构把数据分为ServingLayer、SpeedLayer、BatchLayer三层。Batch层主要是对离线数据进行处理目前该层目前主流做法是通过Yarn调度Spark Job，将接入的数据进行预处理、存储，查询的时候直接在预处理结果上查询并不需要再进行完整的计算，最后以View层提供给到业务；在Speed层主要是对实时增量数据进行处理，每来一次新数据就不断的更新View层，提供给到业务；在Serving层主要是响应用户的请求，根据用户需求把Batch层和Speed层的数据集合到一起，得到最终的数据集。

优点：Lambda架构优点是将流处理和批处理分开，很好的结合了实时计算和流计算的优点，架构稳定，实时计算成本可控，提高了整个系统的容错性、降低了复杂性。

缺点：Batch层和Speed层无法共用需要维护两套逻辑，容易出现逻辑单方面修改导致不一致，离线数据和实时数据很难保障数据的一致性，需要维护两套系统成本较高。



# 作业三：Spark Shuffle 的工作原理   

Spark Shuffle实际上就是数据的排序合并由MapReduce Shuffle衍生来，分为Shuffle write和Shuffle read，Spark Shuffle过程：

1. 每个Mapper根据Reduce的数量创建出相应的bucket，bucket的数量是与Reduce个数有关，当执行慢的时还可以考虑调整配置文件增加reduce个数。

2. 其次Mapper产生的结果会根据设置的partition算法填充到每个bucket中去。这里的partition算法是可以自定义的，当然默认的算法是根据key哈希到不同的bucket中去。

3. 当Reducer启动时，它会根据自己task的id和所依赖的Mapper的id从远端或是本地的block manager中取得相应的bucket作为Reducer的输入进行处理。

   Shuffle 过程本质上都是将 Map 端获得的数据使用分区器进行划分，并将数据发送给对应的 Reducer 的过程，由于Shuffle过程涉及数据的网络传输，和数据重载有不小性能开销实际开发过程中需要尽量减少Shuffle过程。

