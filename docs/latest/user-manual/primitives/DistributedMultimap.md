---
layout: user-manual
project: atomix
menu: user-manual
title: DistributedMultimap
---

The [`DistributedMultimap`][DistributedMultimap] an implementation of Guava's `Multimap`. The default implementation is a set-based multimap where values are sets. All operations on the `DistributedMultimap` are guaranteed to be atomic. And unlike [`AtomicMultimap`][AtomicMultimap], `DistributedMultimap` supports eventually consistent protocols. 

[`DistributedMultimap`][DistributedMultimap] supports event-based notifications of changes to the multimap. Clients can listen for inserted/updated/removed entries by registering event listeners on a multimap.

Finally, [`DistributedMultimap`][DistributedMultimap] supports key set, values, and entry set views that are iterable and support Java 8 streams.

## Configuration

The [`DistributedMultimap`][DistributedMultimap] can be configured programmatically using the [`DistributedMultimapBuilder`][DistributedMultimapBuilder]. To create a new multimap builder, use the `multimapBuilder` method, passing the name of the multimap to construct:

```java
DistributedMultimapBuilder<String, String> multimapBuilder = atomix.<String, String>multimapBuilder("my-multimap");
```

The multimap can be configured with a [protocol][primitive-protocols] to use to replicate changes. Although `DistributedMultimap` can be replicated via eventually consistent protocols, it currently only supports partition group protocols:
* [`MultiRaftProtocol`][MultiRaftProtocol]
* [`MultiPrimaryProtocol`][MultiPrimaryProtocol]

When using partitioned protocols like the ones above, the multimap will be partitioned and replicated among all partitions in the configured [partition group][partition-groups].

```java
MultiRaftProtocol protocol = MultiRaftProtocol.builder()
  .withReadConsistency(ReadConsistency.LINEARIZABLE)
  .build();

Multimap<String, String> multimap = atomix.<String, String>multimapBuilder("my-multimap")
  .withProtocol(protocol)
  .build();
```

The generic parameters in the multimap configuration are the multimap key and value types. By default, arbitrary key and value types may be used. However, when non-standard types are used, class names will be serialized with map entries, and the thread context class loader will be used to load classes from names. To avoid serializing class names, register a key and value type via `withKeyType` and `withValueType`. Class-based serialization can also be disabled via `withRegistrationRequired()`.

```java
Multimap<String, MemberId> multimap = atomix.<String, MemberId>multimapBuilder("my-multimap")
  .withProtocol(protocol)
  .withKeyType(String.class)
  .withValueType(MemberId.class)
  .build();
```

Multimaps support caching. When caching is enabled, the multimap will transparently listen for change events and update a local cache. To enable caching, use `withCacheEnabled()`:

```java
Multimap<String, String> multimap = atomix.<String, String>multimapBuilder("my-multimap")
  .withProtocol(protocol)
  .withCacheEnabled()
  .withCacheSize(1000)
  .build();
```

A multimap can also be constructed in read-only mode using `withReadOnly()`:

```java
Multimap<String, String> multimap = atomix.<String, String>multimapBuilder("my-multimap")
  .withProtocol(protocol)
  .withReadOnly()
  .build();
```

Multimaps can also be configured in configuration files. To configure a multimap primitive, use the `multimap` primitive type:

`atomix.conf`

```hocon
primitives.my-multimap {
  type: multimap
  cache.enabled: true
  protocol {
    type: multi-raft
    group: raft
    read-consistency: linearizable
  }
}
```

To get an instance of the pre-configured multimap, use the `getMultimap` method:

```java
DistributedMultimap<String, String> multimap = atomix.getMultimap("my-multimap");
```

The map's protocol and configuration will be loaded from the Atomix configuration files.

## Operation

The [`DistributedMultimap`][DistributedMultimap] supports most of the same operations as Guava's `Multimap`. All operations performed on the multimap are guaranteed to be atomic. Beyond that atomicity guarantee, the consistency guarantees of read and write operations are specified by the configured protocol.

```java
Multimap<String, String> multimap = atomix.<String, String>multimapBuilder("my-multimap")
  .withProtocol(MultiRaftProtocol.builder()
    .withReadConsistency(ReadConsistency.LINEARIZABLE)
    .build())
  .build();

multimap.put("foo", "bar");
multimap.put("foo", "baz");

Collection<String> values = multimap.get("foo");
```

### Asynchronous Operations

As with all Atomix primitives, an asynchronous analogue of the multimap API - [`AsyncDistributedMultimap`][AsyncDistributedMultimap] - can be retrieved by calling the `async()` method:

```java
Multimap<String, String> multimap = atomix.getMultimap("my-multimap");

AsyncDistributedMultimap<String, String> asyncMultimap = multimap.async();

multimap.putAll("foo", Arrays.asList("bar", "baz")).thenRun(() -> {
  multimap.get("foo").thenAccept(values -> {
    ...
  });
});
```

The asynchronous API uses [`CompletableFuture`][CompletableFuture]s to notify the client once an operation is complete. The thread model provided by all Atomix protocols guarantees `CompletableFuture` callbacks will always be executed on the same thread unless a thread is blocked by a prior primitive operation. Additionally, `CompletableFuture`s will be completed in program order. In other words, if an operation `A` was performed before operation `B` on the client, the future for operation `A` will always be completed before the future for operation `B`.

### Event Notifications

[`DistributedMultimap`][DistributedMultimap] supports publishing event notifications to client listeners. This allows clients to react to insert and remove operations on the multimap. To add a listener to a multimap, simply register the listener via `addListener`:

```java
multimap.addListener(event -> {
  ...
});
```

Atomix guarantees that events will be received in the order in which they occurred inside replicated state machines, and event listeners will be called on an Atomix event thread. Users can optionally provide a custom executor on which to call the event listener:

```java
Executor executor = Executors.newSingleThreadExecutor();
multimap.addListener(event -> {
  ...
}, executor);
```

{:.callout .callout-warning}
Custom executors can change the ordering of events. It's recommended that single thread executors be used to preserve order. Multi-threaded executors cannot provide the same guarantees as are provided by Atomix event threads or single thread executors.

The event listener will be called with an [`MultimapEvent`][MultimapEvent] instance. Each event in Atomix has an associated type which can be read via the `type()` method. To determine the type of modification that took place, use a switch statement:

```java
switch (event.type()) {
  case INSERT:
    ...
    break;
  case REMOVE:
    ...
    break;
}
```

The [`MultimapEvent`][MultimapEvent] provides both the previous value and the updated value for all updates. The previous value can be read via `oldValue()` and the updated value via `newValue()`.

```java
String value;
if (event.type() == MultimapEvent.Type.REMOVE) {
  value = event.oldValue();
} else {
  value = event.newValue();
}
```

The Atomix thread model allows for event listeners to make blocking calls on primitives within event threads. So, in response to an update event, an event listener can e.g. call `put` on the same multimap:

```java
// Recreate entries that are removed from the multimap
multimap.addListener(event -> {
  if (event.type() == MultimapEvent.Type.REMOVE) {
    multimap.put(event.key(), event.newValue().value());
  }
});
```

When performing blocking operations (any operation on a synchronous primitive) within an event threads, additional events and futures will be completed on a background thread pool. This means ordering guarantees are inherently relaxed when event threads are blocked.

### Multimap Views

The [`DistributedMultimap`][DistributedMultimap] primitive supports views that are common to Java maps:
* `keySet()` returns a `DistributedSet` primitive which implements [`java.util.Set`][Set]
* `keys()` returns a `DistributedMultiset` primitive which implements Guava's `Multiset`
* `values()` returns a `DistributedCollection` primitive which implements [`java.util.Collection`][Collection]
* `entries()` returns a `DistributedCollection` primitive view of each key/value mapping which implements [`java.util.Collection`][Collection]

The collections are views of the `DistributedMultimap`'s state, so changes to the multimap will be reflected in the key set, values, or entry set and vice versa.

```java
multimap.put("foo", "bar");

assert multimap.keySet().contains("foo");

multimap.keySet().remove("foo");

assert !multimap.containsKey("foo");
```

When using the [`AsyncDistributedMultimap`][AsyncDistributedMultimap] API, asynchronous analogues of the view primitives will be returned instead.

```java
multimap.put("foo", "bar");

multimap.async().keySet().remove("foo").thenRun(() -> {
  assert !multimap.containsKey("foo");
});
```

### Iterators

All [`DistributedMultimap`][DistributedMultimap] views support lazy iterators:

```java
for (String key : multimap.keySet()) {
  ...
}
```

Iterators are implemented by lazily fetching batches of keys/values/entries from each partition as the items are iterated. Once a primitive iterator has been created, it must either be exhausted or explicitly `close()`d to ensure the resources used to track the iterator state is cleaned up.

```java
Iterator<String> keyIterator = multimap.keySet().iterator();

try {
  String key = keyIterator.next();
} finally {
  iterator.close();
}
```

{:.callout .callout-warning}
Failing to exhaust or explicitly close frequently created primitive iterators may cause a memory leak.

Just as with typical synchronous primitives, the iterators provided for Atomix primitives are backed by an asynchronous implementation called [`AsyncIterator`][AsyncIterator], and the asynchronous backing iterator can be retrieved via the `async()` method:

```java
AsyncIterator<String> asyncIterator = multimap.values().iterator().async();
// or...
AsyncIterator<String> asyncIterator = multimap.async().values().iterator();
```

### Streams

The implementation of lazy iterators for multimap views also allows the multimap to support Java 8 [`Stream`][Stream]s:

```java
Set<String> fooKeys = multimap.entries().stream()
  .filter(entry -> entry.getValue().equals("foo"))
  .map(entry -> entry.getKey())
  .collect(Collectors.toSet());
```

## Cleanup

While a multimap is in use, Atomix may consume some network, memory, and disk resources to manage the multimap. To free up those resources, users should call the `close()` method to allow Atomix to garbage collect the instance.

```java
multimap.close();
```

{% include common-links.html %}
