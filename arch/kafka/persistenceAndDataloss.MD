# kafka's persistence And dataloss guarantee


## persistence
Kafka persist data in disk, not in memories, and in consideration of performance, kafka uses OperatingSystem's pagecache rather than direct disk io. Kafka flush pagecache to disk periodically.


### reason why Kafka not use in memory data but disk persistence:
* using pagecache could be fast enough;
* JVM memory's cost is very high, as it use much more extra data to reprensent a data struct;
* JVM's garbage collection is slow when heap grow huge;



### notes:
* Because uses of pagecache and flush data to disk periodically rather than flush immediatly as event come up, the data cannot be gurantee and may be lost in a single instance Kafka.In a single instance, when an event comes, kafka simply writes it to the pagecache, and leave the flush work later, if an crash occur before the "flush to work", than this event is lost.



## Dataloss guarantee
Kafka provide guarantee of data by using the replication.

Specifically, when an event cameup, Kafka writes it to some number of the instances in clusters, before response to the producer with success. We can control this behavior by specify the "acks" to "all" in the producer's request, and config the topic's "min.insync.replicas" to an appropriate number.

note that in Kafka, the leader of a partition(or major replica of a partition), is constitued as the replication factor. That means the if the replicas "1" indicate an un-replicated implimentation.
