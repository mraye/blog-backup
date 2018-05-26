---
title: java中重写equals一定要重写hashCode方法
date: 2018-05-26 22:00:08
tags: [hashCode]
categories: [java]
---



### java中重写equals一定要重写hashCode方法

原作者博客地址: [Working With hashcode() and equals()](https://dzone.com/articles/working-with-hashcode-and-equals-in-java)

#### equals和hashCode方法

+ equals(Object obj): 由Object类提供，用来判断当前对象和obj是否相等。JDK中默认实现方式是基于内存地址：只有两个对象的内存地址相等，那么它们才相等
+ hashCode(): 该方法由Object类提供，返回一个整型数值代表了对象在内存的地址，对于每一个对象，hashCode()方法随机返回一个唯一的整型的数值

<!-- more -->
#### 使用规则

+ **如果x.equals(y)==true,那么x.hashCode()==y.hashCode()也必须返回true**

也就是说，如果重写了类中的equals方法，那么必须重写hashCode方法


实体类`Ball.java`:

```java
public class Ball {

    private int id;
    private String color;

    public Ball(int id, String color) {
        this.id = id;
        this.color = color;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }
}

```

##### 情景1: 直接使用Object默认实现的equals和hashCode方法

测试:  

```java
@Test
public void commonEquals(){
    Ball ball1 = new Ball(1, "red");
    Ball  ball2 = new Ball(1, "red");
    System.out.println("ball1 hashcode: "+ ball1.hashCode());
    System.out.println("ball2 hashcode: "+ ball2.hashCode());
    System.out.println("ball1.equals(ball2): "+ ball1.equals(ball2));
}
```

结果:  

```console
ball1 hashcode: 26947503
ball2 hashcode: 22527820
ball1.equals(ball2): false
```

**很明显，不同的对象，hashcode不相等，ball1和ball2自然不相等**


##### 情景2: 重写equals方法，但不重写hashCode方法

重写equals方法:  

```java
@Override
public boolean equals(Object obj){
    if (this == obj)
        return true;
    if (obj == null || getClass() != obj.getClass())
        return false;
    Ball that = (Ball)obj;
    return id==that.getId();
}
```

结果:  

```console
ball1 hashcode: 30430942
ball2 hashcode: 16191201
ball1.equals(ball2): true
```

在equals方法中，我们使用id作为两个对象相等的条件

*如果在ArrayList中使用只重写equals之后的Ball*

```java
@Test
public void equalsWithArrayList(){
    ArrayList<Ball> arrayList = new ArrayList<>();
    Ball ball1 = new Ball(1, "red");
    Ball  ball2 = new Ball(1, "red");
    arrayList.add(ball1);
    arrayList.add(ball2);
    System.out.println("arrayList size: "+ arrayList.size());
    System.out.println("arrayList contains Ball: "+ arrayList.contains(new Ball(1,"red")));
}
```

好像并没有发现什么， ArrayList中本来就允许重复元素

*如果在HashMap中使用只重写之后的Ball*

```java
@Test
public void equalsWithHashMap(){
    HashMap<Ball, Integer> map = new HashMap<>();
    Ball ball1 = new Ball(1, "red");
    Ball  ball2 = new Ball(1, "red");
    map.put(ball1, 1);
    map.put(ball2,1);
    System.out.println("map size: "+ map.size());
    System.out.println("map contains Ball: "+ map.get(new Ball(1,"red")));
}
```

结果:  

```console
map size: 2
map contains Ball: null
```

讲道理，map的size应该是1，而且`map.get(new Ball(1, "red"))`应该返回1，什么情况？找到HashMap的put源码:  

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}
```
好吧，HashMap的put操作时，先是比较hashCode值是否相等，然后比较equals方法，如果hashCode不相等，则认为是addEntry操作。实际上，Ball没有重写hashCode方法,第一new的操作，都会在内存中分配不同的地址，所以map.size=2。

再看，`map.get()`源码方法：  

```java
//HashMap.get()源码
public V get(Object key) {
   if (key == null)
       return getForNullKey();
   Entry<K,V> entry = getEntry(key);

   return null == entry ? null : entry.getValue();
}

final Entry<K,V> getEntry(Object key) {
  if (size == 0) {
      return null;
  }

  int hash = (key == null) ? 0 : hash(key);
  for (Entry<K,V> e = table[indexFor(hash, table.length)];
       e != null;
       e = e.next) {
      Object k;
      if (e.hash == hash &&
          ((k = e.key) == key || (key != null && key.equals(k))))
          return e;
  }
  return null;
}
```

可以看到，取值的时候先计算对象的hashCode值，`map.get(new Ball(1,"red"))`,匿名new Ball(1,"red"),之前没有重写hashCode方法，又会产生新的hashCode，所以取出的是null


#### 重写hashCode方法

```java
@Override
public int hashCode() {
    return id;
}
```

再次运行`equalsWithHashMap`测试用例,结果:  

```console
map size: 1
map contains Ball: 1
```

可以看到，这次map的size为1，表明加入的ball1和ball2为同一实例

*同样也适用于HashSet,HashTable,或者其他以hashCode机制作为存储的数据结构*


**结论:在重写了equals方法后，强制要求重写hashCode方法**

+ 如果两个对象相等，那么他们必须有相同的hashCode
+ 如果两个对象有相同的hashCode,并不意味着他们相等
+ 单独重写equals方法会让业务中使用哈希数据结构的数据失效，如HashMap,HashSet,HashTable...
