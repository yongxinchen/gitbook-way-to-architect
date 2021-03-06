Set：元素无序并且不允许重复元素

## java.util

**HashSet：数据结构为哈希表，元素无序、不重复，至多有一个null元素**

> 底层使用 HashMap 来保存所有元素，因此 HashSet 的实现比较简单，相关 HashSet 的操作，基本上都是直接调用底层 HashMap 的相关方法来完成。  
> 特点如下  
>  不能保证元素的排列顺序，顺序有可能发生变化  
>  不是同步的  
>  集合元素可以是null,但只能放入一个null  
> 当向HashSet结合中存入一个元素时，HashSet会调用该对象的hashCode\(\)方法来得到该对象的hashCode值，然后根据 hashCode值来决定该对象在HashSet中存储位置。  
> 简单的说，HashSet集合判断两个元素相等的标准是两个对象通过equals方法比较相等，并且两个对象的hashCode\(\)方法返回值相 等  
> 注意，如果要把一个对象放入HashSet中，重写该对象对应类的equals方法，也应该重写其hashCode\(\)方法。其规则是如果两个对 象通过equals方法比较返回true时，其hashCode也应该相同。另外，对象中用作equals比较标准的属性，都应该用来计算 hashCode的值。

**LinkedHashSet：数据结构是哈希表和链表，与HashSet相比访问更快，插入时性能稍微**

> LinkedHashSet继承自HashSet。同样是根据元素的hashCode值来决定元素的存储位置，但是它同时使用链表维护元素的次序。这样使得元素看起来像是以插入顺序保存的，也就是说，当遍历该集合时候，LinkedHashSet将会以元素的添加顺序访问集合的元素。  
> LinkedHashSet在迭代访问Set中的全部元素时，性能比HashSet好，但是插入时性能稍微逊色于HashSet。

**TreeSet：数据结构是二叉树\(红黑树\)，元素可排序、不重复**

> TreeSet可以确保集合元素处于排序状态。TreeSet支持两种排序方式：自然排序和定制排序，其中自然排序为默认的排序方式。向TreeSet中加入的应该是同一个类的对象。  
> TreeSet判断两个对象不相等的方式是两个对象通过equals方法返回false，或者通过CompareTo方法比较没有返回0
>
> 1、自然排序  
> （1）TreeSet内的元素实现`Comparable`接口，重写该接口的`compareTo(Object obj)`方法，以此确定排序。（元素必须实现该接口，否则程序会抛出异常）。  
> （2）当重写元素对应类的`equals()`方法时，应该保证该方法与`compareTo(Object obj)`方法有一致的结果，即如果两个对象通过`equals()`方法比较返回true时，这两个对象通过`compareTo(Object obj)`方法比较结果应该也为0\(即相等\)
>
> 2、定制排序  
> 自然排序是根据集合元素的大小，以升序排列，如果要定制排序，应该使用`Comparator`接口，实现`int compare(T o1,T o2)`方法
>
> 个人看法：两种方式都是为了排序，一种是元素本身实现Comparable接口，另一种则是当元素本身没有实现\`Comparable\`接口时，可以通过\`Comparator\`接口（传入构造器\`TreeSet\(Comparator comparator\)\`），两者本身没有什么区别。
>
> [Java : Comparable vs Comparator](https://link.jianshu.com/?t=https://stackoverflow.com/questions/4108604/java-comparable-vs-comparator)
>
> In short, there isn't much difference. They are both ends to similar means. In general implement comparable for natural order, \(natural order definition is obviously open to interpretation\), and write a comparator for other sorting or comparison needs.

## java.util.concurrent

**CopyOnWriteArraySet**

 写时复制Set，通过CopyOnWriteArrayList来实现，适用于读多写少的并发场景。

**ConcurrentSkipListSet **

线程安全的并发Set，通过ConcurrentSkipListMap实现，支持对元素进行排序。

## 总结

| Set | 适用场景 | 底层实现 |
| :--- | :--- | :--- |
| HashSet | 元素不能重复 | HashMap |
| LinkedHashSet | 除了元素不重复，还要保持插入顺序 | LinkedHashMap |
| TreeSet | 除了元素不重复，还要对元素进行排序 | TreeMap |
| CopyOnWriteArraySet | 并发，写时复制，适用于读多写少的场景 | CopyOnWriteArrayList |
| ConcurrentSkipListSet | 并发，且支持排序 | ConcurrentSkipListMap |

Set比较聪明，自己什么都不干，都交给Map去实现。



