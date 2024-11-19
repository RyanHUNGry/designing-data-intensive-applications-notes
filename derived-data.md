# Chapter 10: Batch Processing
- Unix provide simple tools for inputting and outputting data via pipes; a Unix process takes in some input and produces some output, but Unix standardizes the output formats so different processes can accept/transmit data
- Unix pipes provide a simple, easy way for inter-process communication
- MapReduce is a programming model rather than a usable system
- MapReduce is similar to Unix tools for data processing, but just distributed across nodes
- A MapReduce job is similar to a single Unix process, but parallelized over multiple nodes
- Unix uses stdin and stdout as input and output, whereas MapReduce jobs read and write files on a distributed filesystem
- Hadoop, a library implementation of MapReduce, implements MapReduce by using a distributed filesystem called HDFS as storage, and MapReduce jobs interact with this storage
- Hadoop replicates data across nodes for fault-tolerance
- MapReduce contains two steps
    1. The mapper is called once for every input record, and its job is to extract a key-value pair
    2. The reducer takes the values shared by same key, and iterates over them to produce a value
    3. Between the mapper and reducer is an implicit step that sorts keys from the mapper before sending to reducer
        i. The sorting allows easy grouping of data to pass to reducer
- MapReduce systems provide APIs to implement MapReduce logic in conventional programming languages
- Hadoop MapReduce has a multi-step distributed execution flow
    1. A Hadoop MapReduce job takes in a file directory, and partitions each file or file block as its own map task to be executed on separate nodes
    2. The map tasks are typically ran on the same node that has the replicated file or file block to be processed, given there is enough RAM and CPU to fit data
    3. Application code for mapping and reducing is not on the actual Hadoop system, so the framework copies the code to the appropriate machines
    4. The output of the mapper are key-value pairs, and they are partitioned based on hash of key and written to sorted file on the mapper's local disk
        i. This means for each key per map task, there exists a sorted file on that mapper's disk per key 
    5. Reducers connect to each of the mappers and download the files of partitioned, sorted key-value pairs
    6. The reducers then take the files from the mappers, which are grouped by key, and merges them together, preserving sort order
    7. The reducer is called with a key and an iterator that incrementally scans over records with the same key, and generate an output file on HDFS per reducer task with replicas on other nodes
- MapReduce is limited as a single job, so multiple jobs are chained where one job reads the files outputted by a previous job
    1. Various workflow schedulers scuh as Airflow and Pig allow managing multiple workflows efficiently
- An example of reduce-side joins is correlating user activity with user profile information, with user activity located in a log file and user profile located in a database
    1. It is too pricy to loop through each activity record and query the database, so the database is copied and extracted as files to the same HDFS cluster hosting user activity files
    2. Now, a MapReduce job can be used to join the two sets of data together for batch processing
    3. For the two sets of data, two mapper types are used (parallelized independently), and the reducer reads the key-value partition files from both sets of mappers
    4. The reducer now has all the values corresponding to key from both datasets, and join logic is now complete and aggregation/processing can be implemented
- A groupby is grouping records with the same key, and then performing some aggregation on the values within each group
    1. Simply make the group key the key in the key-value pair produced by mapper
    2. The reducers will then aggregate the values corresponding to the key
- Sometimes a key can be skewed, which means it occurs more often than other keys and can bottleneck the system with an overloaded reducer
    1. Certain higher-level MapReduce implementation libraries such as Pig, Hive, and Crunch handle skew by sampling which keys are hot, and then load balancing them across multiple reducers; a second MapReduce job is required to aggregate the first-stager reducer values into a single value per key
- Reduce-side joins perform actual join logic in reducers, and the mappers only take the key-value extraction + sorting role
- Reduce-side joins do not need to make assumptions about input, because the mappers prepare the data to be ready for joining, even if they are from heterogenous sources or formats
- However, reduce-side joins are slower due to more sorting and data copying
- Map-side joins requires making assumptions about input data, and ituses a MapReduce job with no reducers or sorting
    1. Broadcast hash joins assume joining a larger dataset with a smaller dataset by loading the smaller dataset into an in-memory hashmap for each mapper, and then partitioning the large dataset across mappers for joining
    2. Partitioned hash joins assume also partition the smaller dataset based on a partition key for the larger dataset, so a single mapper gets all records from both datasets sharing the same hashed key
- MapReduce was originally designed for making search indexes, where documents are processed and index files are generated via MapReduce jobs, allowing full-text search over a set of documents
- MapReduce outputs can be loaded into a database by using key-value stores such as Voldemort within the MapReduce job itself, and then copying the immutable output files to the database after the job is finished
- Hadoop MapReduce share similarities and differences with distributed databases
    1. Hadoop MapReduce provides a general "operating system" (or framework) that can run arbitrary programs whereas MPP databases focus on parallel SQL query execution for analytics
    2. Databases requires data to adhere to schemas, whereas files in HDFS are just byte sequences and can be any data model or encoding
        i. The mapper is what cleans this data up into a standardized form
    3. Hadoop is commonly used for ETL processes
    4. MPP databases only support query, so task-specific logic such as building search indexes or building machine learning systems cannot be expressed with SQL and must be coded up using MapReduce
    5. MapReduce tolerates faults by rerunning a map or reduce task at the granularity of an individual task
- MapReduce is simple to understand, but difficult to use because common functionality such as joins need to be implemented manually
- MapReduce jobs are independent, and they require the full output file of one job as input for another; this requires materializing the files rather than using an in-memory buffer to stream data between processes in Unix
- Some output files are only used by a single job, making output replication overkill for this intermediate state

# Chapter 11: Stream Processing
- A stream refers to data incrementally made available over time, such as in stdin and stdout of Unix
- In stream processing context, a record is more commonly known as an event, and usually contains a timestamp of occurrence
- An event is generated once by a producer (publisher/sender) and then potentially processed by multiple consumers (subscribers or recipients)
- Events are grouped into a topic or a stream rather than as a file in a filesystem in the context of batch processing
- In principle, a file or database is sufficient for producer and consumers, but they are not optimized for continuous polling by the consumer
- There are methods for direct streaming of events from producers to consumers, which means there is no intermediary node used in communication
    1. When a consumer dies, the consumer does not see data that was sent by the producer during the time it is down
    2. If the consumer is overloaded, the system can drop new messages from the producer, causing potential data loss
- A message broker is a database specialized for message streams, allowing producers and consumers to connect to it as clients, improving fault-tolerance of the system
    1. A message broker typically implements queues and provides additional features such as routing and pub/subscribers
    2. A message queue system typically has less features, but many people conflate the two terms 
- Consumers are generally asynchronous due to the queuing behavior of the broker, and producers only waits for broker to confirm it has buffered the message rather than waiting for consumer success
- Although message brokers store data, and some persist data for a while, they still differ from traditional databases; the message broker systems here are AMPQ-style and JMS-style message brokers
    1. Message brokers automatically delete a message when it has been delivered to its consumers (or it has an eventual garbage collection process)
    2. Message brokers do not support index search optimizations like databases
    3. Message brokers do not support queries, they only provide the events in order of generation due to the queue structure
- When there are multiple consumers reading from the same topic, an event can either be load balanced to a single consumer, or fan-out delivered to all consumers
- A client must explicitly tell the broker when it has finished processing a message for the broker to remove it from the queue, making the system fault-tolerant
- Unlike AMQP-style and JMS-style message brokers, log-based message brokers combine durable storage of databases with low-latency notification facilities of messaging
    1. Remember a log is an append-only sequence of records on disk
    2. Logs can be partitioned across nodes, so each partition is a separate log
    3. Topics group partitions that carry messages of the same type
    4. Messages per partition are monotonically ordered by ID, so they are totally ordered per partition, but there is no ordering gaurantee across partitions
    5. Kafka is the prime log-based message broker example
    6. Log-based brokers support fan-out messaging easily because consumers can indepnedently read logs without affecting each other; reading an event does not delete it from log
    7. For load balancing, partitions are assigned to nodes in a consumer group, so each consumer within the group consumers all messages in the partitions it has been assigned
        - i. Number of nodes in a consumer group consuming a topic can be at most the number of log partitions in that topic, because messages within the same partition are delivered to the same node
        - ii. If a single message is slow to process, it will hold up subsequent message processing in that partition
    8. The message broker behaves like a leader database, and the consumer like a follower in the context of consumer offset being similar to log sequence number of single-leader database replication
        - i. Log sequence number allows disconnected follower nodes to replicate without missing any writes
        - ii. Consumer starts consuming messages with an offset greater than the consumer offset, and this allows other nodes to pickup a down consumer and prevents messages from being skipped
    9. A log can get full, so old segments will need to be eventually deleted or extracted to archive storage; however, deployments rarely use full write bandwidth of the disk, so the log can typically keep a buffer of days' or weeks' of data
    10. Unlike AMPQ-style and JMS-style brokers, a consumer offset can be manipulated by a user to look at older messages
    11. Log compaction throws away duplicate messages by key, and keeps only the most recent update for each key
- Databases also take inspiration from stream processing
    1. A replication log is technically a stream of database write events, produced by the leader as it processes transactions
- A common problem where databases take inspiration from message systems is syncing heterogeneous data systems
    1. For instance, an OLTP database for an application could be supported by a search index, a cache, and a data warehouse; syncing these auxilliary systems manually with code logic is difficult due to race conditions
    2. CDC is the process of exposing data changes written to a database and streaming them to a log in which they can be replicated to other systems
    3. In a sense, the database is a producer of events (data changes), and auxilliary systems such as search indexes are consumers of those events
    4. Examples of CDC exposing a database's log include Mongoriver, Debezium, and Bottled Water for MongoDB, MySQL, and PostgreSQL respectively
    5. Other databases are starting to support change streams as a first-class interface
- Event sourcing


