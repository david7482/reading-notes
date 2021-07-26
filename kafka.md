# Kafka
Kafka was born out of LinkedIn’s need to move a very large number of messages efficiently, amounting 
to multiple terabytes of data on an hourly basis. It is optimized for throughput at the 
expense of latency and jitter, but still preserve the following characteristics:

* durability, 
* strict record order
* at-least-once delivery

## Why Kafka is so fast [[link](https://medium.com/swlh/why-kafka-is-so-fast-bde0d987cd03)]

* Log-structured persistence
    * Kafka uses **append-only log** which makes it use **sequential I/O** for both read and write.
    * The append-only log persistence layer will keep up with network traffic. The bottleneck 
      with Kafka would be the network.
* Record batching
    * Batching of records amortizes the overhead of the network round-trip, using larger packets and 
      improving bandwidth efficiency.
* Batch compression
    * The compression ratio of JSON would be 5x - 7x.
    * Record batching would be a client-side operation. It has a positive effect not only on the 
      network bandwidth but also on the brokers’ disk I/O utilization.
* Cheap consumers
    * Kafka doesn't remove messages once it has been consumed.
    * It tracks offsets at each consumer group level.
    * The message would be removed in the background (compaction operation) to only retain the last 
      known offsets for any given consumer group.
    * Only providers and internal Kafka processes could modify log files.
* Unflushed buffered writes
    * Kafka doesn’t actually call **fsync** when writing to the disk before acknowledging the write 
      operation.
    * The only requirement for an ACK is that the record has been written to the I/O buffer.
    * This makes Kafka to be _a disk-backed in-memory queue_.
    * But, this type of writing is unsafe. What makes Kafka durable is running several in-sync replicas; 
      even if one were to fail, the others will remain operational.
* Client-side optimisations
    * Kafka offloads a significant amount of work to clients.
    * This includes the batching of records in an accumulator, hashing the record keys to arrive at the 
      correct partition index, checksumming the records and the compression of the record batch.
    * This lets the client make low-level forwarding decisions; rather than sending a record blindly 
      to the cluster and relying on the latter to forward it to the appropriate broker node.
* Zero-copy
    * Kafka uses the `transferTo()` method of `java.nio.channels.FileChannel` to achieve zero-copy operation.
    * It doesn't involve kernel-space to user-space buffer copy and even reuses the kernel buffer between
      read buffer and socket buffer.
* Avoiding the GC
    * By heavily usage channels, buffers and page cache, Kafka could reduce the loading from GC.
    * The difference in throughput is minimal, but the real gain is to reduce _jitter_.
    * By avoiding the GC, the brokers are less likely to experience a pause that may impact the client.



