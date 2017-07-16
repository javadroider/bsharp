# Chapter 1. Introduction

# Chapter 7. Cancellation and Shutdown

Java does not provide any mechanism for safely forcing a thread to stop what it is doing. Instead, it provides interruption. The cooperative approach is required because we reraly want a task to stop *immediately, *since that could leave shared data structures in inconsistent state.

One such cooperative mechanism is setting a "cancellation request” flag that the task checks periodically:

```java
...
private volatile boolean cancelled;

public void run() {
    BigInteger p = BigInteger.ONE;
    while (!cancelled) {
        p = p.nextProbablePrime();
        synchronized (this) {
            primes.add(p);
        }
    }
}

public void cancel() {
    cancelled = true;
}
...
```

The approach above would not work well if the task calls a blocking method (`BlockingQueue.put`) - the task might never terminate.
`Thread.interrupted()`- **caution!** returns the status and clears it!
Use `Thread.currentThread().isInterrupted()` instead

Blocking library methods like `Thread.sleep` and `Object.wait` try to detect when a thread has been interrupted and return early. They clear **the interrupted flag** and throw `InterruptedException`. JVM does not guarantee when it happens, but it usually happens quickly

If a thread is interrupted when it is not blocked, its interrupted status is set, and it is up to the activity being canceled to poll the interrupted status to detect interruption. Calling `interrupt` does not stop necessarily stop the target thread from what it's doing, but delivers the message that interrupted has been requested.

Instead of using a volatile flag:
```Java
…
while(!Thead.currentThread().isInterrupted()) {
 ...
}
…
public void cancel() {
 interrupt();
}
```

It is important to distinguish between how _tasks_ and _threads_ should react to interruption. A single interrupt request may have more than one desired recipient - interrupting a worker thread in a thread pool may can mean both "cancel the current task" and "shut down the worker thread".

Tasks do not execute threads they own - the borrow threads from the pool. Code that doesn't own the thread should be carefuly to preserve the interrupted status so that the owning code can eventually act on it.

Most blocking library methods simply throw InterruptedException - letting it bubble up to the thread owner as soon as possible. So task can wait to finish some job, then in the `catch` block - call `Thread.currentThread().interrupt()` to restore the flag.

**Because each thread has its own interruption policy, you should not interrupt a thread unless you know what interruption means to that thread.**

When you are calling interruptible blocking method, such as Thread.sleep or BlockingQueue.put, there are a few strategies:
* Propagate exception (add `throws InterruptedException`), making your method a interruptible blocking method - does not work for `Runnable`
* Restore the interruption status so that code higher up on the call stack can deal with it

Activities that do not support cancellation but still call interruptible blocking methods will have to call them in a loop - retrying when interruption is detected:
```Java
public Task getNextTask(BlockingQueue<Task> queue) {
    boolean interrupted = false;
    try {
        while (true) {
            try {
                return queue.take();
            } catch (InterruptedException e) {
                interrupted = true;
                // fall through and retry
            }
        }
    } finally {
        if (interrupted)
            Thread.currentThread().interrupt();
    }
}
```

If code does not call interruptible - poll current thread's interrupted status

Cancelling a task using Future:
```java
public class TimedRun {
    private static final ExecutorService taskExec = Executors.newCachedThreadPool();

    public static void timedRun(Runnable r,
                                long timeout, TimeUnit unit)
            throws InterruptedException {
        Future<?> task = taskExec.submit(r);
        try {
            task.get(timeout, unit);
        } catch (TimeoutException e) {
            // task will be cancelled below
        } catch (ExecutionException e) {
            // exception thrown in task; rethrow
            throw launderThrowable(e.getCause());
        } finally {
            // Harmless if task already completed
            task.cancel(true); // interrupt if running
        }
    }
}
```

If a thread is blocked performing synchronous socket I/O or waiting to acquire an intrinsic lock, interruption has no effect other than setting the interrupted status. How to handle:
* Synchronous I/O in java.io - `read` and `write` in `InputStream` and `OutputStream` are not responsive to interruption, but closing the underlying socket makes any thread blocked in `read` or `write` throw `SocketException`:
```java
public class ReaderThread extends Thread {
    private static final int BUFSZ = 512;
    private final Socket socket;
    private final InputStream in;

    public ReaderThread(Socket socket) throws IOException {
        this.socket = socket;
        this.in = socket.getInputStream();
    }

    public void interrupt() {
        try {
            socket.close();
        } catch (IOException ignored) {
        } finally {
            super.interrupt();
        }
    }

    public void run() {
        try {
            byte[] buf = new byte[BUFSZ];
            while (true) {
                int count = in.read(buf);
                if (count < 0)
                    break;
                else if (count > 0)
                    processBuffer(buf, count);
            }
        } catch (IOException e) { /* Allow thread to exit */
        }
    }

    public void processBuffer(byte[] buf, int count) {
    }
}
```
* Lock acquisition: there is nothing you can do, **unless explicit Lock classes are used, which offer `lockInterruptibly` method**

In the example below:
```java
synchronized(LogService.this) {
  if (isShutdown && writesPending == 0) {
    break;
  }
  String message = queue.take();
  synchronized(LogService.this) {
    writesPending--;
  }
}
```
We don't want to hold a lock when `queue.take()` is called, because `queue.take()` can block - resulting in other threads using the `LogService` to block

Another way to convince a producer-consumer service to shut down is with a poison pill. With FIFO queues, consumers ensure that consumers finish the work on their queue before shutting down; producers should not submit any work after the poison pill. Poison pill will work only when the number of consumers and producers is known. For `N` producers and 1 consumer - consumer stops when it receives `N` pills. For `M` consumers - each producer should put `M` poison pills. Also the queue should be **unbounded**.
Example: IndexingService:
```java
public class IndexingService {
    private static final int CAPACITY = 1000;
    private static final File POISON = new File("");
    private final IndexerThread consumer = new IndexerThread();
    private final CrawlerThread producer = new CrawlerThread();
    private final BlockingQueue<File> queue;
    private final FileFilter fileFilter;
    private final File root;

    public IndexingService(File root, final FileFilter fileFilter) {
        this.root = root;
        this.queue = new LinkedBlockingQueue<File>(CAPACITY);
        this.fileFilter = new FileFilter() {
            public boolean accept(File f) {
                return f.isDirectory() || fileFilter.accept(f);
            }
        };
    }

    private boolean alreadyIndexed(File f) {
        return false;
    }

    class CrawlerThread extends Thread {
        public void run() {
            try {
                crawl(root);
            } catch (InterruptedException e) { /* fall through */
            } finally {
                while (true) {
                    try {
                        queue.put(POISON);
                        break;
                    } catch (InterruptedException e1) { /* retry */
                    }
                }
            }
        }

        private void crawl(File root) throws InterruptedException {
            File[] entries = root.listFiles(fileFilter);
            if (entries != null) {
                for (File entry : entries) {
                    if (entry.isDirectory())
                        crawl(entry);
                    else if (!alreadyIndexed(entry))
                        queue.put(entry);
                }
            }
        }
    }

    class IndexerThread extends Thread {
        public void run() {
            try {
                while (true) {
                    File file = queue.take();
                    if (file == POISON)
                        break;
                    else
                        indexFile(file);
                }
            } catch (InterruptedException consumed) {
            }
        }

        public void indexFile(File file) {
            /*...*/
        };
    }

    public void start() {
        producer.start();
        consumer.start();
    }

    public void stop() {
        producer.interrupt();
    }

    public void awaitTermination() throws InterruptedException {
        consumer.join();
    }
}
```

If a method needs to run a batch of tasks and does not return until all of them finished - we can use a private Executor whose lifetime is bounded by that method:

```java
public boolean checkMail(Set<String> hosts, long timeout, TimeUnit unit)
    throws InterruptedException {
  ExecutorService exec = Executors.newCachedThreadPool();
  final AtomicBoolean hasNewMail = new AtomicBoolean(false);
  try {
    for (final String host : hosts)
      exec.execute(() -> {
        if (checkMail(host))
          hasNewMail.set(true);
      });
  } finally {
    exec.shutdown();
    exec.awaitTermination(timeout, unit);
  }
  return hasNewMail.get();
}
```

When tasks are shutdown in executor using shutdownNow: executor attempts to cancel the tasks currently in progress and **return a list of tasks that never started**. However, there is no general way to find out which tasks started, but did not complete:
```java
public class TrackingExecutor extends AbstractExecutorService {
  private final ExecutorService exec;
  private final Set<Runnable> tasksCancelledAtShutdown =
      Collections.synchronizedSet(new HashSet<Runnable>());

  public TrackingExecutor(ExecutorService exec) {
    this.exec = exec;
  }

  public List<Runnable> getCancelledTasks() {
    if (!exec.isTerminated())
      throw new IllegalStateException(/*...*/);
    return new ArrayList<Runnable>(tasksCancelledAtShutdown);
  }

  public void execute(final Runnable runnable) {
    exec.execute(() -> {
      try {
        runnable.run();
      } finally {
        if (isShutdown()
            && Thread.currentThread().isInterrupted())
          tasksCancelledAtShutdown.add(runnable);
      }
    });
  }
}

```

Web crawler using the TrackingExecutor from above:
```Java
public abstract class WebCrawler {
    private volatile TrackingExecutor exec;
    @GuardedBy("this") private final Set<URL> urlsToCrawl = new HashSet<URL>();

    private final ConcurrentMap<URL, Boolean> seen = new ConcurrentHashMap<URL, Boolean>();
    private static final long TIMEOUT = 500;
    private static final TimeUnit UNIT = MILLISECONDS;

    public WebCrawler(URL startUrl) {
        urlsToCrawl.add(startUrl);
    }

    public synchronized void start() {
        exec = new TrackingExecutor(Executors.newCachedThreadPool());
        for (URL url : urlsToCrawl) submitCrawlTask(url);
        urlsToCrawl.clear();
    }

    public synchronized void stop() throws InterruptedException {
        try {
            saveUncrawled(exec.shutdownNow());
            if (exec.awaitTermination(TIMEOUT, UNIT))
                saveUncrawled(exec.getCancelledTasks());
        } finally {
            exec = null;
        }
    }

    protected abstract List<URL> processPage(URL url);

    private void saveUncrawled(List<Runnable> uncrawled) {
        for (Runnable task : uncrawled)
            urlsToCrawl.add(((CrawlTask) task).getPage());
    }

    private void submitCrawlTask(URL u) {
        exec.execute(new CrawlTask(u));
    }

    private class CrawlTask implements Runnable {
        private final URL url;

        CrawlTask(URL url) {
            this.url = url;
        }

        private int count = 1;

        boolean alreadyCrawled() {
            return seen.putIfAbsent(url, true) != null;
        }

        void markUncrawled() {
            seen.remove(url);
            System.out.printf("marking %s uncrawled%n", url);
        }

        public void run() {
            for (URL link : processPage(url)) {
                if (Thread.currentThread().isInterrupted())
                    return;
                submitCrawlTask(link);
            }
        }

        public URL getPage() {
            return url;
        }
    }
}
```

**Handling abnormal thread termination**
The leading cause of premature thread death is RuntimeException. They are not caught and propagated all the way up the stack, at which point the default behavior is to print a stack trace on the console and let the thread terminate.

Whenever you call another method - you take a leap of faith that it will return normally or throws one of the checked exceptions its signature decalares.

**Shutdown hooks**
In orderly shutdown, JVM starts all shutdown hooks - which are also threads. The rest of the threads continue to work. Order of shutdown hooks is not guaranteed. When all shutdown hooks have completed, JVM can run finalizers (if enabled), and then halt. 
Shutdown hooks should be thread safe and should not make assumptions about the state of application.
Shutdown hooks can be used for service or application cleanup (deleting temp files, cleaning up resources).
In order not to rely on the same resource by multiple shutdown hooks - we can use a single shutdown hook and execute everything sequentially.

# Chapter 8 Applying thread pool
Single threaded pools make stronger promises about concurrency than do arbitrary thread pools. They guarantee that tasks are not executed concurrently, which allows you to relax the thread safety of task code.
`ThreadLocal` allows each thread to have its own private "version" of a variable, but be careful, because thread pools **reuse threads**, and use it only if thread-local value has a lifetime that is bounded by that of a task.

Thread pools work best when tasks are homogeneous and independent.
Mixing long-living and short-living tasks risks "clogging" the pool unless it is very large.
Tasks that depend on other tasks risks deadlock **unless the pool is unbounded**.

If tasks that depend on other tasks execute in a thread pool, they can deadlock.
In a single-threaded executor, a task that submits another task to the esame executor and waits for its result will always deadlock. This is called a **thread starvation deadlock.**

### Long running tasks
One technique that can mitigate the ill effects of long-running tasks is for tasks to use timed resource waits instead of unbounded waits. Thread.join, BlockingQueue.put, CountDownLatch.await, etc have a timed version of the method.

### Managing queued tasks
If arrival rate for new requests exceeds the rate at which they can be handled, requests will still be queued up. If tasks queue up quickly, you will eventually have to throttle the arrival rate to avoid running out of memory.

The default for `newFixedThreadPool` and `newSingleThreadExecutor` is to use an unbounded `LinkedBlockingQueue`; queue could grow without bound.

Alternative is to use `SynchronousQueue`, if rejection is accepted or the pool is unbounded. 

## Extending ThreadPoolExecutor
`ThreadPoolExecutor` was designed for extension, providing several "hooks" for subclasses to override - `beforeExecute`, `afterExecute`, and `terminated`.
`beforeExecute` and `afterExecute` are called in the thread that executes the task. `afterExecute` executes if task completed normally or threw an exception (but not `Error`). If `beforeExecute` throws a `RuntimeException`, the task is not executed and `afterExecute` is not called.

## Parallelizing recursive algorithms

```Java
 public <T> Collection<T> getParallelResults(List<Node<T>> nodes)
            throws InterruptedException {
        ExecutorService exec = Executors.newCachedThreadPool();
        Queue<T> resultQueue = new ConcurrentLinkedQueue<T>();
        parallelRecursive(exec, nodes, resultQueue);
        exec.shutdown(); //graceful, will wait for completion
        exec.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS); //will wait forever TODO Figure out why
        return resultQueue;
    }
```

`private final AtomicInteger taskCount = new AtomicInteger(0);` - incrementing when task is submitted, and decrementing when finished. This way if count = 0, it means that we can stop. **Good way to stop web crawler**

**Still, copy ConcurrentPuzzleSolver.java, Puzzle.java, PuzzleNode.java, PuzzleSolver.java, int a separate project, and play with it!!!**

# Avoiding liveness Hazards
## Deadlocks
When deadlocks do manifest themselves, it is often at the worst possible time - under heavy production load.
**A program will be free of lock-ordering deadlocks if all threads acquire the locks they need in a fixed global order.**

When obtaining multiple locks for objects, we can use System.identityHashCode(obj), and obtain locks (synchronized(obj1)...) in the order. In rare cases, they will be the same - so we can use another tiObject = new Object() as a lock.

You should always try to use open calls - i.e., calling method without holding a lock, because you never know what other methods are doing.

A program that never acquires more than one lock at a time cannot experience lock-ordering deadlock.

Another technique for detecting and recovering from deadlocks is to use the timed `tryLock` of the explicit `Lock` classes. **This is one of the reasons to use explicit locks**. Though when timed lock acquisition fails - we cannot know why: maybe the method was taking too long, or there was an infinite loop. 

## Starvation
Something related to priorities of threads. Avoid the temptation to use thread priorities, since they increase platform dependence.

## Livelock
Livelock is a form of liveness failure in which a thread, while not blocked, still cannot make progress because it keeps retrying an operation that always fails.
Example: when two overly polite people are walking in opposite direction: each steps out of the other's way, and now they're again in each other's way.
The solution to a variety of livelock is to **introduce some randomness** into the retry mechanism.

# Chapter 11. Performance and Scalability
If there are more runnable threads than CPUs, eventually the OS will preemt one thread so that another can use the CPU. This causes _context switch_, which requires saving the execution context of the currently running thread and restoring the execution context of the newly scheduled thread.

Context switches are not free; threads scheduling requires manipulating shared data structures in the OS and JVM. Furthermore, when a thread is switched in, the data it needs is unlikely to be in the **local processor cache**, so a context switch causes a flurry of cache misses, and thus the reasons that schedulers give each runnable thread a certain minimum time quantum even when many other threads are waiting: **it amortizes the cost of context switch over more interrupted execution time, improving overall throughput**.

Synchronization by one thread can also affect the performance of other threads. Synchronization creates traffic on the shared memory bus, which has limited bandwidth and is shared across all processors. If threads must compete for synchronization bandwidth, all threads using synchronization will suffer.

Two factors influence the likelihood of contention of a lock: **how often that lock is requested and how long it is held once acquired**.

An effective way of reducing lock contention is to hold it as briefly as possible, by **moving code that doesn't require the lock out of synchronized blocks**.

The other way to reduce the fraction of time that a lock is held is to have threads ask for it less often. If A and B are **independent** and are guarded by the same lock, they could be split to be held by two separate locks.

Lock stripping: concurrent hash map, has 16 locks for each of the 16 buckets..

Btw, CHM maintains a separate counter for each bucket, also guarded by the bucket lock, and each time `size()` is called, the sum of all counters is obtained. Might not be fully up to date because of that, but Java avoids **a hot field** this way.














