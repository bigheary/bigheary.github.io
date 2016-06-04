---
 layout: post
 title: "Java Learning"
---

我们通常说，在Java里面起多线程有两种方法：直接继承Thread类或者是实现Runnable接口。但是这两种方法都不能有直接的返回值，因为run方法是void类型，也没有提供其他的可以返回某类型的方法，我们可以通过线程间通信或者修改共享变量的方法来获取线程执行后的结果。
　　要直接实现返回变量类型，我们可以使用Callable接口

```
public interface Callable<V> {
    V call() throws Exception;
}
```
　　该接口只声明了一个方法，可以返回一个带有泛型的类型V。在使用是可以将任务写在call里面，并返回相应的值。
　　这种方式的线程必须采用ExecutorService接口中的submit方法提交任务，并且将返回一个Future&lt;V&gt;，用于获取最终的结果。

　　Future提供一个泛型接口
```
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
　　除了get用于获取线程执行的结果外，还提供了控制线程的方法`cancel`以及查询线程工作状态的方法等。从中可以看出，实现时应该是和具体线程紧耦合的。

　　上面是应用层面上的几个点。下面我们想要搞清楚几个问题：Runnable和Callable本质上到底有哪些区别；它们起线程的方式为什么不一样，为什么Callable不能像Runnable那样通过Thread类来起一个线程；还有Future是怎么获取线程运行的结果的。

### **1. Runnable和Callable**
　　Runnable和Callable都是两个接口，其中前者只声明了一个void run方法，后者声明了一个V call方法。我们将任务写在run或者call中。仅仅从这里还看不出有何联系，只能说相似。
　　通过Runnable起线程时，我们都是通过将Runnable传给一个Thread的构造器，最终通过Thread类来起线程的：

```
    public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
        init(group, target, name, stackSize);
    }
```
　　这是Thread构造器中完整的一个版本，其实是调用了一个init方法：

```
    private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
	...
	}
```
　　实现代码省略了，其中target是我们传进来的一个Runnable的实现，将赋值给Thread类里面的一个内部的Runnable target。以上简而言之就是完成了一个开线程的初始化工作，比如配置线程栈空间大小等。
　　起线程时我们还要调用Thread类中的start方法，这其实是起线程的核心：

```
    public synchronized void start() {
        if (threadStatus != 0)
            throw new IllegalThreadStateException();
        group.add(this);
        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {

            }
        }
    }
```
　　可以看到，其中最核心的就是调用了一个start0()方法，这是一个native的方法，也就是用c或者c++实现的部分，我们看不到源码，我想里面应该会调用target的run方法。

　　Thread类里面的run方法：

```
    public void run() {
        if (target != null) {
            target.run();
        }
    }
```
　　从这里我们可以推知，start0()调用Thread的run方法，实际上是调用的我们传入的Runnable的run方法。
　　在采用直接继承Thread的时候，因为我们覆盖了run方法，所以调用的是我们写的run方法。所以可以看到，这两种方式的调用本质上一样的。

- 那Callable起线程的方式和上述方式有何区别呢
因为Callable起线程要依赖于ExecutorService接口的submit方法，AbstractExecutorService实现了ExecutorService的上述方法：

```
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
```
　　三种形式我们可以看到，不管传入的参数是Runnable还是Callable，都会被包装成一个RunnableFuture，这是一个接口，`public class FutureTask<V> implements RunnableFuture<V>`
而`public interface RunnableFuture<V> extends Runnable, Future<V>`,注意这里接口允许多重继承。
　　现在我们需要关注的是，对于Runnable和Callable分别是如何被包装成RunnableFuture的，因为此后他们被execute调用的方式都一样，execute都将他们当作一个Runnable来看待。

　　我们来看FutureTask构造器的不同重载版本：

```
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }

    public FutureTask(Runnable runnable, V result) {
        this.callable = Executors.callable(runnable, result);
        this.state = NEW;       // ensure visibility of callable
    }
```
　　我们看到，在FutureTask里面，我们最终都是构造了一个Callable类型的成员变量callable，对于Callable的参数直接赋值就好了，对于Runnable类型的参数，我们怎么办呢，答案是通过Executors.callable方法得到一个相应的Callable，怎么实现的呢，我们可以想到可以使用适配器。看源码：

```
//class Executors
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result);
    }
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
```
　　下面的是一个RunnableAdapter的实现，也就是将Runnable的run方法包装到Callable的call里面去了，因为两者的行为功能是一致的，都是完成用户的任务。

　　至此我们总结一下，Callable在submit中是被包装成一个RunnableFuture，然后再提交给execute执行的。execute将传入的参数看成是一个Runnable，将会调用其中的run方法，在FutureTask中的run方法：

```
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
　　这里我们也看到，FutureTask作为一个Runnable，其核心实现的run方法最终是调用了其成员变量callable的call方法，而这个callable成员变量是通过FutureTask的构造器传进来的，可能是Runnable也可能是Callable，如果是Runnable还要适配成Callable。

　　通过以上介绍，应该能够比较清楚的了解了Callable的执行逻辑，还有Runnable采用ExecutorService.submit方式执行的逻辑了。

### **1. 关于Future**
　　Future可能是和线程耦合的，我们想来了解一下Future和线程耦合的一些细节：
它是一个接口，在jdk中的唯一实现就是前面提到的FutureTask，我们来看Future的主要方法在FutureTask中的实现：

```
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```
　　这是阻塞读取结果的方法实现，可以看到调用了awaitDone方法：
```
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }
```
　　具体细节还看不太懂，大概就是不断地去查询是否执行完毕，此外，我们还想要了解这个Future是怎么得知线程执行的结果的，看返回值那句`report(s)`:

```
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
```
　　就是直接返回一个成员变量outcome，我们看是哪里将结果赋给outcome的：

```
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
```
　　是这里的set方法，再看哪里调用了set方法，发现是在FutureTask中的run方法里面，代码前面出现过：
```
    public void run() {
        if (state != NEW ||
            !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                         null, Thread.currentThread()))
            return;
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
```
　　总结一下，也就是说FutureTask里面的实现是这样的：get方法查询线程执行状态，当得知运行完毕后直接去一个outcome变量取结果(report()方法)。这里的结果是run方法在运行到后面set进去的。也就是说Future的get方式得以实现是因为FutureTask本身就是一个Future也是Runnable，所以可以控制、获知线程的执行状态，必要是可以获取运行的结果。
以上。

