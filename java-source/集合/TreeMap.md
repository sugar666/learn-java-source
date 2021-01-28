# 集合 -- TreeMap 

## java中常使用的两种排序的方式
[Comparable和Comparator接口的区别](https://blog.csdn.net/u010859650/article/details/85009595)

`Comparable`是排序的接口，是**内部的排序器**，使用`x.compareTo(y)`来比较x，y的大小。
- 比较的原则：
    - `x > y`: return 负数
    - `x == y`: return 0
    - `x < y`: return 正数

`Comparator`是比较器接口，是**外部比较器**，使用`compare(x,y)`进行比较，比较的方式与`compareTo()`类似



```java
public class TreeMapDemo {

    @Data
    class DTO implements Comparable<DTO> {

        private Integer id;

        public DTO(Integer id) {
            this.id = id;
        }

        @Override
        public int compareTo(DTO o) {
            //默认从小到大排序
            return id - o.getId();
        }
    }


    @Test
    public void testIterator() {
        TreeMap<String, String> m = new TreeMap<>();
        m.put("asdf", "nihao");
        m.put("sdf", "nihao");
        m.put("df", "nihao");
        m.keySet().iterator();
    }

    @Test
    public void testTwoComparable() {
        // 第一种排序，从小到大排序，实现 Comparable 的 compareTo 方法进行排序
        List<DTO> list = new ArrayList<>();
        for (int i = 5; i > 0; i--) {
            list.add(new DTO(i));
        }
        Collections.sort(list);
        log.info(JSON.toJSONString(list));

        // 第二种排序，从大到小排序，利用外部排序器 Comparator 进行排序
        Comparator comparator = (Comparator<DTO>) (o1, o2) -> o2.getId() - o1.getId();
        List<DTO> list2 = new ArrayList<>();
        for (int i = 5; i > 0; i--) {
            list2.add(new DTO(i));
        }
        // Collections.sort(list, comparator);
        Collections.sort(list, (o1,o2) -> o2.getId() - o1.getId());
        log.info(JSON.toJSONString(list2));
    }


}
```
## TreeMap
`TreeMap`底层的数据结构也是**红黑树**，与`HashMap`中红黑树的结构是一样的

`TreeMap`的特点：
- 利用了红黑树左节点小，右节点大的特点，使用key进行排序，每个插入的元素都位于正确的位置上，维护了key的大小关系，使用与key需要排序的场景中
- 底层使用的是平衡红黑树的结构，常用的方法例如：`put`，`get`，`containsKey`，`remove`的时间复杂度都是`O(logn)`

### 比较器
传进来外部的比较器，就是用外部的比较器，否则，使用内部实现的比较器

### add
`put`
```java

public V put(K key, V value) {
    Entry<K,V> t = root;
    // 没有root的时候，直接的重建红黑树
    if (t == null) {
        // compare中key不能是null
        compare(key, key); // type (and possibly null) check

        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    
    int cmp;
    Entry<K,V> parent;
    // split comparator and comparable paths
    // 选择不同的比较器，先选择外部的比较器
    Comparator<? super K> cpr = comparator;
    if (cpr != null) {
        // 自旋找到 key 应该新增的位置，就是应该挂载那个节点的头上
        do {
            parent = t;
            // 比较key的大小
            cmp = cpr.compare(key, t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    else {
        if (key == null)
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
            Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0)
                t = t.left;
            else if (cmp > 0)
                t = t.right;
            else
                return t.setValue(value);
        } while (t != null);
    }
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0)
        parent.left = e;
    else
        parent.right = e;
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}
```

`compare`
```java
// 外部比较器为空的时候，使用内部比较器
final int compare(Object k1, Object k2) {
    return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
        : comparator.compare((K)k1, (K)k2);
}
```


1. 新增节点时，就是利用了红黑树左小右大的特性，从根节点不断往下查找，直到找到节点是 null 为止，节点为 null 说明到达了叶子结点;
2. 查找过程中，发现 key 值已经存在，直接覆盖;
3. **TreeMap 是禁止 key 是 null 值的。**

