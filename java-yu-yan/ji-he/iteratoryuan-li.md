## Iteratable

在Java中，容器可进行迭代操作，使用Iteratable接口来标识：

Iteratable中的接口如下：

```java
public interface Iterable<T> {

    //返回迭代器
    Iterator<T> iterator();

    //对每个元素执行action操作，提供了默认实现
    //@since 1.8
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    //返回Spliterator
    //@since 1.8
    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}
```

Collection接口继承了Iterable接口，这表明所有的Colection均支持迭代操作：

```java
public interface Collection<E> extends Iterable<E> {... ...}
```

foreach示例代码

```java
List<Integer> list = new ArrayList<>(Arrays.asList(1,3,5,7));
list.forEach(element ->{
    System.out.println(element);
});

//输出如下
1
3
5
7
```

# Iterator

Iterator，迭代器，用来遍历Collection中的元素。对于Collection的所有实现类而言，必须实现以下接口，即必须返回一个迭代器：

```java
Iterator<E> iterator();
```

Iterator中的接口如下：

```java
public interface Iterator<E> {

    //判断是否下一个元素
    boolean hasNext();

    //返回下一个元素
    E next();

    //移除元素
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    //对剩余元素按顺序执行某个操作
    @since 1.8
    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

主要原理

* 创建指向某个`Collection`的`Iterator`对象时，`Iterator`内部的游标指向的第一个元素。
* 调用`hasNext()`的时候，只是判断下一个元素的有无，并不移动游标。
* 调用`next()`的时候，返回游标指向的元素，之后向下移动游标，使其指向Collection中的下一个元素。
* 调用`remove()`，删除当前游标上次指向的元素

```java
List<Integer> list = new ArrayList<>(Arrays.asList(6, 2, 2, 6, 8));
Iterator<Integer> iterator = list.iterator();
while (iterator.hasNext()){
    Integer next = iterator.next();
    System.out.println(next);
    if(next == 2){
        iterator.remove();
    }
}
System.out.println(">>>>>>>>>after");
for (Integer integer : list){
    System.out.println(integer);
}

//输出如下
6
2
2
6
8
>>>>>>>>>after
6
6
8
```

### **ArrayList中的实现**

ArrayList中有两个Iterator的实现，一个Itr，另一个是增强的ListItr（提供了向前移动/遍历的功能，是ListIterator的实现），这里以Itr为例。

```java
public Iterator<E> iterator() {
    return new Itr();
}
```

```java
private class Itr implements Iterator<E> {
        int cursor;       // 返回的下一个元素的下标，这个下标就是ArrayList中的下标
        int lastRet = -1; // 上次返回的元素的的下标
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor != size;
        }

        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;//将游标指向下一个元素
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
            try {
                ArrayList.this.remove(lastRet);//这里会实际删除ArrayList中的元素，会改变ArrayList的实际长度
                cursor = lastRet;//所以，让游标前移一位，相当于cursor = cursor - 1；
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        //暂时忽略该方法
        public void forEachRemaining(Consumer<? super E> consumer) {
            ... ...        
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

可知，在ArrayList内部的Iterator实现中，内部维护了一个游标cursor，该cursor初始化为0，每调用一次next\(\)方法时，在返回cursor指向的元素之后，将curosr值加1；同时，使用变量lastRet来记录上次next\(\)方法返回的元素的下标。在调用remove\(\)方法后，由于实际数据量少了1个，所以将cursor指向lastRet，以便以便下次next\(\)调用时可以返回正确的结果。

同时，也可以发现：在实现`hasNext()/next()/remove()`等方法时，均是直接操作或依赖ArrayList底层的数据结构。

在Collection其他子类中的实现中，也是如此。由于Collection中各实现类底层结构不尽相同，遍历时所采用的方式也就不一样，所以每个实现类一般都会各个维护自身的Iterator实现。

## Map中的迭代

Map是键值对的集合。对于给定的一个map，我们可以：

* 使用map.keySet\(\) 方法来获取所有的key的集合，类型为Set&lt;K&gt;（key值不会重复）；
* 使用map.values\(\) 方法来获取所有的value的集合，类型为Collection&lt;V&gt;；
* 使用map.entrySet\(\)方法来获取所有键值对的集合，类型为Set&lt;Map.Entry&lt;K,V&gt;&gt;；

这几种Map的迭代方式，其实是降级到Collection再进行的。

下面是对Map进行遍历的一些常见用法：

```java
 //创建map
HashMap<String, String> hashMap = new HashMap();
hashMap.put("1", "one");
hashMap.put("2", "two");
hashMap.put("3", "three");

//使用map.keySet()的方式遍历
Set<String> keySet = hashMap.keySet();
for (String key : keySet){
    System.out.println(String.format("[key]:%s, [value]%s", key, hashMap.get(key)));
}

//使用map.values()的方式遍历：只能获取到values
Collection<String> values = hashMap.values();
for (String value : values){
    System.out.println(String.format("[value]%s", value));
}

//使用map.entrySet()的方式遍历
for (Map.Entry<String, String> entry : hashMap.entrySet()) {
    System.out.println(String.format("[key]:%s, [value]%s", entry.getKey(), entry.getValue()));
}
```

但是，在JDK1.8及之后，HashMap中多了一种遍历的方法，该方法可以更直观地看到迭代方式的实现直接依赖底层数据结构：

```java
public void forEach(BiConsumer<? super K, ? super V> action) {
    Node<K,V>[] tab;
    if (action == null)
        throw new NullPointerException();
    if (size > 0 && (tab = table) != null) {
        int mc = modCount;
        //外层循环为table数组
        for (int i = 0; i < tab.length; ++i) {
            //内层循环为每个table数组中的桶，即链表或红黑树
            for (Node<K,V> e = tab[i]; e != null; e = e.next)
                action.accept(e.key, e.value);
        }
        if (modCount != mc)
            throw new ConcurrentModificationException();
    }
}
```

而这种方式，使用时写法更为简明（借助lambda表达式）：

```java
//使用forEach的方式，写法更简单
hashMap.forEach((key, value)->{
    System.out.println(String.format("[key]:%s, [value]%s", key, value));
});
```



## 

## 参考

[迭代器iterator原理和设计模式](https://blog.csdn.net/hebixi/article/details/52075684)

