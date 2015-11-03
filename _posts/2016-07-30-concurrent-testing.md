---
layout: post
title:  A guide to testing multi-threaded and asynchronous code
date:   2016-07-30
---

If you've been writing code long enough, or maybe even if you haven't, chances are you've hit on a scenario where you want to test some multi-threaded code. The conventional wisdom is that threads and tests should not mix. Usually this works out fine because the thing that you really want to test just happens to run inside of a multi-threaded system, such as an [Akka] actor or a [Netty] ChannelHandler, and can be tested individually without the use of threads. But what if you can't separate things out, or moreover, what if threading is the point of the code you're testing?

I'm here to tell you that while threads in tests might not be the norm, they are ok. The software police will not arrest you for firing up a thread in a unit test, but how to actually go about testing multi-threaded code is another matter.

While many excellent libraries like Akka and Vert.x provide [test][akka-testing] [kits][vertx-testing] specifically tailored to those technologies, testing something more general such as a multi-threaded data structure requires something different. A good place to start in learning about multi-threaded testing is to take a look at the [JSR-166 test kit][jsr-166-testing]. The JSR-166 specification brought us the many excellent concurrency utilities found in the [`java.util.concurrent`][java-util-concurrent] package. Reading through the unit tests for these utilities, a picture emerges of how to go about testing threaded code.

## Go parallel

The first step is to kick off whatever threaded action you wish to test the outcome of. For example, let's use a hypothetical API to register a message handler against a message bus, and publish a message on the bus which will be delivered asynchronously, in a separate thread, to our handler:

```java
messageBus.registerHandler(message -> {
  System.out.println("Received " + message);
};

messageBus.publish("test");
```

Looks good. When the test runs the bus should deliver our message to the handler on another thread, but this isn't terribly useful since we're not asserting anything. Let's update our test to assert that the message bus delivers our message as expected:

```java
String msg = "test";
messageBus.registerHandler(message -> {
  System.out.println("Received " + message);
  assertEquals(message, msg);
};

messageBus.publish(msg);
```

This seems better. We run our test and it's green. Awesome! But the "Received" message never printed - something isn't quite right.

## Wait a minute

In our test above, when a message is published to the message bus, it's delivered by the bus to the handler on another thread. But when our unit testing tool, such as JUnit, executes our test, it doesn't know anything about the message bus' threads. JUnit only knows about the main thread that it executes our test in. So while the message bus is busy trying to deliver our message, the test finishes execution in the main test thread and JUnit reports success. The solution? We need the main test thread to wait for the message bus to deliver our message. Let's add a sleep statement:

```java
String msg = "test";
messageBus.registerHandler(message -> {
  System.out.println("Received " + message);
  assertEquals(message, msg);
};

messageBus.publish(msg);
Thread.sleep(1000);
```

Our test is green and the `Received` statement prints out as expected. Awesome! But having a 1 second sleep means our test takes at least one second to run - no good. We could lower the sleep time, but then we risk the test ending before the message is received. What we need is a way to coordinate between the main test thread and the message handler's thread. Looking at the [`java.util.concurrent`][java-util-concurrent] package, we're sure to find something we can use. How about a [`CountdownLatch`][CountdownLatch]?

```java
String msg = "test";
CountdownLatch latch = new CountdownLatch(1);
messageBus.registerHandler(message -> {
  System.out.println("Received " + message);
  assertEquals(message, msg);
  latch.countDown();
};

messageBus.publish(msg);
latch.await();
```

In this approach, we're sharing a `CountdownLatch` between the main test thread and our message handler thread. The main thread is made to wait on the latch and the test thread releases the waiting main thread by calling `countDown()` on the latch after the message has been received. We no longer need to sleep for 1 second; our test only takes as long as it needs to.

## Ship it!?

With our new `CountdownLatch` awesomeness, we start writing multi-threaded tests like it's going out of style. But pretty quickly we notice that one of our test cases blocks forever and isn't finishing. What's going on? Consider the message bus scenario: the latch is made to wait, but it only releases after a message is received. If the bus is broken and the message is never delivered then our test never completes. So let's add a timeout to the latch:

```java
latch.await(1, TimeUnit.SECONDS);
```

Now our test that was blocking fails after 1 second with a `TimeoutException`. Eventually we find the problem and fix the test, but we decide to leave the timeouts in place. In case this ever happens again, we'd rather our test suite block for a second and fail than block forever and not complete at all.

Another problem we notice when writing our tests is that they all seem to pass even when they maybe shouldn't. How can this be? Consider our message handling test again:

```java
messageBus.registerHandler(message -> {
  assertEquals(message, msg);
  latch.countDown();
};
```

We had to use a `CountdownLatch` to coordinate the completion of our test with the main test thread, but what about the assertion? If the assertion fails, will JUnit know? It turns out that because we're not performing the assertion in the main test thread, any failure of the assertion goes completely unnoticed by JUnit. Let's try a little scenario to verify this:

```java
CountdownLatch latch = new CountdownLatch(1);
new Thread(() -> {
  assertTrue(false);
  latch.countDown();
}).start();

latch.await();
```

Ugh, the test is green! So now what do we do? We need a way to relay any test failures from the message handling thread back to the main test thread. If a failure occurs in the message handling thread, we need it to be re-thrown in the main thread so that the test will fail as expected. Let's take a stab at this:

```java
CountdownLatch latch = new CountdownLatch(1);
AtomicReference<AssertionError> failure = new AtomicReference<>();
new Thread(() -> {
  try {
    assertTrue(false);
  } catch (AssertionError e) {
    failure.set(e);
  }
  latch.countDown();
}).start();

latch.await();
if (failure.get() != null)
  throw failure.get();
```

A quick run and yes, the test fails just as it should! Now we can go back and add CoundownLatches and try/catch blocks and AtomicReferences to all of our test cases. Awesome! Actually, not awesome, that sounds like a lot of work.

## Cut the cruft

Ideally what we want is an API that allows us to coordinate waiting, asserting, and resuming execution across threads, so that unit tests can be made to pass or fail as expected regardless of where the assertion failure occurs. Luckily, [ConcurrentUnit] provides a lightweight construct that does just this: the [Waiter]. Let's adapt our message handling test from above one last time and see what ConcurrentUnit's Waiter can do for us:

```java
String msg = "test";
Waiter waiter = new Waiter();
messageBus.registerHandler(message -> {
  waiter.assertEquals(message, msg);
  waiter.resume();
};

messageBus.publish(msg);
waiter.await(1, TimeUnit.SECONDS);
```

In this test, we can see that Waiter has taken the place of our `CountdownLatch` and`AtomicReference`. Through the Waiter we block the main test thread, perform our assertion, then resume the main test thread so that the test can complete. If the assertion were to fail, the `waiter.await` call automatically unblocks and throws the failure, causing our test to pass or fail as it should, even when asserting from another thread.

## More parallel

Now that we're certified multi-threaded testers, we may want to assert that multiple asynchronous actions occur. ConcurrentUnit's waiter makes this straightforward:

```java
Waiter waiter = new Waiter();
messageBus.registerHandler(message -> {
  waiter.resume();
};

messageBus.publish("one");
messageBus.publish("two");
waiter.await(1, TimeUnit.SECONDS, 2);
```

Here we publish two messages to the bus and verify that both messages are delivered by making the Waiter wait until `resume()` is called 2 times. If the messages fail to be delivered and resume is not called twice within 1 second then the test will fail with a `TimeoutException`.

One general tip with this approach is to make sure that your timeouts are reasonably long enough for any concurrent actions to complete. Under normal conditions when the system under test operates as expected, the timeout should not matter, and only come into play when the system fails for some reason.

## Recap

In this article we learned that multi-threaded unit testing is not evil and is fairly easy to do. We learned about a general approach where we block the main test thread, perform assertions from some other threads, then resuming the main thread. And we learned about [ConcurrentUnit] which can help make this easier to do.

Now go write some tests!

[akka]: http://akka.io/
[netty]: http://netty.io/
[concurrentunit]: https://github.com/jhalterman/concurrentunit
[jsr-166]: https://jcp.org/en/jsr/detail?id=166
[jsr-166-testing]: http://gee.cs.oswego.edu/cgi-bin/viewcvs.cgi/jsr166/src/test/tck/
[vertx-testing]: http://vertx.io/docs/vertx-unit/java/
[akka-testing]: http://doc.akka.io/docs/akka/current/scala/testing.html
[waiter]: http://jodah.net/concurrentunit/javadoc/net/jodah/concurrentunit/Waiter.html
[java-util-concurrent]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html
[CountdownLatch]: https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html
[await]: http://jodah.net/concurrentunit/javadoc/net/jodah/concurrentunit/Waiter.html#await-long-java.util.concurrent.TimeUnit-
[resume]: http://jodah.net/concurrentunit/javadoc/net/jodah/concurrentunit/Waiter.html#resume--