# Scala Future Timeout Patterns
> 18/08/2014

Scala [Futures](http://docs.scala-lang.org/overviews/core/futures.html) provide an elegant way of handling asynchronous operations in Scala. In this article, I would like to focus on the Future timeout patterns; how can you enforce time constraints on Future operations. There are several ways how to deal with Future timeouts.

### Await.result

The most simple and obvious way is simply start blocking for some period of time and then check if the Future has already completed. Future object itself doesn't provide an option to block (as it is discouraged to do so), but there is helper method `Await.result` for this purpose. It takes two arguments: a future to block on and a timeout. If the Future doesn't complete before the timeout, a `TimeoutException`  is thrown.

```scala
def longRunningOperation(): Int = { Thread.sleep(1000); 1 }

test("A Future should be awaitable using Await.result()") {
    val f = Future { longRunningOperation() }
    Await.result(f, 1100.milliseconds) should be(1)
}
```

Note that even though some `Await.result` calls on a Future might fail, the Future is still running and once it completes, the subsequent `Await.result` will succeed:

```scala
test("A Future should continue running even after several Await.result() timeouts") {
    val f = Future { longRunningOperation() }
    intercept[TimeoutException] { Await.result(f, 300.milliseconds) }
    intercept[TimeoutException] { Await.result(f, 500.milliseconds) }
    whenReady(f, timeout(300.milliseconds)) { x => x should be(1) }
}
```

Be very careful about using the blocking `Await.result` calls as it is almost never what you should do. This approach is in fact worse than not creating the Future at all and execute the `longRunningOperation()` synchronously. When you await on a Future, you are actually consuming two threads; the first one is running the long running operation itself and the other one is waiting until the first one completes. Use `Await.result` with extreme caution only as the last resort as it can also cause unexpected deadlocks.

### Futures.firstCompletedOf

The Future companion object provides the `Future.firstCompletedOf` method with the following signature:

```scala
def firstCompletedOf[T](futures: TraversableOnce[Future[T]])(implicit executor: ExecutionContext): Future[T]
```

It takes a collection of Futures as an argument and returns a new Future, which is completed once the first Future from the list completes. Therefore, knowing this method, we can write a custom Future, let's call it the `timeoutFuture`, which would block the thread for some period of time (the timeout) and then throw a `TimeoutException`. Such `timeoutFuture` can then be passed along with the original Future to the `firstCompletedOf` method:

```scala
val e = new TimeoutException("Future timeout")

test("firstCompletedOf() should timeout when the first completed (blocking) Future throws an exception") {
    val f1 = Future { longRunningOperation() }
    val timeoutFuture = Future { Thread.sleep(500); throw e }
    val f = Future.firstCompletedOf(f1 :: timeoutFuture :: Nil)
    whenReady(f.failed, timeout(600.milliseconds)) { ex => ex shouldBe an[TimeoutException] }
}
```

Even though this solution works as expected, it is neither effective nor an elegant one. It shouldn't be necessary to block a thread with the `timeoutFuture` as the only thing it is doing is throwing an exception after some time. Thankfully, Akka comes with a solution called `akka.pattern.after`, which completes a Future after some initial delay:

```scala
test("firstCompletedOf() should timeout when the first completed (non-blocking) Future throws an exception") {
    val f1 = Future { longRunningOperation() }
    val timeoutFuture = akka.pattern.after(500.milliseconds, using = system.scheduler) { Future.failed(e) }
    val f = Future.firstCompletedOf(f1 :: timeoutFuture :: Nil)
    whenReady(f.failed, timeout(600.milliseconds)) { ex => ex shouldBe an[TimeoutException] }
}
```

Using `akka.pattern.after`, we create a new Future in a non-blocking way, which completes after some delay with a specified value (this value can of course be an exception). Unfortunately, this approach requires access to an Akka scheduler.

### Promise completion

When you take a look at the [source code](https://github.com/scala/scala/blob/v2.11.2/src/library/scala/concurrent/Future.scala#L505) of the `Future.firstCompletedOf` method, you can see it is implemented by completing a [Promise](http://docs.scala-lang.org/sips/completed/futures-promises.html).

Using Promise completion directly can be useful if you are unable to create a non-blocking `timeoutFuture` shown in the previous paragraph, possibly due to unavailable Akka scheduler. Promises, on the other hand, can be completed from anywhere, you can use any other scheduler you have at your disposal, for example the standard Java `SchedulerExecutorService`:

```scala
test("SchedulerExecutorService job should be able to complete a promise") {
    val scheduler = Executors.newScheduledThreadPool(1)
    val p = Promise[Int]()
    p tryCompleteWith Future { longRunningOperation() }
    val action = new Runnable {
    override def run(): Unit = p tryFailure e
    }
    scheduler.schedule(action, 500, MILLISECONDS)
    whenReady(p.future.failed, timeout(600.milliseconds)) { ex => ex shouldBe an[TimeoutException] }
}
```

### Akka ask pattern implicit timeout

This one is not a general pattern as it is only applicable in Akka actors. In Akka, there is a pattern called the [ask pattern](http://doc.akka.io/docs/akka/current/scala/futures.html#Use_With_Actors), which provides a technique how to represent another actor's response as a Future:

```scala
test("Actor ask is constrained by an implicit timeout") {
    import akka.pattern.ask
    val actor = system.actorOf(Props(new Actor { override def receive: Receive = PartialFunction.empty } ))
    implicit val _timeout = Timeout(500.milliseconds)
    val f = actor ? "PING"
    whenReady(f.failed, timeout(600.milliseconds)) { ex => ex shouldBe an[TimeoutException] }
}
```

Here, we import `akka.pattern.ask`, then create a new actor, which simply doesn't do anything as its `receive` is an empty partial function not defined for any input message and hence also not responding to any message. Finally, we ask the actor using the `?` method, which returns a Future representing the actor's response, which, in our case, is not going to happen.

The `?` method has the following signature:

```scala
def ?(message : scala.Any)(implicit timeout : akka.util.Timeout) : scala.concurrent.Future[scala.Any]
```

We can see an `akka.util.Timeout` object is required to be in the implicit scope. Therefore, it is not necessary to implement any kind of manual timeout handling using some of the methods described above. Akka makes sure we don't forget to provide an implicit timeout because if we do forget, the code won't compile.

### Interrupting the future operations

None of the approaches discussed above was dealing with interruption of the future operations themselves. Therefore, even though we were combining Futures in various ways and creating new ones representing the timeouts, the original Futures were still running and possibly occupying background threads.

In general, it is not possible (and not desireable) to terminate a Future unless it *cooperates* in some way. By cooperating, I mean the Future is for example periodically checking some termination condition and if that condition is satisfied, the future process terminates. Here is a simple example:

```scala
test("Future should check the termination flag and terminate itself") {
    @volatile var terminated = false
    val f = Future { while(!terminated) println("Running"); 1 }
    /* In 1 second, a newly created Future will set the terminated flag to true */
    akka.pattern.after(1.second, using = system.scheduler) { Future { terminated = true } }
    whenReady(f, timeout(1100.milliseconds)) { x => x should be(1) }
}
```

Here, we create a Future, which is printing a string in a loop to the standard output as long as the `terminated` flag is `false`. Once the flag is set to `true`, the loop terminates and the Future is completed with a result. The second Future, created using the `akka.pattern.after` discussed above, after 1 second sets the `terminated` flag to `true`, which effectively terminates the first Future. The `whenReady` block verifies the Future has really been completed with the expected result.

Also note the `@volatile` annotation on the local variable `terminated`. This code snippet might be a bit surprising for Java developers because in Java the `volatile` modifier is not allowed on local variables and thus this code wouldn't compile. However, unlike Java, Scala local variables might be closed over and modified by closures and thus it is sometimes necessary to mark them as volatile, as well. In older versions of Scala, there was a [bug](https://issues.scala-lang.org/browse/SI-2424), which caused local variables not being marked as `volatile` properly. The bug is already fixed.

An alternative to `@volatile` variables might be using the [AtomicBoolean](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/atomic/AtomicBoolean.html) from the Java SE or synchronize every access to the `terminated` variable.

### Conclusion

We've seen various ways how to introduce a timeout on future operations. `Await.result()` was blocking for specified amount of time and threw an exception if the Future has not yet completed. Other solutions involved creating a new Future, which completed with an error after the specified timeout. Finally, we've also seen how to really terminate the *cooperating* Futures. You can find the full source code of the blogpost in its [GitHub repository](https://github.com/semberal/blog-examples/blob/master/scala-future-timeout-patterns/src/test/scala/eu/semberal/blog/futuretimeoutpatterns/FutureTimeoutsTest.scala).
