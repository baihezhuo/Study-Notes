## ConcurrentModificationException探究

### modCount  ?

> 在ArrayList,LinkedList,HashMap等等的内部实现增，删，改中我们总能看到modCount的身影，modCount字面意思就是修改次数 

```java 
// HashMap
transient int modCount;
// AbstractList的 subclasses implementation
protected transient int modCount = 0;
```

- 可以发现一个公共特点，所有使用modCount属性的全是线程不安全的 。
- iterator(or listiterator)的next、 remove、previous、 set、 add
- 该参数是可选的。

**Fail-Fast 机制** （快速失败）

java.util.List不是线程安全的，因此如果在使用迭代器的过程中有其他线程修改了list，那么将抛出ConcurrentModificationException，这就是fail-fast策略。

- 场景：java.util包下的集合类都是快速失败的，不能在多线程下发生并发修改（迭代过程中被修改）。  

##### Fail-Safe机制 （安全失败）

易混淆。采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。

- 由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到，所以不会触发CME
- 缺点：基于拷贝内容的优点是避免了Concurrent Modification Exception，但同样地，迭代器并不能访问到修改后的内容
- 场景：java.util.concurrent包下的容器都是安全失败，可以在多线程下并发使用，并发修改。

##### ConcurrentModificationException小栗子

> 下面例子在迭代器遍历时会有ConcurrentModificationException。

```java
public class CMETest {
    public static void main(String[] args) {
        Vector<String> list = new Vector<>();
        list.add("1");
        list.add("2");
        list.add("3");
        list.add("4");
        CMETest cmeTest =new CMETest();
        cmeTest.test2(list);
    }
    //foreach本质上还是Iterator
    public Collection<String> test(Vector<String> list){
        for(String temp:list){
            if("2".equals(temp)){
                list.remove(temp);
            }
        }
        return list;
    }

    public  Collection<String> test2(Vector<String> list){
        Iterator<String> iterator = list.iterator();
        while (iterator.hasNext()){
            String next = iterator.next();
            if("2".equals(next)){
                list.remove(next);
            }
        }
        return list;
    }

    //不会出现CME异常
    public Collection<String> test3(Vector<String> list){
        for (int i=0;i<list.size();i++){
            if("2".equals(list.get(i))){
                list.remove(list.get(i));
            }
        }
        return list;
    }

}

```

  原因：迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用 modCount 变量，在被遍历期间如果内容发生变化，就会modCount++。在遍历时首先hasNext()通过cursor != elementCount判断是否遍历完元素了(cursor 下一个元素的索引)。为遍历完进行next()这时会检测modCount变量是否为expectedmodCount值，是的话就返回遍历；否则抛出CME异常，终止遍历。 注意：这里异常的抛出条件是检测到 modCount！=expectedmodCount 这个条件。如果集合发生变化时修改modCount值刚好又设置为了expectedmodCount值，则异常不会抛出。所以不能依赖于这个异常是否抛出而进行并发操作的编程，这个异常只建议用于检测并发修改的bug。        

##### 诡异的现象

当删除的是容器的倒数第二个元素时，正常运行没有CME异常。

```java
public class CMETest {
    public static void main(String[] args) {
        Vector<String> list = new Vector<>();
        list.add("1");
        list.add("2");
        list.add("3");
        CMETest cmeTest =new CMETest();
        cmeTest.test2(list);
    }
    public  Collection<String> test2(Vector<String> list){
        Iterator<String> iterator = list.iterator();
        while (iterator.hasNext()){
            String next = iterator.next();
            if("2".equals(next)){
                list.remove(next);
            }
        }
        return list;
    }
}
```

　原因：迭代器在遍历时，首先会进行hasNext判断然后才进行next。在删除容器的倒数第二个元素后，elementCount--即时此时容量为2，然后进行hasNext判断，下一个元素的索引cursor为2，cursor != elementCount为false,没有下一位，直接结束了。所以并未报CME异常。

##### Iterator.remove(迭代器的remve方法)

```java
        public void remove() {
            if (lastRet == -1)
                throw new IllegalStateException();
            synchronized (Vector.this) {
                checkForComodification();
                Vector.this.remove(lastRet);
                expectedModCount = modCount;
            }
            cursor = lastRet;
            lastRet = -1;
        }
```

采用该方法删除时，删除后会对expectedModCount赋现在的修改次数的值，再后面进行next操作时，expectedModCount = modCount就避免了CME的报错。



#####  避免ConcurrentModificationException

- 使用for循环进行遍历的修改
- 使用Iterator的方法进行修改,Iterator.remove进行删除



​       

​