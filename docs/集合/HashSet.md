# 集合 -- HashSet
HashSet与TreeSet都是在Map的基础上组合出来的类

## 类注释
- 底层的实现基于`HashMap`，保存是无序的（LinkedHashSet是基于LinkedHashMap实现的，可以按照插入的顺序进行遍历）
- 常用的操作`add`，`remove`，`contains`，`size`都是包装了`HashMap`中的方法(底层使用的数据结构是数组，链表与红黑树)，在不考虑hash冲突的情况下，时间复杂度都是`O(1)`
- 是线程不安全的，在`Collections.synchronizeSet`可以实现线程安全
- 在**迭代**的过程中，不能改变其数据结构，只能使用迭代器中的remove方法，否则会抛出`ConcurrentModificationException`

(上面列出来的2，3，4点都是List，Map，Set的共同的特点)

## 通过组合的方式实现`HashSet`

一般，复用已有的类有两种方法：
- **直接的继承**，覆写或直接的使用之前的方法，但是不能改变已有的方法的名称，例如：添加一个元素，只能使用`put`，但是在`HashSet`中使用的是`add`
- 组合基类，把基类当做类中的一个属性，通过调用基类用的方法，来包装本类中的方法（`HashSet`使用的是组合的策略）
    - 组合的方式更加的灵活，可以组合多个不同的类，打破了java只能继承一个类的局限性，而且组合后本类的方法名可以进行自定义，不必复用父类中的方法


给Map中的value默认值，`HashSet`对外只暴露`key`
```java
private transient HashMap<E,Object> map;

// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();

/**
 * Constructs a new, empty set; the backing <tt>HashMap</tt> instance has
 * default initial capacity (16) and load factor (0.75).
 */
public HashSet() {
    map = new HashMap<>();
}	
```

## 初始化

```java
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

```

`HashSet`中初始化的思路 ： 取`c.size()/.75f) + 1, 16`两个值中的最大值，16是HashMap默认的数组容量值

    往 HashMap 拷贝大集合时，如何给 HashMap 初始化大小时，完全可以借鉴这种写法:
        取最大值(期望的值 / 0.75 + 1，默认值 16)。
        

## add
直接的对`HashMap`中的`push()`进行了包装
```java
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```    

## 总结
- 组合策略的合理的使用
- 对复杂的逻辑进行包装，使得暴露的接口简单好用
- 在组合类的时候，需要对类中的api有足够的了解，才能更好的进行类的组合
- HashMap初始化的时候可以参考的模型:`c.size()/.75f) + 1, 16(max)`


