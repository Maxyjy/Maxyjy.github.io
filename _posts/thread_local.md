ThreadLocal是啥？
ThreadLocal<T> 是一个提供线程私有局部变量的类，<T> 泛型来表示 ThreadLocal 中提供什么类型的对象，不同线程访问同个ThreadLocal中存储的变量时，访问都是线程私有的变量，对其修改也不会影响其他线程的私有变量，我们可以粗略的理解为 Thread 中存了一个 Map<ThreadLocal,Value>，用 ThreadLocal 作为 Key 在 Map 中获取Value。

官方的示例很好的展示了其用法。

public static class ThreadId {

    // 下一个ThreadID的值
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // 利用 ThreadLocal 提供的静态方法创建一个带有初始化方法的 ThreadLocal
    private static final ThreadLocal<Integer> threadId = ThreadLocal.withInitial(() -> {
        // 初始化thread id，并自增1
        int id = nextId.getAndIncrement();
        System.out.println("init id=[" + id + "]");
        return id;
    });

    // 每条线程取id时，取的是nextId，取完后自增
    public static int get() {
        int id = threadId.get();
        System.out.println("get id=[" + id + "]");
        return id;
    }
}

public static void main(String[] args) {
    // 不同线程都通过 ThreadId 的 get() 方法来获取 threadId。
    new Thread(() -> System.out.println("on thread [" + ThreadId.get() + "]")).start();
    new Thread(() -> System.out.println("on thread [" + ThreadId.get() + "]")).start();
}
Android 中 Looper 就用了 ThreadLocal 来保存每个线程单独的 Looper。

public final class Looper {
    ...
    // sThreadLocal.get() will return null unless you've called prepare().
    @UnsupportedAppUsage
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    ...
}
在 Looper.prepare() 方法里使用了 sThreadLocal.set(new Looper()) 来初始化，并且限制了每个线程只能有一个Looper。

private static void prepare(boolean quitAllowed) {
    //如果Looper不为null，说明已经创建了线程独有的Looper
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    //初始化这条线程的Looper
    sThreadLocal.set(new Looper(quitAllowed));
}
ThreadLocal只有set()、get()、remove() 三个public方法。

set(T value)
set(T value) 方法，为当前 Thread 生成一个 ThreadLocalMap，用来保存 ThreadLocal 和 value 的关系。

public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当线程的 ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    // map不为null，将当前ThreadLocal作为key，储存泛型value，为null则创建map
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
void createMap(Thread t, T firstValue) {
    // 为thread创建一个ThreadLocalMap，并把第一个value put进去
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
get()
get方法则是拿到thread的ThreadLocalMap，将自己作为key获取value。

public T get() {
    Thread t = Thread.currentThread();
    // 获取Thread t的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    // map不为null时取值
    if (map != null) {
        // map 保存的是ThreadLocal与value的关系，所以传递this将自身作为key获取value
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            //返回结果
            return result;
        }
    }
    // map为null，需要对值进行初始化
    return setInitialValue();
}
remove()
移除ThreadLocal对应的Value

public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
withInitial(Supplier<? extends S> supplier)
ThreadLocal 还提供了一个静态方法，用来创建一个带默认值提供者 Supplier 的 ThreadLocal。

public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
References:

Java 并发 – ThreadLocal详解 https://pdai.tech/md/java/thread/java-thread-x-threadlocal.html
