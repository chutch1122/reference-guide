# Optimizing Aggregate Loading

## Snapshotting

When aggregates live for a long time, and their state constantly changes, they will generate a large amount of events. Having to load all these events in to rebuild an aggregate's state may have a big performance impact. The snapshot event is a domain event with a special purpose: it summarises an arbitrary amount of events into a single one. By regularly creating and storing a snapshot event, the event store does not have to return long lists of events. Just the last snapshot events and all events that occurred after the snapshot was made.

For example, items in stock tend to change quite often. Each time an item is sold, an event reduces the stock by one. Every time a shipment of new items comes in, the stock is incremented by some larger number. If you sell a hundred items each day, you will produce at least 100 events per day. After a few days, your system will spend too much time reading in all these events just to find out whether it should raise an "ItemOutOfStockEvent". A single snapshot event could replace a lot of these events, just by storing the current number of items in stock.

### Creating a snapshot

Snapshot creation can be triggered by a number of factors, for example the number of events created since the last snapshot, the time to initialize an aggregate exceeds a certain threshold, time-based, etc. Currently, Axon provides a mechanism that allows you to trigger snapshots based on an event count threshold.

The definition of when snapshots should be created, is provided by the `SnapshotTriggerDefinition` interface.

The `EventCountSnapshotTriggerDefinition` provides the mechanism to trigger snapshot creation when the number of events needed to load an aggregate exceeds a certain threshold. If the number of events needed to load an aggregate exceeds a certain configurable threshold, the trigger tells a `Snapshotter` to create a snapshot for the aggregate.

The snapshot trigger is configured on an event sourcing repository and has a number of properties that allow you to tweak triggering:

* `Snapshotter` sets the actual snapshotter instance, responsible for creating and storing the actual snapshot event;
* `Trigger` sets the threshold at which to trigger snapshot creation;

A `Snapshotter` is responsible for the actual creation of a snapshot. Typically, snapshotting is a process that should disturb the operational processes as little as possible. Therefore, it is recommended to run the snapshotter in a different thread. The `Snapshotter` interface declares a single method: `scheduleSnapshot()`, which takes the aggregate's type and identifier as parameters.

Axon provides the `AggregateSnapshotter`, which creates and stores `AggregateSnapshot` instances. This is a special type of snapshot, since it contains the actual aggregate instance within it. The repositories provided by Axon are aware of this type of snapshot, and will extract the aggregate from it, instead of instantiating a new one. All events loaded after the snapshot events are streamed to the extracted aggregate instance.

> **Note**
>
> Do make sure that the `Serializer` instance you use \(which defaults to the `XStreamSerializer`\) is capable of serializing your aggregate. The `XStreamSerializer` requires you to use either a Hotspot JVM, or your aggregate must either have an accessible default constructor or implement the `Serializable` interface.

The `AbstractSnapshotter` provides a basic set of properties that allow you to tweak the way snapshots are created:

* `EventStore` sets the event store that is used to load past events and store the snapshots. This event store must implement the `SnapshotEventStore` interface.
* `Executor` sets the executor, such as a `ThreadPoolExecutor` that will provide the thread to process actual snapshot creation. By default, snapshots are created in the thread that calls the `scheduleSnapshot()` method, which is generally not recommended for production.

The `AggregateSnapshotter` provides one more property:

* `AggregateFactories` is the property that allows you to set the factories that will create instances of your aggregates. Configuring multiple aggregate factories allows you to use a single `Snapshotter` to create snapshots for a variety of aggregate types. The `EventSourcingRepository` implementations provide access to the `AggregateFactory` they use. This can be used to configure the same aggregate factories in the Snapshotter as the ones used in the repositories.

> **Note**
>
> If you use an executor that executes snapshot creation in another thread, make sure you configure the correct transaction management for your underlying event store, if necessary.
>
> Spring users can use the `SpringAggregateSnapshotter`, which will automatically look up the right `AggregateFactory` from the application context when a snapshot needs to be created.

{% tabs %}
{% tab title="Axon Configuration API" %}

```java

Configurer configurer = DefaultConfigurer.defaultConfiguration()
                .configureAggregate(AggregateConfigurer.defaultConfiguration(GiftCard.class).configureSnapshotTrigger(c -> new EventCountSnapshotTriggerDefinition(AggregateSnapshotter.builder().eventStore(c.eventStore()).build(),300)));

```
{% endtab %}
{% tab title="Spring Boot AutoConfiguration" %}

It is possible to define a custom `SnapshotTriggerDefinition` for an aggregate as a spring bean. In order to tie the `SnapshotTriggerDefinition` bean to an aggregate, use the `snapshotTriggerDefinition` attribute on `@Aggregate` annotation. Listing below shows how to define a custom `EventCountSnapshotTriggerDefinition` which will take a snapshot on each five hundredths event.

Note that a `Snapshotter` instance, if not explicitly defined as a bean already, will be automatically configured for you. This means you can simply pass the `Snapshotter` as a parameter to your `SnapshotTriggerDefinition`.

```java
@Bean
public SnapshotTriggerDefinition mySnapshotTriggerDefinition(Snapshotter snapshotter) {
    return new EventCountSnapshotTriggerDefinition(snapshotter, 500);
}

...

@Aggregate(snapshotTriggerDefinition = "mySnapshotTriggerDefinition")
public class MyAggregate {...}
```
{% endtab %}
{% endtabs %}

### Storing Snapshot Events

When a snapshot is stored in the event store, it will automatically use that snapshot to summarize all prior events and return it in their place. All event store implementations allow for concurrent creation of snapshots. This means they allow snapshots to be stored while another process is adding events for the same aggregate. This allows the snapshotting process to run as a separate process altogether.

> **Note**
>
> Normally, you can archive all events once they are part of a snapshot event. Snapshotted events will never be read in again by the event store in regular operational scenarios. However, if you want to be able to reconstruct aggregate state prior to the moment the snapshot was created, you must keep the events up to that date.

Axon provides a special type of snapshot event: the `AggregateSnapshot`, which stores an entire aggregate as a snapshot. The motivation is simple: your aggregate should only contain the state relevant to take business decisions. This is exactly the information you want captured in a snapshot. All event sourcing repositories provided by Axon recognize the `AggregateSnapshot`, and will extract the aggregate from it. Beware that using this snapshot event requires that the event serialization mechanism needs to be able to serialize the aggregate.

### Initializing an Aggregate based on a Snapshot Event

A snapshot event is an event like any other. That means a snapshot event is handled just like any other domain event. When using annotations to demarcate event handlers \(`@EventHandler`\), you can annotate a method that initializes full aggregate state based on a snapshot event. The code sample below shows how snapshot events are treated like any other domain event within the aggregate.

```java
public class MyAggregate extends AbstractAnnotatedAggregateRoot {

    // ... 
    
    @EventHandler
    protected void handleSomeStateChangeEvent(MyDomainEvent event) {
        // ...
    }

    @EventHandler
    protected void applySnapshot(MySnapshotEvent event) {
        // the snapshot event should contain all relevant state
        this.someState = event.someState;
        this.otherState = event.otherState;
    }
}
```

There is one type of snapshot event that is treated differently: the `AggregateSnapshot`. This type of snapshot event contains the actual aggregate. The aggregate factory recognizes this type of event and extracts the aggregate from the snapshot. Then, all other events are re-applied to the extracted snapshot. That means aggregates never need to be able to deal with `AggregateSnapshot` instances themselves.

## Caching

A well designed command handling module should pose no problems when implementing caching. Especially when using event sourcing, loading an aggregate from an Event Store is an expensive operation. With a properly configured cache in place, loading an aggregate can be converted into a pure in-memory process.

Here are a few guidelines that help you get the most out of your caching solution:

* Make sure the unit of work never needs to perform a rollback for functional reasons.

  A rollback means that an aggregate has reached an invalid state. Axon will automatically invalidate the cache entries involved. The next request will force the aggregate to be reconstructed from its events. If you use exceptions as a potential \(functional\) return value, you can configure a `RollbackConfiguration` on your command bus. By default, the unit of work will be rolled back on runtime exceptions for command handlers, and on all exceptions for event handlers.

* All commands for a single aggregate must arrive on the machine that has the aggregate in its cache.

  This means that commands should be consistently routed to the same machine, for as long as that machine is "healthy". Routing commands consistently prevents the cache from going stale. A hit on a stale cache will cause a command to be executed and fail at the moment events are stored in the event store. By default, Axon's distributed command bus components will use consistent hashing to route commands.

* Configure a sensible time to live / time to idle

  By default, caches have a tendency to have a relatively short time to live, a matter of minutes. For a command handling component with consistent routing, a longer time-to-idle and time-to-live is usually better. This prevents the need to re-initialize an aggregate based on its events, just because its cache entry expired. The time-to-live of your cache should match the expected lifetime of your aggregate.

* Cache data in-memory

  For true optimziation, caches should keep data in-memory \(and preferably on-heap\) to best performance. This prevents the need to \(se\)serialize aggregates when storing to disk and even off-heap.
