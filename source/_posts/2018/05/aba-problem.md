---
title: CAS(Compare And Swap)导致的ABA问题
date: 2018-05-26 22:04:41
tags: [ABA]
categories: [multi-thread]
---


问题描述

> 多线程情况下，每个线程使用CAS操作欲将数据A修改成B，当然我们只希望只有一个线程能够正确的修改数据，并且只修改一次。当并发的时候，其中一个线程已经将A成功的改成了B，但是在线程并发调度过程中尚未被调度，在这个期间，另外一个线程(不在并发中的请求线程)将B又修改成了A，那么原来并发中的线程又可以通过CAS操作将A改成B


测试用例:  

<!-- more -->

```java
public class AbaPro {

    private static final Random RANDOM = new Random();
    private static final String B = "B";
    private static final String A = "A";
    public static final AtomicReference<String> ATOMIC_REFERENCE = new AtomicReference<>(A);


    public static void main(String[] args) throws InterruptedException {
        final CountDownLatch startLatch = new CountDownLatch(1);

        Thread[] threads = new Thread[20];
        for (int i=0; i < 20; i++){
            threads[i] = new Thread(){
                @Override
                public void run() {
                    String oldValue = ATOMIC_REFERENCE.get();
                    try {
                        startLatch.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    try {
                        Thread.sleep(RANDOM.nextInt()&500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }

                    if (ATOMIC_REFERENCE.compareAndSet(oldValue, B )){
                        System.out.println(Thread.currentThread().getName()+ " 已经对原始值进行了修改,此时值为: "+ ATOMIC_REFERENCE.get());
                    }
                }
            };
            threads[i].start();
        }

        startLatch.countDown();
        Thread.sleep(200);

        new Thread(){

            @Override
            public void run() {
                try {
                    Thread.sleep(RANDOM.nextInt() & 200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                String oldVal = ATOMIC_REFERENCE.get();
                while (!ATOMIC_REFERENCE.compareAndSet(ATOMIC_REFERENCE.get(), A));
                System.out.println(Thread.currentThread().getName() +" 已经将值 "+oldVal+" 修改成原始值: A");
            }

        }.start();
    }

}

```


结果:  

```console
Thread-12 已经对原始值进行了修改,此时值为: B
Thread-20 已经将值 B 修改成原始值: A
Thread-14 已经对原始值进行了修改,此时值为: B
```

可以看到并发中的线程`Thread-12`已经成功的将A修改成B，其他线程`Thread-20`在某一时刻将B修改成A，而并发中的线程`Thread-14`又能再次成功的将A修改成B，虽然最终结果是B,但是中途经历了一次被修改的过程，在某些情况下是致使的



解决方案

java中提供了`AtomicStampedReference`来解决这个问题，它是基于版本或者是一种状态，在修改的过程中不仅对比值，也同时会对比版本号


```java
public class AabProResolve {

    private static final Random RANDOM = new Random();
    private static final String B = "B";
    private static final String A = "A";

    private static final AtomicStampedReference<String> ATOMIC_STAMPED_REFERENCE = new AtomicStampedReference<>(A,0);

    public static void main(String[] args) throws InterruptedException {

        final CountDownLatch startLatch = new CountDownLatch(1);
        Thread[] threads = new Thread[20];

        for (int i=0; i < 20; i++){
            threads[i] = new Thread(){

                @Override
                public void run() {
                    String oldValue = ATOMIC_STAMPED_REFERENCE.getReference();
                    int stamp = ATOMIC_STAMPED_REFERENCE.getStamp();

                    try {
                        startLatch.await();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    try {
                        Thread.sleep(RANDOM.nextInt() & 500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    if (ATOMIC_STAMPED_REFERENCE.compareAndSet(oldValue, B, stamp, stamp+1)){
                        System.out.println(Thread.currentThread().getName()+ " 已经对原始值: "+oldValue+" 进行了修改,此时值为: "+ ATOMIC_STAMPED_REFERENCE.getReference());
                    }
                }
            };
            threads[i].start();
        }
        Thread.sleep(200);
        startLatch.countDown();

        new Thread(){
            @Override
            public void run() {

                try {
                    Thread.sleep(RANDOM.nextInt() & 200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                int stamp = ATOMIC_STAMPED_REFERENCE.getStamp();
                String oldVal = ATOMIC_STAMPED_REFERENCE.getReference();
                while (!ATOMIC_STAMPED_REFERENCE.compareAndSet(
                        B,
                        A,stamp, stamp+1)){
                    stamp = ATOMIC_STAMPED_REFERENCE.getStamp();
                }
                System.out.println(Thread.currentThread().getName() +" 已经将值 "+oldVal+" 修改成原始值: A");


            }
        }.start();

    }

}

```

结果:  

```console
Thread-1 已经对原始值: A 进行了修改,此时值为: B
Thread-20 已经将值 B 修改成原始值: A
```


可以看到并发期间的线程只有`Thread-1`对A进行了修改，保证了只有一个线程对数据的修改，短暂的并发时间之后的其他线程`Thread-20`对其修改自然也就没有影响
