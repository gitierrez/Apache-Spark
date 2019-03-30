# Apache-Spark
This repository includes a description of Spark and its best practices for big data processing.

### References

* High Performance Spark: Best Practices for Scaling and Optimizing Apache Spark - Holden Karau & Rachel Warren

---

# Spark

<img src='imgs/spark.png' width=200>

Apache Spark is a high-performance, general-purpose distributed computing system. Spark enables the processing of large quantities of data with a high-level API.

Spark performs computations of Spark JVMs that last only for the duration of a Spark application. Spark is used in tandem with a distributed storage system (HDFS, Cassandra, S3) and a cluster manager (YARN, Apache Mesos).

## Spark Components

**Spark Core** is the main data processing framework in the Spark ecosystem, and includes APIs in Scala, Java, Python and R.

Spark is built around a data abstraction called *Resilient Distributed Datasets* (RDDs), which are a representation of lazily evaluated, statically-typed, distributed collections which have a number of predefined transformations to manipulate the distributed datasets, as well as I/O functionality to read and write data between the distributed storage system and the Spark JVMs.

In addition to Spark Core, the Spark ecosystem includes other first-party components, including **Spark SQL**, **Spark MLlib**, **Spark ML**, **Spark Streaming**, and **GraphX**, which provide more specific data processing functionality.

* **Spark SQL**: can be used in tandem with Spark Core and defines an interface for a semi-structured data type, called DataFrames.
* **Spark MLlib**: RDD-based standard package of machine learning and statistics algorithms.
* **Spark ML**: DataFrame-based higher-level API for easily creating practical machine learning pipelines.
* **Spark Streaming**: uses the scheduling of the Spark Core for streaming analytics on minibatches of data.
* **GraphX**: a graph processing framework with an API for graph computations.

In addition to these components, the community has written a number of libraries that provide additional functionality, some of these are listed at [https://spark-packages.org/](https://spark-packages.org/)

## Resilient Distributed Datasets

Spark allows users to write a program for the *driver* on a cluster computing system that can perform operations on data in parallel. Spark represents large datasets as RDDs (immutable, distributed collections of objects) stored in the *executors* (or slave nodes). The objects that comprise RDDs are called *partitions* and may be computed on different nodes of a distributed system. The *Spark cluster manager* handles starting and distributing the Spark executors across a distributed system according to the configuration parameters set by the Spark application. The Spark execution engine itself distributes data across the executors for a computation. The paradigm of lazy-evaluation, in-memory storage, and immutability allows Spark to be easy-to-use, fault-tolerant, scalable, and efficient.

#### Lazy Evaluation

Spark does not begin computing the partitions until an **action** is called. An action is a Spark operation that returns something other than an RDD, triggering evaluation of partitions and possibly returning some output to system outside of the Spark executors; for example, bringing data back to the driver (with operations like `count` or `collect`) or writing data to an external storage system (with operations like `copyToHadoop`).

Actions trigger the scheduler, which builds a *directed acyclic graph* (DAG), based on the dependencies between RDD transformations.

Lazy evaluation allows Spark to combine operations that don't require communication with the driver to avoid doing multiple passes through the data.

The RDD itself contains all the dependency information needed to replicate each of its partitions. Thus, if a partition is lost, the RDD has enough information about its lineage to recompute it, allowing for fault-tolerance.

#### In-Memory Persistence and Memory Management

Rather than writing to disk between each pass through the data (MapReduce), Spark has the option of keeping the data on the executors loaded into memory.

Spark offers three options for memory management:

1. **In-memory as deserialized Java objects**: fastest, but not the most memory efficient, since it requires the data to be stored as objects.
2. **In-memory as serialized data**: slower, but more memory efficient.
3. **On-disk**: used for RDDs whose partitions are too large to be stored in RAM of each of the executors. Obviously is the slowest.

The `persist()` function in the `RDD` class lets the user control how to RDD is stored.

#### Immutability and RDD Interface

RDDs are statically typed and immutable, calling a transformation on one RDD will not modify the original RDD but rather return a new RDD object.

RDDs can be created in three ways:

1. By transforming an existing RDD.
2. From a `SparkContext`, which is the API's gateway to Spark for your application.
3. By converting a DataFrame or Dataset (created from the `SparkSession`).

The `SparkContext` represents the connection between a Spark cluster and one running Spark application. The `SparkSession` is the Spark SQL equivalent to a `SparkContext`.

Internally, Spark uses five main properties to represent an RDD:

1. The list of partition objects that make up the RDD - `partitions()`
2. A function for computing an iterator of each partition - `iterator()`
3. A list of dependencies on other RDDs - `dependencies()`
4. A partitioner for RDDs of rows of key/value pairs - `partitioner()`
5. A list of preferred locations for the HDFS file - `preferredLocations()`

There are two types of functions defined on RDDs: *actions* and *transformations*. Actions are functions that return something that is not an RDD, and transformations are functions that return another RDD.

## Spark Job Scheduling

A Spark application consists of a driver process, which is where the high-level Spark logic is written, and a series of executor processes that can be scattered across the nodes of a cluster. The Spark program itself runs in the driver node and sends some instructions to the executors. One Spark cluster can run several Spark applications concurrently. The applications are scheduled by the cluster manager and correspond to one `SparkContext`. Spark applications can, in turn, run multiple concurrent jobs. Jobs correspond to each action called on an RDD in a given application.

## Spark Application

A Spark application corresponds to a set of Spark jobs defined by one `SparkContext` in the driver program. A Spark application begins when a `SparkContext` is created. When the `SparkContext` is started, a driver and a series of executors are started on the worker nodes of the cluster. Each executor is its own JVM.

The `SparkContext` determines how many resources are allotted to each executor, according to the configuration parameters exposed on the `SparkConf` object, which is used to created the `SparkContext`.


>RDDs cannot be shared between applications. Thus transformations, such as join, that use more than one RDD must have the same SparkContext

## Spark Execution Model

A Spark job is the set of RDD transformations needed to compute one final result. Each stage corresponds to a segment of work, which can be accomplished without moving data across the partitions. Within one stage, tasks are the units of work done for each partition of the data.

<img src='imgs/spark_execution.png' width=600>

---

# SparkSQL

Highly performant interfaces, with more efficient storage options, advanced optimizer, and direct operations on serialized data.

DataFrames have a specialized representation and columnar cache format, which allows them to be more space efficient and encode faster than Kryo serialization. Tungsten is a Spark SQL component that provides more efficient Spark operations by working directly at the byte level, including specialized in-memory data structures tuned for the types of operations required by Spark, improved code generation and a specialized wire protocol.

SparkSQL allows specifying behaviors to use when writing out to a path that may already have data:

|Save Mode|Behavior|
|:---|:---|
|`ErrorIfExists`|Throws an exception if the target already exists. If target doesn't exist write the data out.|
|`Append`|If target already exists, append the data to it. If the data doesn't exist write the data out.|
|`Overwrite`|If the target already exists, delete the target. Write the data out.|
|`Ignore`|If the target already exists, silently skip writing out. Otherwise write out the data.|

For example:

```
df.write.mode(SaveMode.Append).save('output/')
```

Remember that the SparkSession is the Core equivalent to the SparkContext.

To create a SparkSession:

```
val session = SparkSession.builder().enableHiveSupport().getOrCreate()
```

When using `getOrCreate()`, if an existing session exists you will get the existing `SparkSession`.

Loading JSON data as a DataFrame:

```
val df = session.read.json(path)
```

Spark SQL automatically handles the schema definition, either **inferred** when loading data or **computed** based on the parent DataFrames and the transformation being applied.

### Schemas

DataFrames expose the schema in both human-readable and programmatic formats through the `df.printSchema()` and `df.schema`, respectively.

The programmatic schema is returned as a `StructField` with the fields:

```
name: String,
dataType: DataType,
nullable: Boolean = true
metadata: Metadata = Metadata.empty
```

Spark SQL allows the following data types:


|Spark SQL Type|Details|
|:--- |:---|
|ByteType|1-byte signed integers (-128, 127)|
|ShortType|2-byte signed integers (-32768, 32767)|
|IntegerType|4-byte signed integers (-2147483648, 2147483647)|
|LongType|8-byte signed integers (–9223372036854775808, 9223372036854775807)|
|DecimalType|Arbitrary precision signed decimals|
|FloatType|4-byte floating-point number|
|DoubleType|8-byte floating-point number|
|BinaryType|Array of bytes|
|Boolean|True/False|
|DateType|Date without time information|
|TimestampType|Date with time information (up to second precision)|
|StringType|Character string values (stored as UTF8)|
|ArrayType(elementType, containsNull)|Array of single type of element|
|MapType(elementType, valueType, valueContainsNull)|Key/value map, valueContainsNull if any values are null|
|StructType(List[StructFields])|Named fields of possible heterogeneous types|

>Schemas are eagerly evaluated, unlike the data underneath. If you find yourself in the shell and uncertain of what a transformation will do, try it and print the schema.

### Transformations

DataFrame transformations can be broken down into four types:

1. Single DataFrame transformations.
2. Multiple DataFrame transformations.
3. Key/value transformations.
4. Grouped/windowed transformations.

#### Single DataFrame transformations

These allow us to do most of the standard things one can do when working a row at a time (narrow transformations with no shuffle). You can still do many of the same operations defined on RDDs, except using Spark SQL expressions instead of arbitrary functions.

Example of a complex filter (`isHappy` is a boolean field and `attributes` is an Array):

```
pandaInfo.filter(
    pandaInfo("isHappy").and(pandaInfo("attributes")(0) > pandaInfo("attributes")(1))
)

```

Spark SQL column operators are defined on the column class, and contains a very large set of available operators, listed in [org.apache.spark.sql.Column](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Column). In addition to these operators directly specified on the column, an even larger set of functions on columns exist in [org.apache.spark.sql.functions](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$).

When constructing a sequence of operations, the `as` operator is useful to specify the resulting column name. For example:

```
pandaInfo.select(
    (pandaInfo("attributes")(0) / pandaInfo("attributes")(1)).as("squishyness")
)
```

If/Else transformation in Spark SQL:

```
pandaInfo.select(
    pandaInfo("id"),
        (when(pandaInfo("pt") === "giant", 0).
         when(pandaInfo("pt") === "red", 1).
         otherwise(2)).as("encodedType")
)
```

To handle missing and noisy data you can use the functions in [DataFrameNaFunctions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameNaFunctions).

GroupBy and Aggregations:

```
pandas.groupBy(pandas("zip")).max("pandaSize")
```

Summary statistics of the numeric fields of a DataFrame can be accessed using the `describe` method:

```
pandas.describe()
```

For multiple or complex aggregations, you should use the `agg` API on the `GroupedData`. For this API you either supply:

* A list of aggregate expressions
* A string representing the aggregates
* A map of column names to aggregate function names

Aggregation functions can be found in [org.apache.spark.sql.functions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.functions$).

Some useful aggregation functions are listed below:

```
approxCount
Distinct
avg
count
countDistinct
first
last
stddev
stddev_pop
sum
sumDistinct
min
max
mean
```

Window functions are also available, specified through the Window object:

```
val windowSpec = Window
     .orderBy(pandas("age"))
     .partitionBy(pandas("zip"))
     .rowsBetween(start = -10, end = 10)
```

Once defined, aggregation functions can be computed over the window.

```
val pandaRelativeSizeCol = pandas("pandaSize") - avg(pandas("pandaSize")).over(windowSpec)
 
pandas.select(pandas("name"), 
              pandas("zip"), 
              pandas("pandaSize"), 
              pandas("age"), 
              pandaRelativeSizeCol.as("panda_relative_size"))
```

Sorting example:

```
pandas.orderBy(pandas("pandaSize").asc, pandas("age").desc)
```

### Multiple DataFrame transformations

These are operations that depend on multiple DataFrames, for example the different types of joins.

These operations tend to be expensive on Spark; sometimes it's better to use regular SQL queries. If you are connected to a Hive Metastore we can directly write SQL queries against the Hive tables and get the results as a DataFrame.

## Interacting with Hive Data

If you have a DataFrame you want to write SQL queries against, you can register it as a temporary table. For example:

```
df.registerTempTable('temp_table')
df.write.saveAsTable('perm_table')
```

Querying works the same, regardless of whether it is a temporary table, existing Hive table or newly saved Spark table:

```
sqlContext.sql('SELECT * FROM temp_table')
```

In addition to registering tables, you can also write queries directly against a specific file path:

```
sqlContext.sql('SELECT * FROM parquet.`path_to_parquet_file`)
```

## Partitions

Partitioning data is an important part of Spark SQL since it powers one of the key optimizations to allow reading only the required data.

When reading partitioned data, you point Spark to the root path of your data, and it will discover the different partitions. Only strings and numeric data can be partitioned.

To save a partitioned data frame:

```
df.write.partitionBy('partition_field').format('csv').save('output/')
```

## UDFs and UDAFs

>When using UDFs or UDAFs written in non-JVM languages, such as Python, it is important to note that you lose much of the performance benefit, as the data must still be transferred out of the JVM. To avoid this penalty, you can write your UDFs in Scala and register them for use in Python.

Registering UDFs is simple: write a regular function and register it using `sqlContext.udf.register('func_name', function)`

If registering a Java or Python UDF you also need to specify your return type.

Registering UDAFs is significantly harder, since you have to extend the UserDefinedAggregateFunction class and implement a number of different functions.

---

# Joins

*The cross product of big data and big data in an out-of-memory exception*.

## Core (RDD) Joins

In general, joins are expensive since they require that corresponding keys from each RDD are located at the same partition so that they can be combined locally.

* If the RDDs do not have known partitioners, they will need to be shuffled so that both RDDs share a partitioner, and data with the same keys lives in the same partitions.
* If the RDDs have the same partitioner, the data may be colocated, so as to avoid network transfer.

>Two RDDs will be colocated if they have the same partitioner and were shuffled as part of the same action. If the RDDs are colocated, both the network transfer and the shuffle can be avoided.

<img src='imgs/colocation.png' width=600>

In order to join data, Spark needs the data that is to be joined to live on the same partition. The default implementation of a *join* in Spark is a *shuffled hash join*, which partitions the second dataset with the same default partitioner as the first, so that the keys with the same hash value from both datasets are in the same partition. This shuffling can be avoided through:

#### Known-partitioner join

This strategy assigns a known partitioner to the  `aggregateByKey` or `reduceByKey` step.

```
def joinScoresWithAddress3(scoreRDD: RDD[(Long, Double)],
    addressRDD: RDD[(Long, String)]) : RDD[(Long, (Double, String))]= {
    // If addressRDD has a known partitioner we should use that,
    // otherwise it has a default hash partitioner, which we can reconstruct by
    // getting the number of partitions.
    val addressDataPartitioner = addressRDD.partitioner match {
        case (Some(p)) => p
        case (None) => new HashPartitioner(addressRDD.partitions.length)
    }
    val bestScoreData = scoreRDD.reduceByKey(addressDataPartitioner, (x, y) => if(x > y) x else y)
    bestScoreData.join(addressRDD)
}
```

#### Broadcast hash joins

A broadcast hash join pushes one of the RDDs (the smaller one) to each of the worker nodes. Then it does a map-side combine with each partition of the larger RDD.

If one of your RDDs can fit in memory or can be made to fit in memory it is always beneficial to do a broadcast hash join, since it doesn't require a shuffle.

```
def manualBroadCastHashJoin[K : Ordering : ClassTag, V1 : ClassTag,
V2 : ClassTag](bigRDD : RDD[(K, V1)],
    smallRDD : RDD[(K, V2)])= {
    val smallRDDLocal: Map[K, V2] = smallRDD.collectAsMap()
    bigRDD.sparkContext.broadcast(smallRDDLocal)
    bigRDD.mapPartitions(iter => {
        iter.flatMap{
            case (k,v1 ) =>
                smallRDDLocal.get(k) match {
                    case None => Seq.empty[(K, (V1, V2))]
                    case Some(v2) => Seq((k, (v1, v2)))
                }
        }
    }, preservesPartitioning = true)
    }
//end:coreBroadCast[]
}
```

#### Partial manual broadcast hash joins

Sometimes not all of our smaller RDD will fit into memory, but some keys are so overrepresented in the large dataset that you want to broadcast just the most common keys. This is especially useful if one key is so large that it can't fit on a single partition. 

In this case you can use `countByKeyApprox` on the large RDD to get an idea of which keys would most benefit from a broadcast. You can then filter the smaller RDD for only these keys, collecting the result locally in a HashMap. Using `sparkContext.broadcast` you can broadcast the HashMap so that each worker only has one copy and manually perform the join against the HashMap. Using the same HashMap you can then filter your large RDD down to not include the large number of duplicate keys and perform your standard join, unioning it with the result of your manual join.

## Spark SQL Joins

Spark SQL supports the same basic join types as core Spark, but the optimizer does more of the heavy lifting for you - trading off some control.

Spark SQL can reorder operations to make joins more efficient, however you can't manually avoid shuffles since you don't contrl the partitioners.

#### Basic joins

```
inner
left_outer
left_anti #returns records in the left table that are not present in the right table
right_outer
full_outer
left_semi #returns records in the left table that are present in the right table
```

For example:

```
df1.join(df2, df1("name") === df2("name"), "inner")
```

#### Self-joins

Is a join of a table with itself - requires renaming to avoid name collision:

```
val joined = df.as("a").join(df.as("b")).where($"a.name" === $"b.name")
```

#### Broadcast hash joins

```
df1.join(broadcast(df2), "key"))
```

---

# Transformations

Most commonly, Spark programs are structured on RDDs: they involve reading data from stable storage into the RDD format, performing a number of computations and data transformations on the RDD and writing the result RDD to stable storage or collecting to the driver. Thus, most of the power of Spark comes from its transformations: operations defined on RDDs that return RDDs.

## Narrow vs Wide Transformations

To summarize: *wide* transformations require a shuffle, while *narrow* transformations do not.

In *narrow* transformations, the child partitions (the partitions in the resulting RDD) depend on a known subset of the parent partitions. In *wide* transformations, multiple child partitions may depend on multiple partitions of the parent RDD.

<img src='imgs/narrow-v-wide.png' width=600>

Narrow dependencies do not required data to be moved across partitions. Consequently, narrow transformations don't require communication with the driver node. Each series of narrow transformations can be computed in the same *stage* of the query execution plan. In contrast, transformations with wide dependencies may require data to be moved across partitions. Thus, the downstream computations cannot be computed before the shuffle finishes.

Another important performance consequence is that the stages associated with one RDD must be executed in sequence. Thus, not only shuffles are expensive since they require data movement, they also limit parallelization.

Additionally, the cost of failure for a transformation with wide dependencies is much higher than for one with narrow dependencies, since it requires more partitions to be recomputed.

> Chaining together transformations with wide dependencies only increases the risk of a very expensive recomputation, particularly if any of the wide transformations have a high probability of causing memory errors. In some instances, the cost of recomputation may be high enough that it is worth checkpointing an RDD, so the intermediate results are saved.

## Type of Returned RDD

RDD are abstracted in two ways:

* They can be of almost any arbitrary type of record (String, Row, Tuple).
* They can be members of one of several implementations of the RDD interface with varying properties.

These distinctions are important because:

* Some transformations can only be applied to RDDs with certain record types.
* Each transformation returns one of the several implementations of the RDD interface, and therefore the same transformation called on two different RDD implementations may be evaluated differently.

> Some RDD implementations retain information about the ordering or locality of the records in the RDD from previous transformations. Understanding the data locality and partitioning information associated with the resulting RDD of a transformation can help avoid unnecessary shuffles.

## Minimizing Object Creation

Garbage collection can slow down the execution of Spark jobs. To avoid this:

* Reuse existing objects
* Use smaller data structures (primitives instead of Objects)

## Iterator-to-Iterator Transformations

The `mapPartitions` transformation is one of the most powerful in Spark since it lets the user define an arbitrary routine on one partition of data. It takes a function from an `iterator` of records to another `iterator` of records.

To allow Spark the flexibility to spill *some* records to disk, it is important to represent your functions inside of `mapPartitions` in such a way that your functions **do not force loading the entire partition in-memory** (like implicitly converting to a list). Iterators have many methods we can use to write functional-style transformations. You may also construct your own custom iterator extending the `Iterator` interface.

When a transformation directly takes and returns an `iterator` without forcing it through another collection, we call it an *iterator-to-iterator* transformation.

> An `iterator` is not actually a collection, but a function that defines a process of accessing the elements in a collection one by one. Not onyl are iterators immutable, but the same element in an iterator can only be accessed once. Since the iterator can only be traversed once, any of the iterator methods that require looking at all the elements in the iterator will leave the original iterator empty.

The primary advantage of using iterator-to-iterator transformations in SPark is that their transformations allow Spark to selectively spill data to disk, saving disk I/O and the cost of recomputation.

## Shared Variables

Spark has two types of shared variables: *broadcast variables* and *accumulators*, each of which can only be written in one context (driver or worker, respectively) and read in the other.

Broadcast variables can be written in the driver program and read on the executors, whereas accumulators are written onto the executors and read on the driver.

#### Broadcast variables

Broadcast variables give us a way to take a local value on the driver and distribute a read-only copy to each machine rather than shipping a new copy with each task. These variables are especially usefull when the same broadcasting variable is used in several transformations.

> The value of a broadcast variable should not be modified after the variable has been created since existing workers will not see the updates and new workers may see the new value. The value for a broadcast variable must be a local, serializable value: no RDDs or other distributed data structures.

#### Accumulators

Accumulators allow us to collect by-product information from a transformation or action on the workers and then bring the result back to the driver. Spark adds to accumulators only once the computation has been triggered. If the computation happens multiple times, Spark will update the accumulator each time.

> Accumulators can be unpredictable. In their current state, they are best used where potential multiple counting is the desired behavior.

## Reusing RDDs

Spark offers several options for RDD reuse, including persisting, caching and checkpointing. However, storing RDD for reuse breaks pipelining, which can be a waste if the stored RDD is only used once or if the transformation is inexpensive to recompute. For some specific Spark programs, reusing an RDD can lead to performance gains, both in terms of speed and failure reduction.

When to reuse RDDs?

* When doing transformations that use the same parent RDD multiple times: persist to keep the RDD partitions in-memory for future transformations.
* When doing multiple actions on the same RDD: persist or checkpoint to break the RDD's lineage, so the same series of transformations preceding the persistance is executed only once.
* When the cost to compute each partition is very high: reuse the RDD after the expensive computation.

In-memory persisting is space-intensive in memory, increasing risk of memory failures and takes time to serialize and deserialize data.

Persisting to disk has the disadvantages of MapReduce, causing expensive write and read operations.

> Avoid at all costs persisting between narrow transformations - since it breaks the lineage and forces Spark to do several passes through the data.

> RDDs that are persisted are not automatically unpersisted when they are no longer going to be used downstream. Instead, RDDs stay in memory for the duration of the Spark application, until the driver program calls the function `unpersist()` or when memory/storage pressure causes their eviction. If you want to take a persisted RDD out of memory to free up space, use `unpersist()`.

### Persistance and Cache Reusing

Useful to avoid recomputation and breaking RDDs with long lineages, since these persistance options keep an RDD on the executors. Spark provides several storage levels:

* `useDisk`: partitions that do not fit in memory will be written to disk.
* `useMemory`: the RDD will be stored in-memory or be directly written to disk.
* `useOfHeap`: the RDD will be stored outside of the Spark executor in an external system.
* `deserialized`: the RDD will be stored as deserialized Java objects.
* `replication`: controls the number of copies of the persisted data to be stored in the cluster. Ensures faster fault tolerance, but incurs in increased space and speed costs. Should only be used when failures are very common.

> The RDD operation `cache()` is equivalent to the persist operation with no storage level argument, i.e., `persist()`. Both `cache()` and `persist()` persist the RDD with the default storage-level `MEMORY_ONLY`, which is equivalent to `StorageLevel(false, true, false, true)`, which stores RDDs in-memory as deserialized Java objects, does not write to disk as partitions get evicted and doesn't replicate partitions.

### Checkpointing

Useful to prevent failures and a high cost of recomputation by saving intermediate results.

Checkpointing writes the RDD to an external storage system such as HDFS or S3, and forgets the RDD's lineage. Since checkpointing requires writing the RDD outside of Spark, checkpointed information survives beyond the duration of a single Spark application and forces evaluation of an RDD.

<img src='imgs/persisting.png' width=600>

---

# Key/Value Data

Anytime we want to perform grouped operations in parallel or change the ordering of records amongst machines (by computing an aggregation statistic or merging customer records), the key/value functionality of Spark is useful, as it allows us to easily parallelize our work.

> Be careful when using `groupByKey()`, since it returns an iterable for each key (which is not distributable), making shuffle reads very expensive. Instead, try to use aggregation operations that can do some map-side reduction to decrease the number of records by key before shuffling, for example `aggregateByKey()` or `reduceByKey()`.

For a single RDD, the following aggregation functions for key/value data are available:

* `groupByKey`
* `combineByKey`
* `aggregateByKey`
* `reduceByKey`
* `foldByKey`

## Multiple RDD Operations

Some operations, other than joins can be operated on multiple RDD inputs.

#### Co-Grouping

The `cogroup` function uses the `CoGroupedRDD` data type. A `CoGroupedRDD` is created from a sequence of key/value RDDs, each with the same key type. `cogroup` shuffles each of the RDDs so that the items with the same value from each of the RDDs will end up on the same partition and into a single RDD by key.

The resulting RDD will have a key and a tuple of iterators containing the values of each cogrouped RDD. `cogroup` can be used as a more efficient alternative than joins when joining multiple RDDs.

## Partitioners and Key/Value Data

A partition in Spark represents a unit of parallel execution that corresponds to one task. An RDD without a known partitioner will assign data to partitions according only to the data size and partition size. The partitioner object defines a mapping from the records in an RDD to a partition index. By assigning a partitioner to an RDD, we can guarantee something about the records on each partition - for example, that it falls within a given range (range partitioner) or includes only elements whose keys have the same hash code (hash partitioner).

There are three methods that exist exclusively to change the way an RDD is partitioned. For RDDs of a generic record type, `repartition` and `coalesce` can be used to simply change the number of partitions that the RDD uses, irrespective of the value of the records in the RDD. For RDDs of key/value pairs, we can use a function called `partitionBy`, which takes a partition object rather than a number of partitions and shuffles the RDD with the new partitioner.

`repartition` shuffles the RDD with a hash partitioner and the given number of partitions. `coalesce` is an optimized version of `repartition` that avoids a full shuffle if the desired number of partitions is less than the current number of partitions. `partitionBy` allows for much more control in the way that the records are partitioned since the partitioner supports defining a function that assigns a partition to a record based on the value of that key.

`repartition` and `coalesce` do not assign a known partitioner to the RDD. In contrast, `partitionBy` results in an RDD with a known partitioner. In some instances, when an RDD has a known partitioner, Spark can rely on the information about data locality provided by the partitioner to avoid doing a shuffle even if the transformation has wide dependencies.

Smart partitioning can be used to minimize the number of times a program causes a shuffle, the distance the data has to travel in a shuffle, and the likelihood that a disproportionate amount of data will be sent to one partition, thus causing out-of-memory errors or straggler tasks.

> When simply reading RDDs from storage, Spark does not know what the underlying partitioning is and therefore can't take advantage of this information, unless the stored RDD was checkpointed, since in the cases of checkpoint persistance, Spark saves some metadata about the RDD including its partitioner (when available).

## The Partitioner Object

The partitioner defines how records will be distributed and thus which records will be completed by each task. Practically, a partitioner is an interface with two methods:

* `numPartitions`: defines the number of partitions in the RDD after partitioning.
* `getPartition`: defines a mapping from a key to the integer index of the partition where records with that key should be sent.

Spark implements two partitioner objects: the `HashPartitioner` and `RangePartitioner`. If neither of these suit your needs, it is possible to define a custom partitioner.

#### HashPartitioner

Determines the index of the child partition based on the hash value of the key. Requires a `partitions` parameter, which determines the number of partitions in the output RDD and the number of bins used in the hashing function.

If unspecified, Spark uses the `spark.default.parallelism` value in the `SparkConf` to determine the number of partitions. If this value in unset, Spark defaults to the largest number of partitions that the RDD has had in its lineage.

#### RangePartitioner

Assigns records whose keys are in the same range to a given partition. Is required for sorting since it ensures that by sorting records within a given partition, the entire RDD will be sorted. If there are too many duplicate keys for all the records associated with one key to fit on one executor, then it may cause memory errors. 

To create a `RangePartitioner` you must specify the number of partitions and the actual RDD to sample. The RDD must be a tuple and the keys must have an ordering defined.

#### Custom Partitioning

To define a unique function for partitioning the data, Spark allows the user to define a custom partitioner. In order to define a partitioner you must implement the following methods:

* `numPartitions`: a method that returns an integer number of partitions greater than zero.
* `getPartition`: a method that takes a key (of the same type of the RDD being partitioned) and returns an integer representing the index of the partition that specifies where records with that key belong. The integer must be between zero and the number of partitions defined.
* `equals` (optional): a method to define equality between partitioners. Is useful to avoid unnecessary shuffles when the data is already partitioned according to the partitioner. For example, the equality method for a `HashPartitioner` returns true if the number of partitions are equal. The `RangePartitioner` does so only if the range bounds are equal.
* `hashcode` (required only if `equals` has been overriden): the hashcode of a `HashPartitioner` is simply its number of partitions. The hashcode of a `RangePartitioner` is a hash function derived from the range bounds.

------

Unless a transformation is known to only change the value part of the key/value pair in Spark, the resulting RDD will not have a known partitioner. Some common transformations like `map` and `flatMap` can change the key. If we don't want to modify the keys and keep the partitioner, we can use the `mapValues` function or the `mapPartitions` function with the `preservesPartitioning` flag set to true. 

## Co-Located and Co-Partitioned RDDs

Co-*located* RDDs are RDDs with the same partitioner that reside in the same physical location in memory. All of the CoGroupedRDD functions (a category which includes `cogroup` and `join` operations) require the RDDs being co-located.

RDDs can only be combined without any network transfer if they have the same partitioner and if each of the corresponding partitions in-memory are on the same executor. Partitions will be in-memory on the same executor if they were partitioned in the lineage associated with the same job.

Co-*partitioned* RDDs are RDDs that are partitioned by the same known partitioner.

-------

PairRDDs or key/value pairs have the following operations available:
``` 
mapValues
flatMapValues
keys
values
sampleByKey
```

OrderedRDDs have the following operations available:
```
sortByKey
repartitionAndSortWithinPartitions
filterByRange
```

---