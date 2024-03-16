## Storage and Retrieval

Two families of storage engines that are used in sql and nosql dbs are Log-structured storage engines and page-oriented storage engines.

# Log-Structured Storage Engines
In log-structed storage, the records are stored in log, which is a append only data file. Each line denotes an individual record. Overriding any existing record does not actually override it, but appends a new log in the end. we always should refer to the latest instance to get the updated version of a record. As writes are just append operations, it is very efficient. But read operation is inefficient as it has to scan the entire db file, taking O(n) complexity. For improving read performance, we introduce index, which is an additional structure that is derived from the primary data and acts as a signpost to help us locate the data. Indexing adds overhead on writes, because the index also needs to be updated everytime data is written. It is an trade off: well-chosen indexes speed up reads queries, but slows down writes.
**Hash Indexes**
Simple indexing strategy which keeps an in-memory hash map where every key is mapped to a byte offset in the data file. Whenever we append a new key-value pair to the file, we also update the hasp map to reflect the offset of the data. When we want to look up a value, use the hash map to find the offset in the data file, seek to that location, and read the value. Bitcask, the storage engine of Riak uses this index with a requirement that all the keys fit in the RAM. Such storage engines are well suited for situations where we have not many distinct keys and have a large number of writes per key.
As we only append on writes, to avoid running out of disk space, we break the log into segments of certain size by closing a segment file when it reaches a certain size and making subsequent writes to a new segement file. We then perform compaction on the closed segement, which is the process of throwing away duplicate keys in the log, and keeping only the most recently updated one for each key.As compaction results in smaller segments, we can also merge the segments during compaction. The merging and compaction of frozen segments can be done in background thread to form a new segment which will be used hereforth.
Each segement how has its own in-memory hash table, mapping keys to offset. To find a key, we first check the hash table of most recent segment, if the key is not present we check the second-most recent segement's hash table and so on.
Limitations of hash table indexes are,
- the hash table must fit in memory 
- range queries are not efficient.

# SSTables and LSM-Trees
Sorted String Table are structures in which the sequence of records are sorted by some key. To make a log-structured segment a SST, we need to sequence the records in the log in sorted order of key and ensure that each key only appears once within each merged segment file. The second requirement is taken care by compaction process. For first requirement, during the merging process, we read the input segments side by side, look at the first key in each segment, copy the lowest key to the output segment. Now we do not have to keep an index with all the keys in memory. We can have indexes for interleaved records and figure out that a record will be within any two indexes based on the value of its key.
Creating and maintaining of SSTable works as follows, 
    - when a write comes in, add it to an in-memory balanced tree data structure(for example a red-black tree or AVL tree). This in-memory tree is sometimes called memtable.
    - when a memtable gets bigger than some threshold - typically a few mb - write it out to the disk as an SSTable file. This is efficient as the tree already maintains the key-value pair sorted by key. This new SSTable file becomes the most recent segment of the database. While the SSTable is being written out to disk, writes can continue to a new memtable instance. 
    - to serve read requests, first try to find the key in the memtable, then in the most recent on-disk segement, then the next-older segment, etc.,.
    - run merging and compaction in the background to combine segment files and to discard overwritten or deleted values.
One problem with this approach is if the db crashes, the most recent writes(which are in memtable but not yet written out to disk) are lost. To avoid it, we keep a separate log(just a normal log file) on disk to which every write is immediately appended. On crashes, we use this to restore the memtable or delete it once the memtable is written out to the disk.
This algorithm is used in levelDB and rockDB storage engines. Cassandra and HBase also use similar storage engines.
This indexing structure is called Log-Structured Merge Tree(LSM-Tree). Storage engines based on this principle of compaction and merging sorted files are called LSM storage engines.
Lucene, an indexing engine for full-text search used by elasticsearch and solr, uses a similar method for storing its term dictionary.This is implemented as a key-value structure where the key is a word and the value is the list of IDs of all the documents that contains the word. In lucene, this mapping from word to postings list is kept in SSTable-like sorted files.
A LSM-tree algorithm can be slow when searching for a non-existent key. To optimize this we use Bloom filters, which is a memory-efficient data structure for appraximating whether a key does not appear in the db.
Different strategies to determine the order and timing of how SSTables are compacted and merged are size-tiered and leveled compaction,
- size-tiered compaction: newer and smaller SSTables are successively merged into older and larger SSTables.
- levelled compaction: the key range is split up into smaller SSTables and older data is moved into separate 'levels', which allows the compaction to proceed more incrementally and use less disk space.

# B-Tree
B-Tree is the widely used indexing structure used in all relational dbs. Like SSTable, B-tree keeps key-value pairs sorted by key, allowing efficient key lookup and range queries. The log-structured indexes break the db down into variable size segments and write a segment sequentially. By contrast, B-tree break the database down into fixed-size blocks or pages, traditionally 4KB, and read or write one page at a time. One page is designated as the root of the B-tree. The root contains several keys and references to child pages. Each child is reponsible for a continuous range of keys, and the keys between the references indicate where the boundaries between those ranges lie. The branching ends in a leaf page, which either contains the value of the key inline or a reference to the page containing the value. The number of references to child pages in one page of the B-tree is called the branching factor.
When adding new key, if there isn't enough space in the page to accommodate, then the page is split into two half-full pages, and the parent page is updated to account for the new subdivision of key ranges. This ensures that the tree remains balanced: a Btree with n keys always has a depth of O(log n) 
The basic underlying write operation of a B-tree is to overwrite a page on the disk with new data. All the reference to that page remains intact when the page is overwritten. But in log-structured indexes like LSM-tree, we only append to files and never modify files in-place.
To prevent from corrupted index, B-tree implements an additional data structure on disk called write-ahead-log, an append only file to which every B-tree modification must be written before it can be applied to the page of the tree itself. 
Additional complication in updating pages is the careful concurrency control, if multiple threads are going to access the B-tree at the same time.
LSM trees are typically faster for writes, whereas B-trees are thought to be faster for reads. 
*LSM Tree Advantages:*
- log structured merge trees are typically able to sustain higher write throughput than B-trees, partly because they sometimes have lower write amplification(the effect of one write to db resulting in multiple writes to the disk), and partly because they sequentially write compact SSTable files rather than having to overwrite several pages in the tree. This difference is particularly important on magnetic hard drives, where sequential writes are much faster than random writes.
- LSM-Trees can be compressed better and thus often produce smaller files on disk than B-tree.
*LSM Tree Disadvantages:*
- the compaction process can sometime interfere with the performance of the ongoing reads and writes. 

# Other Indexing Structure:
The key-value indexes are like a primary key which uniquely identifies one record. Its common to have secondary indexes which are constructed from key-value indexes but the keys are not unique, i.e., there might be many records with the same key. This can be solved either by making each value in the index a list of matching row identifiers or by making each key unique by appending a row identifier to it.

# Storing value within the index:
The key in an index is the thing that queries search for, but the value can be, the actual row or a reference to the row stored elsewhere. In latter case, the place where rows are stored is known as heap file. The heap file approach is common as it avoids duplicating data when multiple secondary indexes are present: each index just references a location in the heap file and the actual data is kept in one place. This approach will have complexity when updating if the value has to be moved to new heap location and all the referring indexes needs to be updated.
In some situations, the extra hop from index to heap file causes performance penalty for reads, so the row is stored within an index. This is know as clustered index. Eg., in mysql innoDB storage engine, the primary key of a table is always a clustered index, and the secondary indexes refer to the primary key.
A compromise between clustered and non-clustered index is known as a covering index or index with included columns, which stores some of the table's columns within the index.

# Multi-column index:
It is used when we need to query based on multiple fields simultaneously. Common type of multi-column index is concatenated index, in which all fields are combined in the order specified in index definition to form the key. For multi-dimensional index requirements, eg. querying by latitute and longitude, B-tree and LSM-tree will not be efficient.

# Full-text search and fuzzy indexes:
It is used in cases where we need to query for similar keys instead of exact match.

# keeping data in memory:
Some in-memory dbs are intended for caching use only, where its acceptable for data to be lost if the machine restarts. But few in-memory dbs aims for durability by special hardware(battery-backed RAM), writing a log of changes to disk, writing periodic snapshots to disk or replicating to other machine. 
RAMCloud is an open-source, in-memory key-value store with durability(using a log-structured approach for the data in memory as well as the data in disk). 
Redis and couchbase provide weak durability by writing to disk asynchronously.
The performance advantage of in-memory db is not the fact that they don't have to read from disk, even a disk-based storage engine can provide the same by caching the recently used disk blocks in memory.It is faster because if avoids the overhead of encoding in-memory data structure in a form that can be written to disk.
In-memory database architecture could be extended to support datasets larger than the available memory, without bringing back the overhead of a disk-centric architecture. The so-called anti-caching approach works by evicting the least recently used data from memory to disk when there is not enough memory, and loading it back into memory when it is accessed again in the future.

# Transaction processing vs Analytics:
Transactions refers to a group of reads and writes that form a logical unit.
Online transaction processing(OLTP) refers to access pattern which typically looks up a small number of records by some key, using an index and records are inserted or updated based on the user's input interactively.
Data Analytics involves reading a huge number of records, only reading a few columns per record, and calculates aggregate statistics(such as count, sum, or average) rather than returning the raw data to the user. This access pattern is call Online analytic processing(OLAP).
Databases used for OLAP is called data warehouse.
Data is extracted from the OLTP database with periodic dumps or continued streams, transformed into an anaylsis friendly schema, cleaned up and then loaded into the data warehouse. This process of getting data into the data warehouse is known as extract-transform-load(ETL)

**Star and Snowflake: schemas for analytics:**
Many data warehouses use a data-model called star schema( also known as dimensional modeling)
At the center of the star schema is the fact table. Each row in the fact table represents an event that occured at a particular time.
Some columns in the fact table are attributes. Others are foreign key references to other tables, called dimensional tables. Each row in the fact table represents an event, the dimensions represent the who, what, where, when, how and why of the event.
Snowflake schema is a variation of star schema, where the dimensional table are further broken down into subdimensions.

# Column Oriented Storage:
Analytic query usually doesn't require all the columns at one time.
In most OLTP dbs, storage is laid out in a row-oriented fashion, all the values from one row of a table are stored next to each other. Document dbs also store an entire document as one contiguous sequence of bytes. With proper indexing, the storage engine can find where to get the particular row, but it still has to load all of those rows with all the attributes from disk into memory, parse them and filter out the unrequired attributes.
In column-oriented storage, instead of storing all the values from one row together, all the values from each column is stored together. If each column is stored in a separate file, a query only needs to read and parse those columns that are used in that query.
Column-oriented storage layout relies on each column file containing the rows in the same order.

# Column Compression:
Bitmap compression is a compression technique used in data wharehouses. We take a column with n distinct values and turn it into n seperate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if that row has that value, and 0 if not. 
If n is small, say 200 distinct values, those bitmaps can be stored with one bit per row. If n is bigger, there will be lot of zeros in most of the bitmaps, which is sparse, then the bitmaps are run-length encoded. (4-0, 5-1, 2-0, ...)
Cassandra and Hbase have column families, within which, they store all columsn from a row together, along with a row key, and they do not use column compression. Bigtable model is still mostly row oriented. 
Column-oriented storage also use vectorized processing, a technique where operations are performed on entire vector or array of data at once, rather than processing individual elements sequentially. This approach leverages SIMD(Single Instruction Multiple Data) operation capability of modern CPU to perform computations in parallel, leading to significant performance improvement.

# Sort Order in column storage:
The data needs to be sorted an entire row at a time, even though it is sorted by column. A second column can determine the sort order of any rows that have the same value in the first column.
Data warehouse vertica, extends this idea and stores the redundant data stored in different machines using sort order of a different column.

# Writing to Column-oriented storage:
Update in-place, like B-tree is not possible with compressed columns. If we want to insert a row in the middle of a sorted table, we should most likely have to rewrite all the column files. As rows are identified by their position within a column, the insertion has to update all columns respectively. 
Hence we use LSM-trees. All writes first go to an in-memory store, where they are added to a sorted structure and prepared for writing to disk. It doesn't matter whether the in-memory store is row-oriented or column-oriented. When enough writes are accumulated, they are merged with column files in disk and written to new files in bulk.

# Aggregation: Data cubes and materialized view:
Materialized view is caching the most frequently queried aggregate values. It is similar to views in sql, except that views are just a shortcut but materialized views are actual copies. 
When the underlying data changes, a materialized view must be updated. Such updates will make writes more expensive, hence materialized views are not often used in OLTP. In read heavy OLAP it makes sense. 
Data cubes are a grid of aggregates grouped by different dimensions.

Disk seek time is often the bottleneck with OLTP systems. Disk bandwidht is often the bottlenect with OLAP systems.
