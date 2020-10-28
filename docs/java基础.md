# java基础

# String

String类型是不可变类型，体现在String类源码上：==使用了两重的final进行了修饰==
- String类被`final`修饰
    - final关键字的作用：
        - 被final修饰的类不能被继承
        - 被final修饰的方法不要被重写
        - 被final修饰的对象需要在声明的时候就进行初始化，且不能被修改==内存地址==
- String中保存数据是==使用char的数组，value==，value也是被final修饰的，被赋值以后，其内存地址是无法进行修改的

String
```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence,
               Constable, ConstantDesc {

    /**
     * The value is used for character storage.
     *
     * @implNote This field is trusted by the VM, and is a subject to
     * constant folding if String instance is constant. Overwriting this
     * field after construction will cause problems.
     *
     * Additionally, it is marked with {@link Stable} to trust the contents
     * of the array. No other facility in JDK provides this functionality (yet).
     * {@link Stable} is safe here, because value is never null.
     */
    @Stable
    private final byte[] value;
```

## substring
String中的字符转的截取`substring`调用的是底层的`Arrays.copyOfRange()`

substring
```java
public String substring(int beginIndex, int endIndex) {
    int length = length();
    checkBoundsBeginEnd(beginIndex, endIndex, length);
    int subLen = endIndex - beginIndex;
    if (beginIndex == 0 && endIndex == length) {
        return this;
    }
    return isLatin1() ? StringLatin1.newString(value, beginIndex, subLen)
                      : StringUTF16.newString(value, beginIndex, subLen);
}
```

`Arrays.copyOfRange()`

```java
public static String newString(byte[] val, int index, int len) {
    if (String.COMPACT_STRINGS) {
        byte[] buf = compress(val, index, len);
        if (buf != null) {
            return new String(buf, LATIN1);
        }
    }
    int last = index + len;
    return new String(Arrays.copyOfRange(val, index << 1, last << 1), UTF16);
}
```

`Arrays.copyOfRange()`调用的是native方法，`System.arraycopy`


```java
public static byte[] copyOfRange(byte[] original, int from, int to) {
    int newLength = to - from;
    if (newLength < 0)
        throw new IllegalArgumentException(from + " > " + to);
    byte[] copy = new byte[newLength];
    System.arraycopy(original, from, copy, 0,
                     Math.min(original.length - from, newLength));
    return copy;
}
```

## equals
首先判断是不是同一个对象，然后判断是不是空对象，在用`instanceof`判断是不是同一类型的对象，在去进行不同属性的比较。

在String中，由于具体储存是由char型的数组value进行储存的，在进行String比较的时候：
- 比较是不是使用了同种编码
- 在去一个字符一个字符的比较是不是对应相等的


```java
public boolean equals(Object anObject) {
    if (this == anObject) {
        return true;
    }
    if (anObject instanceof String) {
        String aString = (String)anObject;
        if (coder() == aString.coder()) {
            return isLatin1() ? StringLatin1.equals(value, aString.value)
                              : StringUTF16.equals(value, aString.value);
        }
    }
    return false;
}
```


```java
@HotSpotIntrinsicCandidate
public static boolean equals(byte[] value, byte[] other) {
    if (value.length == other.length) {
        for (int i = 0; i < value.length; i++) {
            if (value[i] != other[i]) {
                return false;
            }
        }
        return true;
    }
    return false;
}

```

## replace
由于String类是不可变的对象，在使用`replace`进行替换的时候，实际上是新建了一个新的String

返回的`String`类型

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        String ret = isLatin1() ? StringLatin1.replace(value, oldChar, newChar)
                                : StringUTF16.replace(value, oldChar, newChar);
        if (ret != null) {
            return ret;
        }
    }
    return this;
}
```


```java
public static String replace(byte[] value, char oldChar, char newChar) {
    int len = value.length >> 1;
    int i = -1;
    while (++i < len) {
        if (getChar(value, i) == oldChar) {
            break;
        }
    }
    if (i < len) {
        byte buf[] = new byte[value.length];
        for (int j = 0; j < i; j++) {
            putChar(buf, j, getChar(value, j)); // TBD:arraycopy?
        }
        while (i < len) {
            char c = getChar(value, i);
            putChar(buf, i, c == oldChar ? newChar : c);
            i++;
       }
       // Check if we should try to compress to latin1
       if (String.COMPACT_STRINGS &&
           !StringLatin1.canEncode(oldChar) &&
           StringLatin1.canEncode(newChar)) {
           byte[] val = compress(buf, 0, len);
           if (val != null) {
               return new String(val, LATIN1);
           }
       }
       return new String(buf, UTF16);
    }
    return null;
}
```

## split
在使用`split`进行字符串的分割的时候，==会出现空字符串的形式==

推荐使用 Guava 的 API 对字符串进行分割与连接（split/join）

# Long
==Long有缓存机制，缓存了 -128 ~ 127内所有的Long值==，使用static在初始化的时候就会生成缓存，在这个范围类的Long值，直接的从缓存中拿去，节约内存的消耗


```java
private static class LongCache {
    private LongCache() {}

    static final Long[] cache;
    static Long[] archivedCache;

    static {
        int size = -(-128) + 127 + 1;

        // Load and use the archived cache if it exists
        VM.initializeFromArchive(LongCache.class);
        if (archivedCache == null || archivedCache.length != size) {
            Long[] c = new Long[size];
            long value = -128;
            for(int i = 0; i < size; i++) {
                c[i] = new Long(value++);
            }
            archivedCache = c;
        }
        cache = archivedCache;
    }
}
```

`IntegerCache`

```java
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer[] cache;
    static Integer[] archivedCache;

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                h = Math.max(parseInt(integerCacheHighPropValue), 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(h, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        // Load IntegerCache.archivedCache from archive, if possible
        VM.initializeFromArchive(IntegerCache.class);
        int size = (high - low) + 1;

        // Use the archived cache if it exists and is large enough
        if (archivedCache == null || size > archivedCache.length) {
            Integer[] c = new Integer[size];
            int j = low;
            for(int i = 0; i < c.length; i++) {
                c[i] = new Integer(j++);
            }
            archivedCache = c;
        }
        cache = archivedCache;
        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}

```

short,long,int,double,float都会有缓存

在使用类型转换的时候，尽量的使用`valueOf`，valueOf 方法会从缓存中去拿值，如果命中缓存，会减少资源的开销，parseLong 方法就没有这个机制。


```java
public static Long valueOf(long l) {
    final int offset = 128;
    if (l >= -128 && l <= 127) { // will cache
        return LongCache.cache[(int)l + offset];
    }
    return new Long(l);
}

```
他们返回类型的不同是最大的原因。
static int parseInt(String s) 将字符串参数作为有符号的十进制整数进行分析。
static Integer valueOf(int i) 返回一个表示指定的 int 值的 Integer 实例。 
static Integer valueOf(String s) 返回保持指定的 String 的值的 Integer 对象。 
从返回值可以看出他们的区别   parseInt()返回的是基本类型int 
而valueOf()返回的是包装类Integer 
 Integer是可以使用对象方法的  而int类型就不能和Object类型进行互相转换。

# java中的关键字 
##  static
- 父类的静态变量和静态块比子类优先进行初始化
- 静态变量和静态块比类构造器优先初始化

静态变量和静态块的初始化的顺序和在代码中编写的顺序有关


## transient
常用来修饰变量，代表当前的变量是无需进行序列化的（序列化是将变量变成二进制流，写在硬盘中，一般是在IO中进行使用）
[transient](https://baijiahao.baidu.com/s?id=1636557218432721275&wfr=spider&for=pc)

# Collections （工具类的构造函数应该是私有的）
对原始的集合类进行了封装，一种是线程安全的，一种是不可变类

==Collection是接口，set，List，Map都继承Collection==

1. max 方法泛型 T 定义得非常巧妙，意思是泛型必须继承 Object 并且实现 Comparable 的接 口。一般让我们来定义的话，我们可以会在方法里面去判断有无实现 Comparable 的接口， 这种是在运行时才能知道结果。而这里泛型直接定义了必须实现 Comparable 接口，在编译 的时候就可告诉使用者，当前类没有实现 Comparable 接口，使用起来很友好;
2. 给我们提供了实现两种排序机制的好示例:自定义类实现 Comparable 接口和传入外部排序 器。两种排序实现原理类似，但实现有所差别，我们在工作中如果需要些排序的工具类时， 可以效仿。

```java
public static <T extends Object & Comparable<? super T>> T max(Collection<? extends T> coll) {
        Iterator<? extends T> i = coll.iterator();
        T candidate = i.next();

        while (i.hasNext()) {
            T next = i.next();
            if (next.compareTo(candidate) > 0)
                candidate = next;
        }
        return candidate;
    }
```


Collections ==对原始集合类进行了封装==，提供了更好的集合类给我们，==一种是线程安全的集合， 一种是不可变的集合==

## 线程安全的集合
使用`Synchronized`同步锁实现线程安全，`Collections`中的线程安全的方法都是用`Synchronized`作为前缀


`SynchronizedList`

```java
static class SynchronizedList<E>
    extends SynchronizedCollection<E>
    implements List<E> {
    private static final long serialVersionUID = -7754090372962971524L;

    final List<E> list;

    SynchronizedList(List<E> list) {
        super(list);
        this.list = list;
    }
    SynchronizedList(List<E> list, Object mutex) {
        super(list, mutex);
        this.list = list;
    }

    public boolean equals(Object o) {
        if (this == o)
            return true;
        synchronized (mutex) {return list.equals(o);}
    }
    public int hashCode() {
        synchronized (mutex) {return list.hashCode();}
    }

    public E get(int index) {
        synchronized (mutex) {return list.get(index);}
    }
    public E set(int index, E element) {
        synchronized (mutex) {return list.set(index, element);}
    }
    public void add(int index, E element) {
        synchronized (mutex) {list.add(index, element);}
    }
    public E remove(int index) {
        synchronized (mutex) {return list.remove(index);}
    }

    public int indexOf(Object o) {
        synchronized (mutex) {return list.indexOf(o);}
    }
    public int lastIndexOf(Object o) {
        synchronized (mutex) {return list.lastIndexOf(o);}
    }

    public boolean addAll(int index, Collection<? extends E> c) {
        synchronized (mutex) {return list.addAll(index, c);}
    }

    public ListIterator<E> listIterator() {
        return list.listIterator(); // Must be manually synched by user
    }

    public ListIterator<E> listIterator(int index) {
        return list.listIterator(index); // Must be manually synched by user
    }

    public List<E> subList(int fromIndex, int toIndex) {
        synchronized (mutex) {
            return new SynchronizedList<>(list.subList(fromIndex, toIndex),
                                        mutex);
        }
    }

    @Override
    public void replaceAll(UnaryOperator<E> operator) {
        synchronized (mutex) {list.replaceAll(operator);}
    }
    @Override
    public void sort(Comparator<? super E> c) {
        synchronized (mutex) {list.sort(c);}
    }

    private Object readResolve() {
        return (list instanceof RandomAccess
                ? new SynchronizedRandomAccessList<>(list)
                : this);
    }
}
```

## 不可变集合
只能在进行初始化的时候进行复制，其他的时候只提供查询的接口，使用`set`时，会抛出异常`UnsupportedOperationException`

`UnmodifiableList`

```java
static class UnmodifiableList<E> extends UnmodifiableCollection<E>
                              implements List<E> {
    private static final long serialVersionUID = -283967356065247728L;

    final List<? extends E> list;

    UnmodifiableList(List<? extends E> list) {
        super(list);
        this.list = list;
    }

    public boolean equals(Object o) {return o == this || list.equals(o);}
    public int hashCode()           {return list.hashCode();}

    public E get(int index) {return list.get(index);}
    public E set(int index, E element) {
        throw new UnsupportedOperationException();
    }
    public void add(int index, E element) {
        throw new UnsupportedOperationException();
    }
    public E remove(int index) {
        throw new UnsupportedOperationException();
    }
    public int indexOf(Object o)            {return list.indexOf(o);}
    public int lastIndexOf(Object o)        {return list.lastIndexOf(o);}
    public boolean addAll(int index, Collection<? extends E> c) {
        throw new UnsupportedOperationException();
    }
}
```

# Objects
`Object`位于`java.lang`
`Objects`位于`java.util`

==使用Objects判断相等与判空==

```java
public static boolean equals(Object a, Object b) {
    return (a == b) || (a != null && a.equals(b));
}

public static boolean deepEquals(Object a, Object b) {
    if (a == b)
        return true;
    else if (a == null || b == null)
        return false;
    else
        return Arrays.deepEquals0(a, b);
}

```


