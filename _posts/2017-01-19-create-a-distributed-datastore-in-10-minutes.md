---
layout: post
title:  How to Create a Distributed Datastore in 10 Minutes
date: 2017-01-19
---

While the availability of numerous excellent open technologies has enabled us to build products and services faster than ever, there remain problem areas where technologies have yet to fully impact. One of the most challenging areas involves building consistent, fault-tolerant distributed systems.

In a typical system you pull data in and out of somewhere that is safe, such as [ZooKeeper] or [etcd]. Then you operate on that data in places that are not necessarily safe, such as your application. And while etcd and ZooKeeper provide guarantees about the consistency of data in their possession, they cannot guarantee the consistency of broader state transitions and transactions that may involve other parts of your application. For that, we need another approach.

## Changing the Paradigm

[Copycat] is a framework that sidesteps the standard paradigm of building applications dependent on external systems for data consistency by allowing you to embed your application logic, written as a [state machine][copycat-state-machines], *directly into* Copycat, where consistency and fault tolerance are taken care of for you. The result is an ability to implement solutions to complex distributed coordination problems in a way that is relatively simple and concise and that encapsulates the logic and semantics of your application, without having to worry about reliability guarantees.

What kind of things can we build with Copycat? It's up to you. From low level distributed primitives like [locks][dlock], [groups][dgroup] and [maps][dmap] to full blown distributed systems for scheduling, messaging, service discovery or data storage, nearly anything is possible.

## From Zero to Distributed Datastore

A good place to start with Copycat is to build something with it, so let's create a distributed key-value datastore. We're not aiming to create just any datastore though, we want something with strong consistency guarantees, fault-tolerance in the case of network partitions, durability in the case of node failure, and notifications for data changes - something along the lines of etcd. Is it really possible to build an etcd clone in 10 minutes? Well, no, but we can amazingly close, building a datastore with the same basic features, and more importantly, the same reliability guarantees, in that amount of time.

## The State Machine

The first step in building our datastore is to define a state machine to contain our datastore's state and logic. Since our datastore is storing key-value pairs, we'll encapsulate data in memory using a simple `HashMap`. Seriously, a HashMap? What about thread-safety? What about durability? Copycat will take care of these things for us as we'll learn more about later. But first, let's define our state machine:

```java
public class KeyValueStore extends StateMachine {
  private Map<Object, Object> storage = new HashMap<>();
}
```

In order to operate on our state machine, we'll need to declare some operations. Copycat supports two types of operations: *commands* which are intended for *writes* and *queries* which are intended *reads*. Let's start by defining some of the basic etcd-style operations: `put`, `get` and `delete`:

```java
public class Put implements Command<Object> {
  public Object key;
  public Object value;

  public Put(Object key, Object value) {
    this.key = key;
    this.value = value;
  }
}

public class Get implements Query<Object> {
  public Object key;

  public Get(Object key) {
    this.key = key;
  }
}

public class Delete implements Command<Object> {
  public Object key;

  public Delete(Object key) {
    this.key = key;
  }
}
```

With our operations defined, let's implement handling for them inside of our `StateMachine`:

```java
public class KeyValueStore extends StateMachine {
  private Map<Object, Commit> storage = new HashMap<>();

  public Object put(Commit<Put> commit) {
    Commit<Put> put = storage.put(commit.operation().key, commit);
    return put == null ? null : put.operation().value;
  }

  public Object get(Commit<Get> commit) {
    try {
      Commit<Put> put = map.get(commit.operation().key);
      return put == null ? null : put.operation().value;
    } finally {
      commit.release();
    }
  }

  public Object delete(Commit<Delete> commit) {
    Commit<Put> put = null;
    try {
      put = storage.remove(commit.operation().key);
      return put == null ? null : put.operation().value;
    } finally {
      if (put != null)
        put.release();
      commit.release();
    }
  }
}
```

As you can see, the `put`, `get` and `delete` implementations handle `Commit` objects that contain operations submitted to the state machine. Operations are executed on a single thread, so thread-safety is a non-issue, and after being handled, operations return a result that reflects the internal state of our machine.

Aside from the state machine's storage, Copycat also stores an internal [log] of every command processed by the state machine along with its result, which it uses for failure handling and other purposes. Periodic compaction is performed on the log in order to remove commits that are no longer needed.  To help Copycat know when it's safe to remove a commit from its log, the state machine should `release` commits that do not contribute to the state of the machine. A `Put` operation, for example, is not released until after a `Delete` operation is received for the same key. A `Get` operation, on the other hand, is released right away since it does not contribute to the state of the machine.

With this, our basic key-value store is now implemented! We'll add some more advanced operations later, but now let's get ready to try it out.

## Creating the Server

To manage our state machine we'll need to build a `CopycatServer` instance. The server must be initialized with an address to listen for communication on:

```java
Address address = new Address("123.456.789.0", 5000);
CopycatServer.Builder builder = CopycatServer.builder(address);
```

We'll configure the server to use our state machine:

```java
builder.withStateMachine(KeyValueStore::new);
```

And configure a [`Transport`][Transport] for the server to use when communicating with clients other servers in a cluster:

```java
builder.withTransport(NettyTransport.builder()
  .withThreads(4)
  .build());
```

We'll configure a [`Storage`][Storage] implementation for our state machine's log, using on disk storage in this case:

```java
builder.withStorage(Storage.builder()
  .withDirectory(new File("logs"))
  .withStorageLevel(StorageLevel.DISK)
  .build());
```

And finally we'll create the server:

```java
CopycatServer server = builder.build();
```

## Bootstrapping a Cluster

Once a server has been built, we can use it to bootstrap a new cluster:

```java
server.bootstrap().thenAccept(srvr -> System.out.println(srvr + " has bootstrapped a cluster"));
```

At this point our state machine is up and running, but let's `join` some additional servers to the cluster:

```java
Address clusterAddress = new Address("123.456.789.0", 5000);
server.join(clusterAddress).thenAccept(srvr -> System.out.println(srvr + " has joined the cluster"));
```

And just like that, we have created a clustered key-value store!

## Performing Operations

In order to submit operations to our datastore, we'll need to create a `CopycatClient`. We'll be sure to configure the same `Transport` for our client that we configured for our servers:

```java
CopycatClient client = CopycatClient.builder()
  .withTransport(NettyTransport.builder()
    .withThreads(2)
    .build())
  .build();
```

Then we'll point our client to any of the servers in our cluster, and `connect`:

```java
Address clusterAddress = new Address("123.456.789.0", 5000);
client.connect(clusterAddress).join();
```

With our client connected, let's submit a `put` operation:

```java
CompletableFuture<Object> future = client.submit(new Put("foo", "Hello world!"));
Object result = future.get();
```

We can also submit `get` and `delete` operations in the same way as a `put`:

```java
client.submit(new Get("foo")).thenAccept(result -> System.out.println("foo is: " + result));
client.submit(new Delete("foo")).thenRun(() -> System.out.println("foo has been deleted"));
```

From here we can wrap the client in a CLI or REST API to allow other types of access, but we'll leave that as an exercise for another time.

## Achieving Consistency

Now that we have an initial system up and running, let's take a step back to discuss what's going on under the covers. Remember at the outset when we stated that it's not enough to build our own key-value store, we want it to be fully replicated, durable, strongly consistent, and able to handle failures. How do we do all that? It turns out, we already have.

Copycat utilizes a [sophisticated implementation][consensus] of the [Raft consensus][raft] algorithm to ensure that every operation against your state machine is replicated to every member of the cluster, in a safe way. To accomplish this, each server in the cluster maintains a separate copy of the state machine along with a [log] of all operations that have been performed on the state machine and their results. Logs can be durably stored according to the configured [`StorageLevel`][StorageLevel] and are used to restore the state of a machine in the event of a failure.

In order to achieve strong consistency, Copycat utilizes a majority quorum to ensure that write operations are approved by a majority of nodes in the cluster before they take effect. In the event of a network partition or system failure where a quorum can no longer be achieved, Copycat will cease to process write operations in order to prevent data inconsistency from occurring.

Copycat clusters elect a leader to serve as the focal point for processing operations. When a command is submitted by a client to a server, it's forwarded to the leader which in turn sends the command to the rest of the cluster. Each server then applies the command to its state machine, appends the result to its log, and returns a response to the leader. Once the leader has received a response from a majority of the cluster (including itself), it applies the command to its own state machine and log then sends a response back to the client.

Copycat supports configurable [consistency levels][ConsistencyLevel] per query operation. When a query is submitted by a client to a server, it can either be forwarded to the leader if linearizable consistency is desired, or it can be responded to by any server if sequential consistency is sufficient.

## Achieving Fault-Tolerance

Copycat utilizes heartbeats and timeouts to assert healthy connectivity between servers. If a leader fails to issue a heartbeat within the configured timeout period, the remaining members of the cluster will elect a new leader to coordinate the processing of operations. Likewise, if a follower fails to respond to a heartbeat, that server may be removed from the cluster. 

Since Copycat requires a majority quorum in order to maintain consistency and remain available, Copycat supports [passive and reserve][membership] servers which can be made to replace active servers in the event of a failure. When a new server joins the cluster, the leader streams its log to the server which in turn applies the logged operations to its state machine. Once the server is fully caught up, the leader will promote the new server to an active member of the cluster.

Now that we understand a little about how Copycat turns our basic state machine into a robust, distributed key-value store, let's turn back to our implementation and add a few more advanced capabilities.

## Time to Live

One nice feature that etcd supports is time-to-live for keys. This allows keys to be auto-deleted after a certain time period. Let's add TTL support to our datastore. We'll start by defining a new `PutWithTtl` command:

```java
public class PutWithTtl implements Command<Object> {
  public Object key;
  public Object value;
  public long ttl;

  public PutWithTtl(Object key, Object value, long ttl) {
    this.key = key;
    this.value = value;
    this.ttl = ttl;
  }

  @Override
  public CompactionMode compaction() {
    return CompactionMode.EXPIRING;
  }
}
```

Since a `PutWithTtl` command should result in the removal of state after some amount of time, we need to indicate this to Copycat so that it can properly compact these commits from the log. We do this by providing a `compaction()` implementation that returns `CompactionMode.EXPIRING`.

Next we'll need to implement handling of the `PutWithTtl` command inside of our state machine:

```java
public Object putWithTtl(Commit<PutWithTtl> commit) {
  Object result = storage.put(commit.operation().key, commit);
  executor.schedule(Duration.ofMillis(commit.operation().ttl), () -> {
    storage.remove(commit.operation().key);
    commit.release();
  });
  return result;
}
```

Here we schedule a future action to execute after the TTL has been exceeded, which will remove the commit from storage and release it, similar to our `delete` implementation from earlier. We use the state machine's internal `executor` to schedule the entry removal since this ensures we won't encounter any thread-safety issues inside of our state machine.

## Watch What Happens

With TTL implemented, let's add one final feature: *watchers*. Watchers in etcd and in ZooKeeper allow clients to receive a notification when a key has been accessed. This an important feature for implementing a variety of coordination patterns, but it typically carries various caveats including rigid semantics and lesser reliability guarantees.

Copycat, on the other hand, provides a session eventing capability that allows arbitrary data to be published directly to clients from anywhere inside a state machine. This flexibility enables us to easily model complex distributed primitives such as [groups][group-join], [leader election][leader-election], and [messaging][message-consumers] where server-side information is published to clients in an efficient and semantically appropriate manner. Session events are guaranteed not to be lost in the case of a server failure and are always delivered in sequential order.

To leverage session events for our datastore, we'll start by defining a new `Listen` command that will indicate a client's interest in receiving events from our state machine:

```java
public class Listen implements Command<Object> {
}
```

Next we'll enhance our `KeyValueStore` implementation to handle the `Listen` command:

```java
public class KeyValueStore extends StateMachine {
  private Map<Object, Commit> storage = new HashMap<>();
  private Set<Commit> listeners = new HashSet<>();
  
  public void listen(Commit<Listen> commit) {
    listeners.add(commit);
  }
```

The `listen` method simply stores the client submitted commit, which we'll later use to publish events back to the client. We'll need to define an `EntryEvent` type that will encapsulate our event data:

```java
public class EntryEvent<K, V> implements Serializable {
  public Object key;
  public Object oldValue;
  public Object newValue;

  public EntryEvent(Object key, Object oldValue, Object newValue) {
    this.key = key;
    this.oldValue = oldValue;
    this.newValue = newValue;
  }
  
  public String toString() {
    return String.format("EntryEvent [key=%s, oldValue=%s, newValue=%s]", key, oldValue, newValue);
  }
}
```

And finally we'll enhance our `KeyValueStore` to publish `EntryEvent`s from within our existing command handlers using the client session associated with any `Listen` commands:

```java
private void publish(String event, Object key, Object oldValue, Object newValue) {
  listeners.forEach(commit -> {
    commit.session().publish(event, new EntryEvent(key, oldValue, newValue));
  });
}

public Object put(Commit<Put> commit) {
  Commit<Put> put = storage.put(commit.operation().key, commit);
  Object oldValue = put == null ? null : put.operation().value;
  publish("put", commit.operation().key, oldValue, commit.operation().value);
  return oldValue;
}

public Object putWithTtl(Commit<PutWithTtl> commit) {
  Object result = storage.put(commit.operation().key, commit);
  executor.schedule(Duration.ofMillis(commit.operation().ttl), () -> {
    Commit<PutWithTtl> put = storage.remove(commit.operation().key);
    Object oldValue = put == null ? null : put.operation().value;
    publish("expire", commit.operation().key, oldValue, null);
    commit.release();
  });
  return result;
}

public Object delete(Commit<Delete> commit) {
  Commit<Put> put = null;
  try {
    put = storage.remove(commit.operation().key);
    Object oldValue = put == null ? null : put.operation().value;
    publish("delete", commit.operation().key, oldValue, null);
    return oldValue;
  } finally {
    if (put != null)
      put.release();
    commit.release();
  }
}
```

On the client side, we'll publish a `Listen` command to indicate our interest in receiving events:

```java
client.submit(new Listen()).thenRun(() -> LOG.info("Now listening for events")).join();
```

Then we can register event listeners for specific events:

```java
client.onEvent("put", (EntryEvent event) -> System.out.println("Put: " + event));
client.onEvent("delete", (EntryEvent event) -> System.out.println("Delete: " + event));
client.onEvent("expire", (EntryEvent event) -> System.out.println("Expire: " + event));
```

Now, when state changes occur within our datastore, clients will be notified.

## Wrapup

Well, that's it. Our 10 minutes are up and with the help of Copycat we've created a production ready, strongly consistent clustered key-value store from scratch. We also learned a bit about consistency and fault tolerance in distributed systems, and hopefully now we're ready to create something else with our new knowledge.

The goal of Copycat and it's sister project [Atomix] isn't to build a clone of any specific technology such as etcd, as achievable as that may now seem. The goal is to empower users to build systems to suit their own needs.

Copycat allows us to build complex systems *faster*, *safer*, and *more easily* than before. So, now that you've seen what it can do, what will you build?
<hr>
*To learn more about Copycat, check out the [project website][copycat]. Also be sure to read about its sister project, [Atomix], a suite of distributed primitives built on Copycat. Source code for the example datastore in this article is available [here][source].*

[copycat]: http://atomix.io/copycat/
[copycat-state-machines]: http://atomix.io/copycat/docs/state-machine/
[copycat-docs]: http://atomix.io/copycat/docs/

[atomix]: http://atomix.io/atomix/
[atomix-community]: http://atomix.io/community/
[gitter-chat]: https://gitter.im/atomix/atomix

[etcd]: https://coreos.com/etcd/
[etcd-api]: https://coreos.com/etcd/docs/latest/api.html

[zookeeper]: https://zookeeper.apache.org/

[membership]: http://atomix.io/copycat/docs/membership/
[reference-counting]: https://en.wikipedia.org/wiki/Reference_counting
[CompletableFuture]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html
[ConsistencyLevel]: http://atomix.io/copycat/api/latest/io/atomix/copycat/Query.ConsistencyLevel.html
[Storage]: http://atomix.io/copycat/api/latest/io/atomix/copycat/server/storage/Storage.html
[StorageLevel]: http://atomix.io/copycat/api/latest/io/atomix/copycat/server/storage/StorageLevel.html
[Transport]: http://atomix.io/catalyst/api/latest/io/atomix/catalyst/transport/Transport.html
[raft]: https://raft.github.io/
[consensus]: http://atomix.io/copycat/docs/architecture-introduction/
[log]: https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying
[dlock]: http://atomix.io/atomix/docs/concurrency/#distributedlock
[dmap]: http://atomix.io/atomix/docs/collections/#distributedmap
[dgroup]: http://atomix.io/atomix/docs/groups/
[source]: https://github.com/jhalterman/jodah/tree/master/key-value-store
[group-join]: http://atomix.io/atomix/docs/groups/#listening-for-members-joining-the-group
[leader-election]: http://atomix.io/atomix/docs/groups/#leader-election
[message-consumers]: http://atomix.io/atomix/docs/groups/#message-consumers