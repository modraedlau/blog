---
title: ArrayList的迭代删除
date: 2019-01-10 17:27:46
tags: ArrayList ConcurrentModificationException
category: Java并发编程
---

## 引子 ##
今天遇到一个问题，就是在使用ArrayList迭代删除出现问题，算是很常见的问题，我知道怎么写是正确的，但是一直不知道为什么会这样，今天我就把这个问题稍微研究深入一点，甚至从JDK的设计思路来想下为什么要这么设计。
先从问题开始，示例代码：
```java
public static void main(String[] args) {
    ArrayList<String> list = new ArrayList<>();
    list.add("a");
    list.add("b");
    list.add("c");

    Iterator<String> it = list.iterator();
    while (it.hasNext()) {
      // it.remove(); // IllegalStateException
      list.remove(it.next()); // ConcurrentModificationException
    }
}
```
貌似在迭代中，怎么删除都异常，使用使用迭代器删除抛出IllegalStateException，list本身删除抛出ConcurrentModificationException。
使用迭代器删除为什么会出现IllegalStateException呢，我们来看下在ArrayList实现的迭代器源码remove方法开始部分：
```java
if (lastRet < 0)
    throw new IllegalStateException();
```

原来是指向不正确，明眼人马上就看出来了正确删除写法：
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    it.next();
    it.remove(); // OK
}
```

那么直接使用list删除就必然出现ConcurrentModificationException吗，其实不然，下面展示一些：
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
  list.remove("a"); // OK
  break;
}
```

那为什么list.remove(it.next());就会出现异常呢，为什么会这样？还是要到ArrayList的源码中去看，ArrayList的迭代器next方法在最开始的时候都会checkForComodification：
```java
public E next() {
    checkForComodification();
```

那我们继续跟进行checkForComodification方法：
```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```
再根据上下文我们知道这个modCount表示的实际修改的次数，而expectedModCount表示预计修改的次数，如果不相等就会发生ConcurrentModificationException。

这会我们来debug下，添加三个元素后迭代开始时，debug可以的代：**modCount = 3, expectedModCount = 3**；
接下来我们使用list.remove一次，再进入这里可以发现：**modCount = 4, expectedModCount = 3**；
这说明多了一次修改！然后迭代器在next方法内checkForComodification时抛出ConcurrentModificationException。

那为什么it.remove又是正常的，我们再跟入ArrayList的迭代器实现中的remove方法：
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount; // 注意这里
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
是不是会突然恍然大悟，迭代器中的实现将expectedModCount = modCount又给变成最新的修改值了，而下一次的next方法就会正常执行。


## 思考 ##
使用modCount的作用主要是为了判断发生不正常的修改，而使用list.remove()就会发生，因为在多处会发生修改，不单只会在迭代时还可以在其他任意地方，甚至在多线程环境下造成混乱，而抛出此异常也是给开发人员警告。
而JDK的设计人员想引导开发人员使用迭代器remove()，我想主要是使用时只会在迭代元素时，而且迭代器本身有状态判断，基本不会造成其他问题，所以像迭代器remove()操作基本可用。
