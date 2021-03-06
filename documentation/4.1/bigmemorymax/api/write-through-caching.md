---
---
# Write-through and Write-behind Caching with the CacheWriter {#Write-through-and-Write-behind-Caching-with-the-CacheWriter}

{toc|2:3}

## Introduction
Write-through caching is a caching pattern where writes to the cache cause writes to an underlying resource. The cache acts as a facade
to the underlying resource. With this pattern, it often makes sense to read through the cache too.
Write-behind caching uses the same client API; however, the write happens asynchronously.


While file systems or a web-service clients can underlie the facade of a write-through cache, the most common underlying resource is a database.
To simplify the discussion, we will use the database as the example resource.

## Potential Benefits of Write-Behind
The major benefit of write-behind is database offload. This can be achieved in a number of ways:

* time shifting - moving writes to a specific time or time interval. For example, writes could be batched up and written overnight, or at 5 minutes past the hour, to avoid periods of peak contention.
* rate limiting - spreading writes out to flatten peaks. Say a Point of Sale network has an end-of-day procedure where data gets written up to a central server. All POS nodes in the same time zone will write all at once. A very large peak will occur. Using rate
limiting, writes could be limited to 100 TPS, and the queue of writes are whittled down over several hours
* conflation - consolidate writes to create fewer transactions. For example, a value in a database row is updated by 5 writes, incrementing it from 10 to 20 to 31 to 40 to 45. Using conflation, the 5 transactions are replaced by one to update the value from 10 to 45.

These benefits must be weighed against the limitations and constraints imposed.

## Limitations & Constraints of Write-Behind

### Transaction Boundaries
If the cache participates in a JTA transaction, which means it is an XAResource, then the cache can be made consistent with the
  database. A write to the database, and a commit or rollback, happens with the transaction boundary.
In write-behind, the write to the resource happens after the write to the cache. The transaction boundary is the write to the outstanding queue, not
the write behind. In write-through mode, commit can get called and both the cache and the underlying resource can get
committed at once.
Because the database is being written to outside of the transaction, there is always a risk that a failure on the eventual write
will occur. While this can be mitigated with retry counts and delays, compensating actions may be required.

### Time delay
The obvious implication of asynchronous writes is that there is a delay between when the cache is updated and when the database
is updated. This introduces an inconsistency between the cache and the database, where the cache holds the correct value and the
database will be eventually consistent with the cache. The data passed into the CacheWriter methods is a snapshot of the cache
entry at the time of the write to operation.
A read against the database will result in incorrect data being loaded.

### Applications Tolerant of Inconsistency
The application must be tolerant of inconsistent data. The following examples illustrate this requirement:

* The database is logging transactions and only appends are done.
* Reading is done by a part of the application that does not write, so there is no way that data can be corrupted. The application is tolerant of delays. For example, a news application where the reader displays the articles that are written.

Note if other applications are writing to the database, then a cache can often be inconsistent with the database.

### Node time synchronisation
Ideally node times should be synchronised. The write-behind queue is generally written to the underlying resource in timestamp order, based
on the timestamp of the cache operation, although there is no guaranteed ordering.
The ordering will be more consistent if all nodes are using the same time. This can easily be achieved by configuring
your system clock to synchronise with a time authority using Network Time Protocol.

### No ordering guarantees
The items on the write-behind queue are generally in order, but this isn't guaranteed. In certain situations and more particularly in
clustered usage, the items can be processed out of order. Additionally, when batching is used, write and delete collections are
aggregated separately and can be processed inside the CacheWriter in a different order than the order that was used by the queue.
Your application must be tolerant of item reordering or you need to compensate for this in your implementation of the
CacheWriter. Possible examples are:

* Working with versioning in the cache elements.

    You may have to explicitly version elements. Auto-versioning is off by default and is effective only for unclustered MemoryStore caches. Distributed caches or caches that use off-heap or disk stores cannot use auto-versioning. To enable auto-versioning, set the system property `net.sf.ehcache.element.version.auto` (it is false by default). Note that if this property is turned on for one of the ineligible caches, auto-versioning will silently fail.

* Verifications with the underlying resource to check if the scheduled write-behind operation is still relevant.

## Using a combined Read-Through and Write-Behind Cache
For applications that are not tolerant of inconsistency, the simplest solution is for the application to always read through
the same cache that it writes through. Provided all database writes are through the cache, consistency is guaranteed. And in the
distributed caching scenario, using Terracotta clustering extends the same guarantee to the cluster.
If using transactions, the cache is the XAResource, and a commit is a commit to the cache.
The cache effectively becomes the System Of Record ("SOR"). Terracotta clustering provides
HA and durability and can easily act as the SOR. The database then becomes a backup to the SOR.
The following aspects of read-through with write-behind should be considered:

### Lazy Loading
The entire data set does not need to be loaded into the cache on startup. A read-through cache
uses a `CacheLoader` that loads data into the cache on demand. In this way the cache can be populated lazily.

### Caching of a Partial Dataset
If the entire dataset cannot fit in the cache, then some reads will miss the cache and fall through to the `CacheLoader` which will
  in turn hit the database. If a write has occurred but has not yet hit the database due to write-behind, then the database will be inconsistent.
The simplest solution is to ensure that the entire dataset is in the cache. This then places some implications on cache configuration
  in the areas of expiry and eviction.

#### Eviction
Eviction or flushing of elements, occurs when the maximum elements for the cache have been exceeded. Be sure to size the cache appropriately to avoid eviction or flushing. See [How to Size Caches](/documentation/4.1/configuration/cache-size) for more information.

#### Expiry
Even if all of the dataset can fit in the cache, it could be evicted if Elements expire. Accordingly, both `timeToLive` and
  `timeToIdle` should be set to eternal ("0") to prevent this from happening.

## Introductory Video
Alex Snaps the primary author of Write Behind presents an [introductory video](http://vimeo.com/21193026) on Write Behind.

## Sample Application
We have created a sample web application for a raffle which fully demonstrates how to use write behind.
You can also [checkout](https://github.com/alexsnaps/Ehcache-Raffle) the Ehcache Raffle application, that demonstrates Cache Writers
  and Cache Loaders from github.com.


## Configuration
There are many configuration options. See the `CacheWriterConfiguration` for properties that may be set and their effect.
Below is an example of how to configure the cache writer in XML:

    <cache name="writeThroughCache1" ... >
    <cacheWriter writeMode="write-behind" maxWriteDelay="8" rateLimitPerSecond="5"
            writeCoalescing="true" writeBatching="true" writeBatchSize="20"
            retryAttempts="2" retryAttemptDelaySeconds="2">
       <cacheWriterFactory class="com.company.MyCacheWriterFactory"
                       properties="just.some.property=test; another.property=test2"
		       propertySeparator=";"/>
    </cacheWriter>
    </cache>

Further examples:

    <cache name="writeThroughCache2" ... >
     <cacheWriter/>
    </cache>
    <cache name="writeThroughCache3" ... >
     <cacheWriter writeMode="write-through" notifyListenersOnException="true" maxWriteDelay="30"
          rateLimitPerSecond="10" writeCoalescing="true" writeBatching="true" writeBatchSize="8"
          retryAttempts="20" retryAttemptDelaySeconds="60"/>
    </cache>
    <cache name="writeThroughCache4" ... >
     <cacheWriter writeMode="write-through" notifyListenersOnException="false" maxWriteDelay="0"
          rateLimitPerSecond="0" writeCoalescing="false" writeBatching="false" writeBatchSize="1"
          retryAttempts="0" retryAttemptDelaySeconds="0">
     <cacheWriterFactory class="net.sf.ehcache.writer.WriteThroughTestCacheWriterFactory"/>
     </cacheWriter>
    </cache>
    <cache name="writeBehindCache5" ... >
     <cacheWriter writeMode="write-behind" notifyListenersOnException="true" maxWriteDelay="8"
          rateLimitPerSecond="5" writeCoalescing="true" writeBatching="false" writeBatchSize="20"
          retryAttempts="2" retryAttemptDelaySeconds="2">
     <cacheWriterFactory class="net.sf.ehcache.writer.WriteThroughTestCacheWriterFactory"
                     properties="just.some.property=test; another.property=test2"
		     propertySeparator=";"/>
     </cacheWriter>
    </cache>

This configuration can also be achieved through the `Cache` constructor in Java:

    Cache cache = new Cache(
    new CacheConfiguration("cacheName", 10)
    .cacheWriter(new CacheWriterConfiguration()
    .writeMode(CacheWriterConfiguration.WriteMode.WRITE-BEHIND)
    .maxWriteDelay(8)
    .rateLimitPerSecond(5)
    .writeCoalescing(true)
    .writeBatching(true)
    .writeBatchSize(20)
    .retryAttempts(2)
    .retryAttemptDelaySeconds(2)
    .cacheWriterFactory(new CacheWriterConfiguration.CacheWriterFactoryConfiguration()
       .className("com.company.MyCacheWriterFactory")
       .properties("just.some.property=test; another.property=test2")
       .propertySeparator(";"))));

Instead of relying on a `CacheWriterFactoryConfiguration` to create a `CacheWriter`, it's also possible to explicitly
register a `CacheWriter` instance from within Java code. This allows you to refer to local resources like database
connections or file handles.

    Cache cache = manager.getCache("cacheName");
    MyCacheWriter writer = new MyCacheWriter(jdbcConnection);
    cache.registerCacheWriter(writer);

### Configuration Attributes
The CacheWriterFactory supports the following attributes:

#### All modes

*  write-mode \[write-through | write-behind] - Whether to run in write-behind or write-through mode. The default is write-through.

#### write-through mode only

*  notifyListenersOnException - Whether to notify listeners when an exception occurs on a store operation. Defaults to false. If using cache replication, set this attribute to "true" to ensure that changes to the underlying store are replicated.

#### write-behind mode only

* writeBehindMaxQueueSize - The maximum number of elements allowed per queue, or per bucket (if the queue has multiple buckets). "0" means unbounded (default). When an attempt to add an element is made, the queue size (or bucket size) is checked, and if full then the operation is blocked until the size drops by one. Note that elements or a batch currently being processed (and coalesced elements) are not included in the size value. Programmatically, this attribute can be set with:

`net.sf.ehcache.config.CacheWriterConfiguration.setWriteBehindMaxQueueSize()`

* writeBehindConcurrency - The number of thread-bucket pairs on the node for the given cache (default is 1). Each thread uses the settings configured for write-behind. For example, if rateLimitPerSecond is set to 100, each thread-bucket pair will perform up to 100 operations per second. In this case, setting writeBehindConcurrency="4" means that up to 400 operations per second will occur on the node for the given cache. Programmatically, this attribute can be set with:

`net.sf.ehcache.config.CacheWriterConfiguration.setWriteBehindConcurrency()`

*  maxWriteDelaySeconds - The maximum number of seconds to wait before writing behind. Defaults to 0. If set to a value greater than 0, it permits operations to build up in the queue to enable effective coalescing and batching optimisations.
*  rateLimitPerSecond - The maximum number of store operations to allow per second.
*  writeCoalescing - Whether to use write coalescing. Defaults to false. When set to true, if multiple operations on the same key are present in the write-behind queue, then only the latest write is done (the others are redundant). This can dramatically reduce load on the underlying resource.
*  writeBatching - Whether to batch write operations. Defaults to false. If set to true, storeAll and deleteAll will be called rather than store and delete being called for each key. Resources such as databases can perform more efficiently if updates are batched to reduce load.
*  writeBatchSize - The number of operations to include in each batch. Defaults to 1. If there are less entries in the write-behind queue than the batch size, the queue length size is used. Note that batching is split across operations. For example, if the batch size is 10 and there were 5 puts and 5 deletes, the CacheWriter is invoked. It does not wait for 10 puts or 10 deletes.
*  retryAttempts - The number of times to attempt writing from the queue. Defaults to 1.
*  retryAttemptDelaySeconds - The number of seconds to wait before retrying.

## API
CacheLoaders are exposed for API use through the `cache.getWithLoader(...)` method. CacheWriters are exposed with
`cache.putWithWriter(...)` and `cache.removeWithWriter(...)` methods.
For example, following is the method signature for `cache.putWithWriter(...)`.

    /**
    * Put an element in the cache writing through a CacheWriter. If no CacheWriter has been
    * set for the cache, then this method has the same effect as cache.put().
    *
    * Resets the access statistics on the element, which would be the case if it has previously
    * been gotten from a cache, and is now being put back.
    *
    * Also notifies the CacheEventListener, if the writer operation succeeds, that:
    *
    * - the element was put, but only if the Element was actually put.
    * - if the element exists in the cache, that an update has occurred, even if the element
    * would be expired if it was requested
    *
    *
    * @param element An object. If Serializable it can fully participate in replication and the
    * DiskStore.
    * @throws IllegalStateException    if the cache is not
    *    {@link net.sf.ehcache.Status#STATUS_ALIVE}
    * @throws IllegalArgumentException if the element is null
    * @throws CacheException
    */
    void putWithWriter(Element element) throws IllegalArgumentException, IllegalStateException,
    CacheException;

See the Cache JavaDoc for the complete API.

## SPI
The Ehcache write-through SPI is the `CacheWriter` interface. Implementers perform writes to the underlying resource
in their implementation.

    /**
    * A CacheWriter is an interface used for write-through and write-behind caching to a
    * underlying resource.
    * <p/>
    * If configured for a cache, CacheWriter's methods will be called on a cache operation.
    * A cache put will cause a CacheWriter write
    * and a cache remove will cause a writer delete.
    * <p>
    * Implementers should create an implementation which handles storing and deleting to an
    * underlying resource.
    * </p>
    * <h4>Write-Through</h4>
    * In write-through mode, the cache operation will occur and the writer operation will occur
    * before CacheEventListeners are notified. If
    * the write operation fails an exception will be thrown. This can result in a cache which
    * is inconsistent with the underlying resource.
    * To avoid this, the cache and the underlying resource should be configured to participate
    * in a transaction. In the event of a failure
    * a rollback can return all components to a consistent state.
    * <p/>
    * <h4>Write-Behind</h4>
    * In write-behind mode, writes are written to a write-behind queue. They are written by a
    * separate execution thread in a configurable
    * way. When used with Terracotta Server Array, the queue is highly available. In addition
    * any node in the cluster may perform the
    * write-behind operations.
    * <p/>
    * <h4>Creation and Configuration</h4>
    * CacheWriters can be created using the CacheWriterFactory.
    * <p/>
    * The manner upon which a CacheWriter is actually called is determined by the
    * {@link net.sf.ehcache.config.CacheWriterConfiguration} that is set up for a cache
    * using the CacheWriter.
    * <p/>
    * See the CacheWriter chapter in the documentation for more information on how to use writers.
    *
    * @author Greg Luck
    * @author Geert Bevin
    * @version $Id: $
    */
    public interface CacheWriter {
    /**
    * Creates a clone of this writer. This method will only be called by ehcache before a
    * cache is initialized.
    * <p/>
    * Implementations should throw CloneNotSupportedException if they do not support clone
    * but that will stop them from being used with defaultCache.
    *
    * @return a clone
    * @throws CloneNotSupportedException if the extension could not be cloned.
    */
    public CacheWriter clone(Ehcache cache) throws CloneNotSupportedException;
     /**
    * Notifies writer to initialise themselves.
    * <p/>
    * This method is called during the Cache's initialise method after it has changed it's
    * status to alive. Cache operations are legal in this method.
    *
    * @throws net.sf.ehcache.CacheException
    */
    void init();
    /**
    * Providers may be doing all sorts of exotic things and need to be able to clean up on
    * dispose.
    * <p/>
    * Cache operations are illegal when this method is called. The cache itself is partly
    * disposed when this method is called.
    */
    void dispose() throws CacheException;
    /**
    * Write the specified value under the specified key to the underlying store.
    * This method is intended to support both key/value creation and value update for a
    * specific key.
    *
    * @param element the element to be written
    */
    void write(Element element) throws CacheException;
    /**
    * Write the specified Elements to the underlying store. This method is intended to
    * support both insert and update.
    * If this operation fails (by throwing an exception) after a partial success,
    * the convention is that entries which have been written successfully are to be removed
    * from the specified mapEntries, indicating that the write operation for the entries left
    * in the map has failed or has not been attempted.
    *
    * @param elements the Elements to be written
    */
    void writeAll(Collection<Element> elements) throws CacheException;
     /**
    * Delete the cache entry from the store
    *
    * @param entry the cache entry that is used for the delete operation
    */
    void delete(CacheEntry entry) throws CacheException;
     /**
    * Remove data and keys from the underlying store for the given collection of keys, if
    * present. If this operation fails * (by throwing an exception) after a partial success,
    * the convention is that keys which have been erased successfully are to be removed from
    * the specified keys, indicating that the erase operation for the keys left in the collection
    * has failed or has not been attempted.
    *
    * @param entries the entries that have been removed from the cache
    */
    void deleteAll(Collection<CacheEntry> entries) throws CacheException;
      /**
     * This method will be called whenever an Element couldn't be handled by the writer and all of
     * the {@link net.sf.ehcache.config.CacheWriterConfiguration#getRetryAttempts() retryAttempts}
     * have been tried.
     * <p>When batching is enabled, all of the elements in the failing batch will be passed to
     * this method.
     * <p>Try to not throw RuntimeExceptions from this method. Should an Exception occur,
     * it will be logged, but the element will still be lost.
     * @param element the Element that triggered the failure, or one of the elements in the
     * batch that failed.
     * @param operationType the operation we tried to execute
     * @param e the RuntimeException thrown by the Writer when the last retry attempt was
     * being executed
     */
    void throwAway(Element element, SingleOperationType operationType, RuntimeException e);

    }

## FAQ

#### Is there a way to monitor the write-behind queue size?
Use the method `net.sf.ehcache.statistics.LiveCacheStatistics#getWriterQueueLength()`. This method returns the number of elements on the local queue (in all local buckets) that are waiting to be processed, or -1 if no write-behind queue exists. Note that elements or a batch currently being processed (and coalesced elements) are not included in the returned value.

#### What happens if an exception occurs when the writer is called?
Once all retry attempts have been executed, on exception the element (or all elements of that batch) will be passed to the `net.sf.ehcache.writer.CacheWriter#throwAway` method.
The user can then act one last time on the element that failed to write. A reference to the last thrown RuntimeException, and the type of operation that failed to execute for the element, are received.
Any Exception thrown from that method will simply be logged and ignored. The element will be lost forever. It is important that implementers are careful about proper Exception handling in that last method.

A handy pattern is to use an eternal cache (potentially using a writer, so it is persistent) to store failed operations and their element. Users can monitor that cache and manually intervene on those errors at a later point.
