# redis

## Reliability


## Scalability
* Redis cluster: the official way to get automatic sharding and high availability. GA at 2015/04.
  * Redis cluster node needs 2 ports: client port (default: 6379) and cluster port (6479 + 10000)
    * cluster port is used by cluster nodes for node-to-node communication using a binary protocol (little bandwidth).
    * client port needs to be open to all clients + cluster nodes.
  * Cluster topology
    * Redis Cluster is a full mesh where every node is connected with every other node using a TCP connection.
    * nodes use a gossip protocol and a configuration update mechanism in order to avoid exchanging too many messages between nodes during normal conditions
  * Redirection
    * Check if the command is acceptable: single key or multiple keys to the same slot
    * Then check which node is responsible for the key
    * If the node itself handles the key, it is simply processed.
    * Otherwise, it would return a `MOVED` error, such as `MOVED 3999 127.0.0.1:6381`
    * Then, the client needs to reissue the command to the specified node.
  * Partition
    * Using `hash partitioning`: hash slots approach with pre-defined 16384 slots (16384/8=2048)
      * partition key = CRC16(key) % 16384
      * each node would take care of a range of hash slots, e.g, {A: 0-5500, B: 5501-11000, C: 11001-16383}
    * Moving slots between nodes and adding/removing nodes does not require downtime.
    * `Hash tags`: `this{foo}key` and `another{foo}key` are in the same hash slot.
    * Redis cluster master-slave model
    * Redis Cluster consistency guarantees
      * doesn't guarantee `strong consistency`
      * async replication -> data might be lost if master is crashed before it replicates data to its slave
      * client could use `WAIT` command to wait for synchronous replication
    * `Node timeout`: After node timeout has elapsed, a master node is considered to be failing, and can be 
      replaced by one of its replicas.
    * Each node keeps record of the owner of each slot
    * it would redirect client to the correct node if client sends a command with not-owned key.
    * Client side partitioning + Query routing
  * `Node ID`: This ID will be used forever by this specific instance in order for the instance to have a 
    unique name in the context of the cluster. Every node remembers every other node using this IDs, and not 
    by IP or port. IP addresses and ports may change, but the unique node identifier will never change for 
    all the life of the node.
  * A Redis cluster client shall keep the mapping between hash slot and nodes, to directly use the right 
    connection to the right node. The map is refreshed only when cluster configuration changed, such as failover 
    or adding or removing nodes.
  * Resharding
    * Redis cluster supports resharding without downtime.
    * `redis-cli --cluster reshard <host>:<port> --cluster-from <node-id> --cluster-to <node-id> --cluster-slots <number of slots> --cluster-yes`
  * Normally slave nodes will redirect clients to the authoritative master for the hash slot involved 
    in a given command, however clients can use slaves in order to scale reads using the READONLY command.
  * Failover
    * Use `DEBUG SEGFAULT` command to simulate a Redis node crash
    * Use `CLUSTER FAILOVER` command on a slave node to perform `manual failover`
      * Manual failovers are special and are safer compared to failovers resulting from actual master failures, since 
        they occur in a way that avoid data loss in the process.
        1. Basically clients connected to the master we are failing over are stopped.
        2. master sends its replication offset to the slave, that waits to reach the offset on its side
        3. When the replication offset is reached, the failover starts, and the old master is informed about the configuration switch.
        4. When the clients are unblocked on the old master, they are redirected to the new master.
      * To promote a replica to master, it must first be known as a replica by a majority of the masters in the cluster.
        It means we need to wait for a while if a new replica just added into a cluster to make sure the masters are 
        aware of this new replica.
  * Add a new node
    * Add a master node
      * `redis-cli --cluster add-node <new-host-ip>:<new-host-port> <any-node-ip>:<any-node-port>`
      * With the above command, a new *empty* master node is added. However, it holds no data and has no assigned hash slots.
      * Still need to perform `resharding` to move hash slots to this new master node.
    * Add a replica node
      * `redis-cli --cluster add-node <new-host-ip>:<new-host-port> <any-node-ip>:<any-node-port> --cluster-slave`  
        redis-cli will add the new node as replica of a random master among the masters with less replicas.
      * `redis-cli --cluster add-node <new-host-ip>:<new-host-port> <any-node-ip>:<any-node-port> --cluster-slave --cluster-master-id <master-node-id>`  
        redis-cli will add the new node as replica to the specified master node.
  * Remove a node
    * `redis-cli --cluster del-node <any-node-ip>:<any-node-port> <node-id>`  
      Both master or slave nodes could be removed with this command. But, **in order to remove a master node it must be empty**.
  * Node failure
  * Redis cluster doesn't need another layer, e.g, Redis Sentinel, to manage the cluster
  * Redis cluster doesn't support multiple keys operation
  * Fault detection
    * Rely on heartbeat packets: ping and pong
    * Nodes continuously exchange heartbeat packages. Every node would ping every other node for every `NODE_TIMEOUT / 2` seconds.
    * Heartbeat packet
      * Common header: NodeID, timestamp, node state(master/slave, PFAIL, FAIL, etc), hash slot bitmap, etc
      * Gossip section: 
        * This section has a few random nodes among the nodes known to the sender. 
        * NodeID, IP/port of the node, node state
        * Gossip section is useful for failure detection and node discovery.
    * A node flags another node A with the `PFAIL` flag when the node A is not reachable for more than NODE_TIMEOUT time.
    * A node flags another node A with the `FAIL` flag when the majority of masters signal node A is `PFAIL` or `FAIL`
  * Salve election
    * A slave starts an election when its master is in `FAIL` state and it doesn't disconnect from master over `NODE_TIMEOUT * cluster-slave-validity-factor`
    * As soon as a master is in `FAIL` state, a slave waits a delay before trying to get promoted.  
      `delay = 500 ms + random delay between 0 and 500 ms + SLAVE_RANK * 1000 ms`  
      SALVE_RANK means the up-to-date rank of the salve. The most up-to-date slave is at rank 0.
    * The slave would broadcast `FAILOVER_AUTH_REQUEST` to every master then wait `2 * NODE_TIME` for replies.
    * A master can no longer vote for another slave of the same master for a period of `NODE_TIMEOUT * 2`
    * Once a slave receives ACKs from the majority of masters, it wins the election.
  * Replica migration
    * Redis cluster would migrate slaves nodes to a master that has no slave coverage.
    * This dynamic master-slave layout would improve availability.
    * controlled by `cluster-migration-barrier` flag
  * Replication
    * asynchronous replication (default), but Redis client could use `WAIT` command to address synchronous replication.
    * When a master and a replica instances are well-connected, the master keeps the replica updated by sending a stream of commands to the replica.
    * When the link between the master and the replica breaks, the replica reconnects and attempts to proceed with a partial resynchronization.
    * When a partial resynchronization is not possible, the replica will ask for a full resynchronization.  
      The master would create a snapshot(RDB) and send it to the replica and continue sending the stream of commands as the dataset changes.
    * Replicas could be connected by other sub-replicas in a cascading-like structure.
    * Replication can be used both for scalability and availability. For example, slow O(N) operations could be offloaded to replicas.
    * Master and replicas would use individual `offset` value for incremental replication. However, if master has no enough backlog,
      then a full synchronization happens:
      * Master creates a background process to create a RDB dump. At the same time, master starts to buffer all write commands.
      * When RDB is done, master transfer it to the replica. The replica load the RDB.
      * The master will then stream all buffered WRITE commands to the replica.

## Redis Persistence
* 4 different persistent options: none, RDB, AOF, RDB+AOF
* RDB (Redis Databse):
  * RDB is very a compact single-file point-in-time of your Redis data.
  * single-file: backup friendly
  * DB allows faster restarts with big datasets compared to AOF.
  * Snapshotting the whole memory into a compressed, high-desnity file through a forked child process
  * In theory the child should use as much memory as the parent being a copy, but actually thanks to 
    the copy-on-write semantic implemented by most modern operating systems the parent and child process 
    will share the common memory pages. A page will be duplicated only when it changes in the child or in the parent.
    * `echo 1 > /proc/sys/vm/overcommit_memory`
  * By using RDB, you could still have data loss because it only did snapshotting periodically.
  * RDB used `fork()` which might take some milliseconds if the dataset is huge.
  * Use `SAVE` or `BGSAVE` command to create the RDB file
    * `SAVE`: blocking command; not recommend
    * `BGSAVE`: nonblocking command (still block when forking child process)
  * Steps:
    * Redis forks. We now have a child and a parent process.
    * The child starts to write the dataset to a temporary RDB file.
    * When the child is done writing the new RDB file, it replaces the old one.

* AOF (Append Only File):
  * AOF Redis is much more durable
    * fsync options: `no fsync`, `fsync every second` (default), `fsync every write`
      * `no fsync` means let OS make the fsync which typically happens every 30s in Linux.
  * The AOF log is an append only log which performs high performance on disk.
  * Just like write-ahead-logs in LSM tree, AOF log also needs rewriting to remove redundant logs.
    * `BGREWRITEAOF`
    * Redis could rewrite the AOF logs in background.
    * Log rewriting uses the same copy-on-write trick already in use for snapshotting.
    * Redis use `rename(2)` to switch the old and new AOF file after the compaction is finished. (`rename(2)` is atomic)
  * AOF files are usually bigger than the equivalent RDB files for the same dataset.
  * AOF can be slower than RDB depending on the exact fsync policy.
  * turn on AOF if `appendonly yes` in the config file
  * Steps:
    * Redis forks, so now we have a child and a parent process.
    * The child starts writing the new AOF in a temporary file.
    * The parent accumulates all the new changes in an in-memory buffer (but at the same time it writes the new changes 
      in the old append-only file, so if the rewriting fails, we are safe).
    * When the child is done rewriting the file, the parent gets a signal, and appends the in-memory buffer at the 
      end of the file generated by the child.
    * Now Redis atomically renames the old file into the new one, and starts appending new data into the new file.

* RDB+AOF:
  * In this case, Redis could use AOF to reconstruct the dataset becuase it is more completed.

* Redis >= 2.4, it would avoid run both AOF rewite and RDB snapshotting at the same time. This prevents 2 background tasks
  from doing heavy disk I/O at the same time.


## Others
* LRU cache
* RTT

## Reference
* https://redis.io/topics/faq
* https://redis.io/topics/persistence
* https://medium.com/fcamels-notes/redis-%E5%92%8C-redis-cluster-%E6%A6%82%E5%BF%B5%E7%AD%86%E8%A8%98-fdc19a3117f3
* https://mp.weixin.qq.com/s?__biz=MzA4MTc4NTUxNQ==&mid=2650520049&idx=1&sn=17a260f877b3ac4ff2b393981213ff94
* https://github.com/heibaiying/Full-Stack-Notes/blob/master/notes/Redis_%E9%9B%86%E7%BE%A4%E6%A8%A1%E5%BC%8F.md
* https://github.com/heibaiying/Full-Stack-Notes/blob/master/notes/Redis_%E6%8C%81%E4%B9%85%E5%8C%96.md