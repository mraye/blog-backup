---
title: InputStream不能重用问题
date: 2018-05-26 21:43:52
tags: [inputstream]
categories: [java]
---



**问题：使用InpustStream读取文件内容，并且使用同一个InputStream进行对象反序列化的时候**

```java

@Test
public void readFromFile(){

    File file = new File("person.data");
    try (InputStream is = new FileInputStream(file)) {
        byte[] bytes = new byte[1024];
        int len = 0;
        StringBuilder sb = new StringBuilder();
        while ((len=is.read(bytes))!=-1){
            sb.append(new String(bytes, "UTF-8"));
        }
        System.out.println(sb.toString());
        ObjectInputStream ois = new ObjectInputStream(is);
        Person person = (Person) ois.readObject();
        System.out.println(person);
    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}

```

<!-- more -->

结果:  

```console

java.io.EOFException
	at java.io.ObjectInputStream$PeekInputStream.readFully(ObjectInputStream.java:2325)
	at java.io.ObjectInputStream$BlockDataInputStream.readShort(ObjectInputStream.java:2794)
	at java.io.ObjectInputStream.readStreamHeader(ObjectInputStream.java:801)
	at java.io.ObjectInputStream.<init>(ObjectInputStream.java:299)
```


**原来是InputStream在进行一次读取之后不能被重复使用**

***解决方案***

+ 重新打开一个InputStream
+ 将InputStream流中读取的字节写入ByteArrayOutputStream中暂存，用ByteArrayOutputStream可以重新构建多个InputStream达到重复使用的效果


**使用ByteArrayOutputStream暂存方法**

```java
@Test
public void readFromFile(){
    File file = new File("person.data");
    try (InputStream is = new FileInputStream(file)) {
        byte[] bytes = new byte[1024];
        int len = 0;
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        while ((len=is.read(bytes))!=-1){
            baos.write(bytes,0, len);
        }
        System.out.println(new String(baos.toByteArray(), "UTF-8"));
        InputStream reuseIs = new ByteArrayInputStream(baos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(reuseIs);
        Person person = (Person) ois.readObject();
        System.out.println(person);

    } catch (FileNotFoundException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

结果:  

```console
//omit file content...
Person{name='null', age=0}
```
