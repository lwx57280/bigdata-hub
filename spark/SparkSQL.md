# SparkSQL

## 1、概述

spark用于处理**结构化数据**的一个模块，提供了除数据之外的额外信息，spark用这些元信息进行优化

特点：
- 易整合：无缝整合了SQL查询和Spark编程
- 统一的数据访问：使用相同的方式连接不同的数据源
- 兼容Hive
- 标准数据连接：使用JDBC或ODBC连接数据库

相关背景：

- SparkSQL前身是Shark，基于Hive所开发的工具，修改了Hive架构中的内存管理、物理计划、执行三个模块，使得能够运行在Spark引擎上
- Shark对于HIve有太多的依赖，制约了Spark的发展。2014年6月1日开始Shark停止更新，全力发展SparkSQL

### Datasets/DataFrame/RDD之间的关系

#### 版本上

- Spark1.0 => RDD

- Spark1.3 => DataFrame

- Spark1.6 => Dataset

  > 在后期的 Spark 版本中，DataSet 有可能会逐步取代RDD和 DataFrame 成为唯一的API 接口



#### 相同点

- 都是Spark下的分布式弹性数据集，为处理超大型数据提供便利
- 三者都有惰性机制
- 三者都有共同的算子函数
- 三者都会根据Spark的内存情况自动缓存运算。即使数据量非常大，也不用担心内存溢出
- 三者都有partition的概念
- DataFrame 和DataSet 均可使用模式匹配获取各个字段的值和类型

#### 区别和联系

- RDD是最底层的抽象，其他两个都是基于RDD做了更高级的封装，更加友好易用
- DataFrame的概念来源于python的pandas，比RDD多了表头，也叫SchemaRDD
    - DataFrame也是懒执行的，性能上比RDD高，主要原因：
      - 使用了优化器Catalyst进行针对性优化：比如filter下推、裁剪等（减少了数据读取）
      - RDD只能在stage层面进行简单、通用的流水线优化
    - 跟hive类型，也支持嵌套数据结构（struct、array、map）
    - DataFrame是一个分布式数据容器，类似于传统数据库中的二维表格
    - DataFrame 每一行的类型固定为Row，需要通过解析才能获取每一列的值
    - DataFrame 与DataSet 一般不与 spark mllib 同时使用
    - DataFrame 与DataSet 均支持 SparkSQL 的操作
    - DataFrame 与DataSet 支持一些特别方便的保存方式，比如保存成 csv
- Datasets是spark1.6版本以后提出，提供了强类型支持，是DataFrame的拓展（DataFrame只知道字段不知道字段的类型），是spark最新的数据抽象。
    - 既有类型安全检查，也具有DataFrame的查询优化特性
    - Dataset 和DataFrame 拥有完全相同的成员函数，区别只是没一行的数据类型不同
    - DataFrame是Dataset的特例 DataFrame = Dataset[Row] Row就是数据结构信息
    - 也会经过优化器Catalyst优化
    - 可以和java bean结合，在编译时就能检查出其他两个在运行才会发现的异常（因为不仅知道字段，还知道字段的类型）
- RDD转为DataFrame是不可逆的，而转为Datasets是可逆的。
```
// 创建Datasets并指定java bean
Encoder<Employee> employeeEncoder = Encoders.bean(Employee.class);
String path = "examples/src/main/resources/employees.json";
Dataset<Employee> ds = spark.read().json(path).as(employeeEncoder);
ds.show();
```




DataFrame/Datasets 结构化数据，是需要有结构的描述信息Schema，也就是元数据，创建方式参见 [SchemaTest.java](https://github.com/fancyChuan/bigdata-learn/blob/master/spark/src/main/java/learningSpark/sparkSQL/SchemaTest.java)

RDD与Datasets在序列化上的区别：
- RDD: 通过Java serialization 或 Kryo 序列化
- Datasets: 通过 Encoder 来序列化对象

Encoder 动态生成代码以便spark各种操作，并在执行计划中做优化？？Encoder也有点像是在指定Datasets中字段的类型，只不过不能使用传统的比如String，而是要用优化过的Encoders.String()

### Datasets/DataFrame/RDD相互转换

<img src="img/RDD、DataFrame和DataSet的关系.png" style="zoom:80%;" />

#### RDD转为Dataset

SparkSQL 能够自动将包含有 case 类的RDD 转换成DataSet，case 类定义了 table 的结构，case 类属性通过反射变成了表的列名。Case 类可以包含诸如 Seq 或者 Array 等复杂的结构

- 利用反射获取Schema信息

- 指定Schema信息
    - 当java bean无法被使用时，需要指定，比如RDD的元素是字符串
    - 使用DataTypes创建Schema信息

```
scala> case class User(name:String, age:Int) defined class User

scala> sc.makeRDD(List(("zhangsan",30), ("lisi",49))).map(t=>User(t._1, t._2)).toDS
res11: org.apache.spark.sql.Dataset[User] = [name: string, age: int]

```

#### RDD转DataFrame  

```
testDF = rdd.toDF("col1", "col2")
```
#### Dataset转DataFrame

实际开发中，一般通过样例类/POJO类来将RDD转为DataFrame

```
testDF = testDS.toDF
```
#### DataFrame转Dataset

DataFrame其实是DataSet的特例，它们之间可以相互转换。

```
testDS = testDF.as[Sechma]
```
#### DataFrame/Dataset转RDD

这两种类型实际上都是对RDD的封装，因此可以直接获取内部的RDD

```
// 注意: 此时得到的RDD存储类型为Row
val rdd1 = testDF.rdd
val rdd2 = testDS.rdd
```



#### 聚合操作

用于Row是无类型的自定义聚合操作
- 需要继承 UserDefinedAggregateFunction 抽象类并实现方法
- 步骤
    - 先定义**输入的数据**、**中间计算结果**和**最终结果** 的结构化信息
    - 实现 initialize() update() merge() evaluate() 几个函数
    - 注册使用
    

用于Row是有结构化的自定义聚合操作
- 继承 Aggregator 并实现方法

两种自定义聚合函数与RDD的aggregate操作的区别于联系

| 对比项   | 无类型                          | 类型安全（类型）   | RDD的聚合操作    |
|-------|------------------------------|------------|-------------|
| 继承类   | UserDefinedAggregateFunction | Aggregator |             |
| 初始值   | initialize()                 | zero()     | new Tuple() |
| 更新操作  | update()                     | reduce()   | 实现接口        |
| 合并操作  | merge()                      | merge()    | 实现接口        |
| 计算返回值 | evaluate()                   | finish()   | 对结果RDD再操作   |

#### 数据源
SparkSQL支持多种数据源
- 基本的load/save操作
```
Dataset<Row> usersDF = spark.read().load("examples/src/main/resources/users.parquet");
usersDF.select("name", "favorite_color").write().save("namesAndFavColors.parquet");
```
- 手动指定读取数据源的类型(json, parquet, jdbc, orc, libsvm, csv, text)和选项
```
Dataset<Row> peopleDF = spark.read().format("json").load("examples/src/main/resources/people.json");
peopleDF.select("name", "age").write().format("parquet").save("namesAndAges.parquet");

Dataset<Row> peopleDFCsv = spark.read().format("csv")
    .option("sep", ";")
    .option("inferSchema", "true")
    .option("header", "true")
    .load("examples/src/main/resources/people.csv");

usersDF.write().format("orc")
    .option("orc.bloom.filter.columns", "favorite_color")
    .option("orc.dictionary.key.threshold", "1.0")
    .option("orc.column.encoding.direct", "name")
    .save("users_with_options.orc");
```
- 直接用SQL读取文件
```
Dataset<Row> sqlDF = spark.sql("SELECT * FROM parquet.`examples/src/main/resources/users.parquet`");
```

数据保存模式


### 常用参数配置
```
-- 设置动态资源分配的最小executor为300
set spark.dynamicAllocation.minExecutors=300;
set spark.dynamicAllocation.maxExecutors=1500;
-- 开启hdfs shuffle功能，将shuffle数据双写到hdfs上，能有效解决集群负载高引起的shuffle失败问题。注意会带来多写一份数据的时间开销和hdfs集群负担，只建议在shuffle数据量较大（单个task shuffle数据量超过1g）的任务开启
set spark.shuffle.hdfs.enabled=true;

-- Adaptive execution开关，包含自动调整并行度，解决数据倾斜等优化。
set spark.sql.adaptive.enabled=true;
-- Spark Adaptive Execution 相关，动态最大的并行度。
set spark.sql.adaptive.maxNumPostShufflePartitions=4093;


set spark.sql.shuffle.partitions=600;
set spark.driver.memory=20g;
set spark.executor.memory=20g;
```



## 2、SparkSQL核心编程

学习重点：使用DataFrame和DataSet进行编程，了解它们之间的关系和相互转换

### 2.1 编程起点：上下文环境对象

- SparkCore中，主要是SparkContext
- 老版本中，SparkSQL提供两种起点：
  - 一个叫 SQLContext，用于 Spark自己提供的SQL 查询；
  - 一个叫HiveContext，用于连接 Hive 的查询
- 新版本中，统一为SparkSession，实质上是SQLContext和HiveContext的组合，所以它两能用的API在SparkSession同样能用
  - SparkSession内部封装了SparkContext，计算实际上是SparkContext完成的
  - spark-shell中，SparkSession被命名为spark，也就是可以在命令行中直接使用spark来使用SparkSession

### 2.2 DataFrame

#### 创建

- 从Spark数据源进行创建，支持的文件格式有以下这些：

```
scala> spark.read.
csv   format   jdbc   json   load   option   options   orc   parquet   schema   table   text   textFile
```

- 从RDD进行转换
- 从Hive Table进行查询返回

#### SQL语法

要使用SQL风格来查询数据，必须要有临时视图或者全局视图

- 临时视图：df.createOrReplaceTempView("people") 是会话范围生效的，当前session失效视图也不可用
- 全局视图：df.createGlobalTempView("people") 被绑定到 global_temp 类似于库，通过global_temp.people调用

> 是整个application生效的

```
scala> spark.sql("SELECT * FROM global_temp.people").show()
+---+--------+
|age|username|
+---+--------+
| 20|zhangsan|
| 30|	lisi|
| 40|	wangwu|
+---+--------+

scala> spark.newSession().sql("SELECT * FROM global_temp.people").show()
+---+--------+
|age|username|
+---+--------+
| 20|zhangsan|
| 30|	lisi|
| 40|	wangwu|
+---+--------+
```

#### DSL语法

DataFrame提供了一个DSL去管理结构化的数据，可以在Scala、Java、Python中使用DSL，使用DSL语法风格就不必去创建临时视图了



注意:涉及到运算的时候, 每列都必须使用$, 或者采用引号表达式：单引号+字段名

```
scala> df.printSchema root
|-- age: Long (nullable = true)
|-- username: string (nullable = true)

scala> df.select($"username",$"age" + 1).show()
scala> df.select('username, 'age + 1).show()
# 过滤age>30
scala> df.filter($"age">30).show
+---+---------+
|age| username|
+---+---------+
| 40|	wangwu|
+---+---------+
# 聚合汇总
scala> df.groupBy("age").count.show
+---+-----+
|age|count|
+---+-----+
| 20|	1|
| 30|	1|
| 40|	1|
+---+-----+

```

##### 聚合操作

常用的聚合操作【java写法】

- 分组计数 df.groupBy("column").count().show()
- 结果按指定字段排序 df.groupBy("column").count().sort("count") // 计数默认字段名为 "count"
- 结果按指定字段排序 df.groupBy("column").count().sort(col("count").desc()) 
- 字段重命名       df.groupBy("column").count().withColumnRenamed("count", "cnt").sort("cnt")
- 字段重命名       df.groupBy("column").agg(count).sort("cnt")
- 直接使用SQL解决： spark.sql("select column, count(1) cnt from df group by column order by cnt desc")



### 2.3 DataSet

DataSet 是具有强类型的数据集合，需要提供对应的类型信息

#### 创建DataSet

- 实际使用最多：使用样例类来创建DataSet

- 使用基本类型的序列创建DataSet（实际用的很少）



```
scala> case class Person(name: String, age: Long) defined class Person

scala> val caseClassDS = Seq(Person("zhangsan",2)).toDS()
caseClassDS: org.apache.spark.sql.Dataset[Person] = [name: string, age: Long] scala> caseClassDS.show
+---------+---+
|	name|age|
+---------+---+
| zhangsan| 2|
+---------+---+

# 使用基本类型的序列
scala> val ds = Seq(1,2,3,4,5).toDS
ds: org.apache.spark.sql.Dataset[Int] = [value: int]

```



