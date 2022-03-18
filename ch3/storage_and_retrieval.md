# CHAPTER 3: Storage and Retrieval

# Data Structures That Power Your Database
* 用一个 text file 来完成一个简单的 kv store。所有的 record 都是 append only 的。
    * pro：write is efficient
    * con: when data set is large, **hard to query**!
* 解决办法：add an **index** 
    * pro: query speed up
    * con: index is an additional metadata -> slow down writes
* 所以，DB 不会 index everything，它会让 developer 自己选需要 index 的 part（which is often queried）

## Hash Indexes

### simplest indexing strategy: in-memory hash map(hash table)
    * append a new key-value pair
    * update the hash map to reflect the offset(location) in the data file
    * when you look up a value -> use hash map to find the offset

![](img/1.png)

### in-memory hashmap 应用场景 （Bitcask）
* value is updated frequently

### 如何避免 disk run out of space?
* break the log into segments -> frequently write to new segment file
* perform log compaction

![](img/2.png)

* merge several segments together when performing compaction

![](img/3.png)

### Some details

1. File Format
    * binary format -> faster and simpler

2. Deleting records
    * use **tombstone** label deleted records

3. Crash recovery (**in-memory hashmap crashed 了怎么办？**)

    * reading entire segment files from beginning to end
    * update the most recent offset (rebuild in-memory hash map)
    * better: storing a snapshot for each segment's hash map

4. Partially written records(**DB 写到一半挂了怎么办**)
    * Bitcask file + checksums
    * delete and ignore partially written records

5. Concurrency control
    * data file segments are append-only and immutable -> **read concurrent**
    * mostly only one write thread 

### 为啥不直接 overwrite old value？

* appending and segment merging is much faster than random writes
* concurrency and crash recovery are much simpler if segment files are append only or immutable
* merging old segments avoids the problem of files getting fragmented over time

### Hash table index limitations

* hash table must fit in memory
* range queries are not efficient

---------

## SSTables and LSM-Trees

### SSTable
* 紧接着上一个 section，把 log files 里面的 key sorted 存起来的 format，叫 Sorted String Table（SSTable）
    * each key only appears once within each merged segment file

### SSTables 的 advantages over log segments

1. Merging segments is simple and efficient - **MergeSort**

![](img/4.png)

> 如果 same key 出现在 several input segments 怎么办？
    
    * keep the one from most recent
    * discard values in old segments

2. 不需要为了查找一个 key 把所有的 keys 都存在 memory
 
    * 大概定位 key 的位置
    * 在对应的 range 里，再查找 key
    * **注意**：SSTable 是存在 disk 的，那么还需要一个 in-memory index - memtable 来快速定位 range

![](img/5.png)

3. group a range of records in block and compress it
    * save disk space
    * save I/O

### storage engine workflow using SSTables

* 如何serve write 呢？write come in -> add it to an in-memory balanced tree (e.g. red-black tree), we call it **memtable**
* memtable grows bigger than threshold -> write it disk as an **SSTable file** (所以 memtable 是在 memory 存着，SSTable 是在 disk)
* 如何 serve read 呢？read request -> look into memtable -> most recent on disk segment -> next on disk segment, etc.
* merge and compact segment files in the background

### 这个 workflow 的一个 problem？
* 如果 DB crash 了，在 memtable 的 records 还没来得及 write to disk, 就 lost 了。
* Solution：every write appended in memtable -> keep a log on disk, 这个 log 可以是不 sorted 的，只是为了防止 memtable records lost 和用来 recover

### Making an LSM-tree out of SSTables

* **Cassandra** and **HBase** 就是用 Memtable + SSTable 的 idea 实现的
* 最早这个 indexing structure 叫 log-structured merge tree(**LSM-Tree**)
    * Lucene（used by ElasticSearch and Solr） 也是用 LSM Tree 实现的

### LSM-tree 的一个 problem？
* 如果 DB 没有要 search 的 key，那么就会很慢！
* Solution：**Bloom Filters**。quickly tell you whether a key is existed or not

### Different strategies for SSTables compacted and merged

#### size-tiered compaction (HBase, Cassandra)
* newer and smaller SSTables are successively merged into older and larger SSTables

#### leveled compaction (Cassandra)
* key range is split up into smaller SSTables and older data is moved into separate “levels”

