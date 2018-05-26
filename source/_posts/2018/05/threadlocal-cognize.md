---
title: threadlocal-cognize
date: 2018-05-26 22:09:30
tags: [ThreadLocal]
categories: [multi-thread]
---



### ThreadLocal关键字

线程局部变量，只有当前线程可以访问。既然只有当前线程可以访问，自然是线程安全的

> ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread(ThreadLocal实例通常是一个类中的私有的属性`private static`,并且期望状态和线程相关联)



#### ThreadLocal生命周期

在ThreadLocal的API中指出：  

> Each thread holds an implicit reference to its copy of a thread-local variable as long as the thread is alive and the ThreadLocal instance is accessible; after a thread goes away, all of its copies of thread-local instances are subject to garbage collection (unless other references to these copies exist).
> (只要当前线程存活并且ThreadLocal实例可以访问，每个线程都保存对其线程局部变量副本的隐式引用。线程消失后，线程本地实例的所有副本都将被垃圾收集（除非存在对这些副本的其他引用）)

<!-- more -->

#### ThreadLocal内部实现

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

在ThreadLocal的set方法中，先获取当前线程对象，`getMap`方法通过当前线程获取`ThreadLocalMap`对象[可以当成是类似是map]，如果map不为空，则将当前`threadlocal`实例作为key设置到`ThreadLocalMap`中,不然先创建`ThreadLocalMap`,然后设值

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}


private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

在ThreadLocal的get方法中，先获取当前线程对象，`getMap`方法通过当前线程获取`ThreadLocalMap`对象[可以当成是类似是map]，如果map不为空,则将当前`threadlocal`实例作为key值从map中取出value并返回。如果map没能创建，则调用`setInitialValue`方法，步骤与`set`方法差不多

那么`ThreadLocalMap`是什么呢？

```java
static class ThreadLocalMap {

       static class Entry extends WeakReference<ThreadLocal<?>> {
           /** The value associated with this ThreadLocal. */
           Object value;

           Entry(ThreadLocal<?> k, Object v) {
               super(k);
               value = v;
           }
       }

       private static final int INITIAL_CAPACITY = 16;

       private Entry[] table;
}
```

可以看到，`ThreadLocalMap`中包含一个table数组，数组中装有`Entry`对象，其中`Entry`是一个类似map的key-value存储的对象，以`threadlocal`实例为key。现在可以明白，一个线程中绑定了一个`ThreadLocalMap`，而`ThreadLocalMap`中包含多个以`threadlocal`实例为key的Entry对象

ThreadLocalMap中的set操作：

```java
private void set(ThreadLocal<?> key, Object value) {

   Entry[] tab = table;
   int len = tab.length;
   // 找到要存储的索引位置
   int i = key.threadLocalHashCode & (len-1);

   // 如果索引位置不为空null(已经存在Entry对象)
   for (Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {
       ThreadLocal<?> k = e.get();

       // 如果 当前索引的key所代表的是同一个threadlocal实例
       // 则用新值替换
       if (k == key) {
           e.value = value;
           return;
       }

       // 如果threadlocal实例已经被回收了即 null ---> value
       // 就用新值new Entry(key, value)替换原来的旧slot中的Entry对象
       if (k == null) {
           replaceStaleEntry(key, value, i);
           return;
       }
   }

   // 如果索引位置没有Entry对象，则插入新增
   tab[i] = new Entry(key, value);
   int sz = ++size;

   // 清除陈旧的Entry后，并且size 超过了阀值
   if (!cleanSomeSlots(i, sz) && sz >= threshold)
       // 尝试移除陈旧的(无效)Entry,如果size 还是超过了阀值，就调整table的大小(扩容)
       rehash();
}
```

ThreadLocalMap中的getEntr方法

```java
private Entry getEntry(ThreadLocal<?> key) {
   // 根据key计算在table中的索引位置
   int i = key.threadLocalHashCode & (table.length - 1);
   Entry e = table[i];
   // 直接命中
   if (e != null && e.get() == key)
       return e;
   else
       // 非直接命中(hashcode冲突)，遍历整个table当且仅当 key==e.get()
       return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
   Entry[] tab = table;
   int len = tab.length;

   while (e != null) {
       ThreadLocal<?> k = e.get();
       if (k == key)
           return e;
       if (k == null)
           expungeStaleEntry(i);
       else
           i = nextIndex(i, len);
       e = tab[i];
   }
   return null;
}
```

ThreadLocalMap中的remove方法:  

```java
// 根据key删除Entry对象
private void remove(ThreadLocal<?> key) {
   Entry[] tab = table;
   int len = tab.length;
   int i = key.threadLocalHashCode & (len-1);
   for (Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {
       if (e.get() == key) {
           e.clear();
           expungeStaleEntry(i);
           return;
       }
   }
}
```


回头再看Entry的定义:

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
   /** The value associated with this ThreadLocal. */
   Object value;

   Entry(ThreadLocal<?> k, Object v) {
       super(k);
       value = v;
   }
}
```

Entry继承了`WeakReference`(弱引用)，那什么是`WeakReference`？在系统GC的时候，只要发现是弱引用，不管堆空间是否足够，都会将对象进行回收，即当Entry中的key变成null(threadLocal实例被回收),那么GC也就会立即将这个key值为null的Entry回收，防止内存泄露，这就是key值不使用强引用的原因


**可以看到在`ThreadLocalMap`中的 set()，get() 和 remove()中都有清除key为null值的操作，说白就是为了避免内存泄露**



由于ThreadLocal生命周期的特性，如果使用不当，将会出现**内存泄露**的问题

**内存泄露(Memory Leak)**

> A memory leak in Java is amount of memory hold by object which are not in use and should have been garbage collected, but because of unintended strong references, they still live in Java heap space(Java中的内存泄露是指一些本该被垃圾收集器回收并且占有很大内存的对象，因为一些意外强引用`strong references`,致使它们仍旧存活在Java的堆空间中)


ThreadLoacl发生内存泄露的场景

ThreadLocal的生命周期随着当前线程创建而创建，销毁而销毁的。特别是在使用线程池时(固定数量线程池)，线程可能会被复用，线程会一直持有`ThreadLocalMap`的引用，如果在ThreadLocalMap中已经存储了一个Entry(tl_instance, value)的对象，如果以key为`tl_instance`的Entry在使用之后中并没有被及时的移除，并且value引用是一个占用内存比较大的强引用对象，又假设可能这个`tl_instance`很长时间都不会被用到(名存实亡)，那么就会造成内存泄露，如果及时将不用的`tl_instance`设成null值，由于Entry中key是弱引用的特性，就会回收长时间不使用key值为`tl_instance`的Entry,并且将value的引用对象也一并回收，保证内存不泄露


安全使用ThreadLocal:每次调用threadloal变量时，把它放在`try..finally`代码块中，并在在finally中调用threadlocal中的remove方法，如:

```java
try{
  sdf = threadlocal.get();
}finally{
  threadlocal.remove();
}
```


#### 使用ThreadLocal


都知道SimpleDateFormat在多线程环境下，在不使用任何同步机制的时候是线程不安全的


```java
public class ThreadLocalSdf {



    private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

    private static final ThreadLocal<SimpleDateFormat> sdfl = new ThreadLocal<>();

    //线程不安全
    public static void commonSdf() throws InterruptedException {
        Thread[] threads = new Thread[20];
        final CountDownLatch startLatch = new CountDownLatch(1);
        for (int i=0; i < 20; i++){
            threads[i] = new Thread(){

                @Override
                public void run() {
                    try {
                        startLatch.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    try {
                        Date date = sdf.parse("2018-03-28 12:23:"+(new Random().nextInt()%60));
                        System.out.println(sdf.format(date));
                    } catch (ParseException e) {
                        e.printStackTrace();e.printStackTrace();
                    }
                }
            };
            threads[i].start();
        }
        Thread.sleep(200);
        startLatch.countDown();

        for (Thread thread: threads){
            thread.join();
        }
    }

    // 使用threadlocal构建线程安全的SimpleDateFormat
    public static void threadlcSdf() throws InterruptedException {
        Thread[] threads = new Thread[20];
        final CountDownLatch startLatch = new CountDownLatch(1);
        for (int i=0; i < 20; i++){
            threads[i] = new Thread(){

                @Override
                public void run() {
                    try {
                        startLatch.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    try {
                        if (sdfl.get() == null){
                            sdfl.set(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
                        }

                        Date date = sdfl.get().parse("2018-03-28 12:23:"+(new Random().nextInt()%60));
                        System.out.println(sdfl.get().format(date));
                    } catch (ParseException e) {
                        e.printStackTrace();e.printStackTrace();
                    }
                }
            };
            threads[i].start();
        }
        Thread.sleep(200);
        startLatch.countDown();

        for (Thread thread: threads){
            thread.join();
        }

    }



    public static void main(String[] args) throws InterruptedException {

        commonSdf();
//        threadlcSdf();
    }

}
```

运行`commonSdf`方法,会出现以下异常:   

```console
java.lang.NumberFormatException: For input string: ""
	at java.lang.NumberFormatException.forInputString(NumberFormatException.java:65)
	at java.lang.Long.parseLong(Long.java:601)
	at java.lang.Long.parseLong(Long.java:631)
	at java.text.DigitList.getLong(DigitList.java:195)
	at java.text.DecimalFormat.parse(DecimalFormat.java:2051)
```
