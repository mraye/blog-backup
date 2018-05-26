---
title: trycatchfinally的一些理解
date: 2018-05-26 21:56:23
tags: [try]
categories: [java]
---



### trycatchfinally的一些理解


#### 情景1:try-finally-with-return

测试用例：

```java
public int tfWithReturnTest(){
   try{
       System.out.println("enter try block...");
       return 1;
   }finally {
       System.out.println("enter finally block...");
//            return 2;
   }
}

//junit test case
@Test
public void tcfTest(){
    int rs  = tfWithReturnTest();
    System.out.println("result: "+ rs);
}
```

<!-- more -->

结果:

```console
enter try block...
enter finally block...
result: 1
```

打开finally代码块中的return语句,再次执行testCase:  

```console
enter try block...
enter finally block...
result: 2
```

**可以发现，无论有没有return,程序都会执行finally块中的代码，如果finally中没有return,程序执行完try中代码便把控制权转交给finally,处理完finally中扩逻辑之后又把控制权转交给try中的return返回最后的结果;如果finally中存在return,程序执行完try中代码便把控制权转交给finally,处理完finally逻辑之后就直接返回结果，不再走try中的return**

#### 情景2:try-finally-exception-with-return


测试用例:  
```java
public int tfExceptionWithReturnTest(){
   try{
       System.out.println("enter try block...");
       if (1==1) {
           throw new IllegalArgumentException("IllegalArgumentException in try block...");
       }
       return 1;
   }finally {
       System.out.println("enter finally block...");
       // return 2;
   }
}

//junit test case
@Test
public void tcfTest(){
   int rs = tfExceptionWithReturnTest();
   System.out.println("result: "+ rs);
}
```

结果:  

```console
enter try block...
enter finally block...
java.lang.IllegalArgumentException: IllegalArgumentException in try block...
```

打开finally代码块中的return语句,再次执行结果:  

```console
enter try block...
enter finally block...
result: 2
```

**可以发现，无论有没有return,程序都会执行finally块中的代码，如果finally中没有return,程序执行完try中代码将要抛出异常时，便把控制权转交给finally,处理完finally中扩逻辑之后又把控制权转交给try，让try中执行`throw exception`的操作;如果finally中存在return,程序执行完try中代码将要抛出异常时，便把控制权转交给finally,处理完finally逻辑之后就直接返回结果，不再让try中执行`throw exception`的操作**


#### 情景3:try-catch-finally-exception-with-return

```java
public int tcfWithReturnTest(){
    try{
        System.out.println("enter try block...");
//            if (1==1) {
//                throw new IllegalArgumentException("IllegalArgumentException in try block...");
//            }
        return 1;
    }catch (Throwable t){
        System.out.println("enter catch block...");
//            throw new  RuntimeException("throw runtimeException in catch block...");
        return 2;
    }finally {
        System.out.println("enter finally block...");
//            return 3;
    }
}

//junit test case
@Test
public void tcfTest(){
//        int rs  = tfWithReturnTest();
//        int rs = tfExceptionWithReturnTest();
    int rs = tcfWithReturnTest();
    System.out.println("result: "+ rs);

}
```

1.正常运行结果:  

```console
enter try block...
enter finally block...
result: 1
```

2.打开finally代码块中的return语句：  

```java
public int tcfWithReturnTest(){
       try{
           System.out.println("enter try block...");
//            if (1==1) {
//                throw new IllegalArgumentException("IllegalArgumentException in try block...");
//            }
           return 1;
       }catch (Throwable t){
           System.out.println("enter catch block...");
//            throw new  RuntimeException("throw runtimeException in catch block...");
           return 2;
       }finally {
           System.out.println("enter finally block...");
           return 3;
       }
   }

```
结果:  
```console
enter try block...
enter finally block...
result: 3                   //直接在finally块中返回，而不是将控制权交给try,让try返回结果
```

3.打开try中的抛出异常代码:  

```java
public int tcfWithReturnTest(){
   try{
       System.out.println("enter try block...");
       if (1==1) {
           throw new IllegalArgumentException("IllegalArgumentException in try block...");
       }
       return 1;
   }catch (Throwable t){
       System.out.println("enter catch block...");
//            throw new  RuntimeException("throw runtimeException in catch block...");
       return 2;
   }finally {
       System.out.println("enter finally block...");
//            return 3;
   }
}
```

结果:  

```console
enter try block...
enter catch block...
enter finally block...
result: 2                   //返回的是catch中的值  
```

**可以看到，finally代码块无论如何都会被执行，在try中抛出异常之后，catch代码块中捕获异常，处理catch中的逻辑之后在抛出异常之前(或return之前)，将控制权交给finally块中，但finally中没有retun关键字，就又将控制权交给catch,由catch抛出异常(或直接return返回值)**


综上所述:

+ finally在try-catch-finally语句中一定会被执行
+ 如果finally中没有return关键字，程序会在执行finally代码块中的逻辑之后将控制权交给try或者是catch代码块;如果在try中抛出异常，先执行catch中逻辑,如果catch中return或者是throw Exception操作时，先将控制权交给finally,在finally中处理逻辑之后，将控制权交给catch,由catch执行throw Exception和return的操作
