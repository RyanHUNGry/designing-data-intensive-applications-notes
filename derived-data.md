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
- 