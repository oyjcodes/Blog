---
title: HashMap源码解析
date: 2019-07-11 11:14:24
tags: 
- 源码阅读
categories: "集合"    
---
## HashMap源码解析
1. 默认的常量
```java
//创建 HashMap 时未指定初始容量情况下的默认容量  16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
//HashMap 的最大容量 2^30 
static final int MAXIMUM_CAPACITY = 1 << 30;
//HashMap 默认的装载因子,当 HashMap 中元素数量超过容量装载因子时，进行　resize()　操作
static final float DEFAULT_LOAD_FACTOR = 0.75f; 
//链表转红黑树的阈值 
static final int TREEIFY_THRESHOLD = 8; 
//用来确定何时将解决 hash 冲突的红黑树转变为链表
static final int UNTREEIFY_THRESHOLD = 6;
```
2. 存储结构
内部包含了一个 Node 类型的数组 table。观察 Node 可以发现table是一个链表
```java
    /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;
```
Node 存储着键值对。它包含了四个字段，从 next 字段我们可以看出 table 是一个链表。即数组中的每个位置被当
成一个桶，一个桶存放一个链表。HashMap 使用拉链法来解决冲突，同一个链表中存放哈希值和散列桶取模运算结
果相同的 Ndoe.
```java
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;//保存节点的hash值
        final K key;//保存节点的key值
        V value;//保存节点的value值
        Node<K,V> next;//指向链表结构下的当前节点的 next 节点，红黑树 TreeNode 节点中也有用到

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```
 TreeNode<K,V> 继承 LinkedHashMap.Entry<K,V>，用来实现红黑树相关的存储结构
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;  // 存储当前节点的父节点
        TreeNode<K,V> left;　//存储当前节点的左孩子
        TreeNode<K,V> right;　//存储当前节点的右孩子
        TreeNode<K,V> prev;    // 存储当前节点的前一个节点
        boolean red;　// 存储当前节点的颜色（红、黑）
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }
        TreeNode(int hash, K key, V val, Node<K,V> next) {
            super(hash, key, val, next);
        }

        /**
         * Returns root of tree containing this node.
         */
        final TreeNode<K,V> root() {
            for (TreeNode<K,V> r = this, p;;) {
                if ((p = r.parent) == null)
                    return r;
                r = p;
            }
        }
```

3. HashMap的结构<br>
![avatar](1563864906121.png)<br>


4. 拉链法的工作原理
```java
HashMap<String, String> map = new HashMap<>();
map.put("K1", "V1");
map.put("K2", "V2");
map.put("K3", "V3");
```
. 新建一个 HashMap，默认大小为 16；<br>
. 插入 <K1,V1> 键值对，先计算 K1 的 hashCode 为 115，使用除留余数法得到所在的桶下标 115%16=3。<br>
. 插入 <K2,V2> 键值对，先计算 K2 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6。<br>
. 插入 <K3,V3> 键值对，先计算 K3 的 hashCode 为 118，使用除留余数法得到所在的桶下标 118%16=6，插在<K2,V2> 前面。<br>
注意：应该注意到链表的插入是以头插法方式进行的，例如上面的 <K3,V3> 不是插在 <K2,V2> 后面，而是插入在链表头
部<br>

查找需要分成两步进行：
计算键值对所在的桶；
在链表上顺序查找，时间复杂度显然和链表的长度成正比。

5. put操作
```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    } 
    // 键为 null 单独处理
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    // 确定桶下标
    int i = indexFor(hash, table.length);
    // 先找出是否已经存在键为 key 的键值对，如果存在的话就更新这个键值对的值为 value
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
    // 插入新键值对
    addEntry(hash, key, value, i);
    return null;
    }

```
确定桶下标的最后一步是将 key 的 hash 值对桶个数取模：hash%capacity，如果能保证 capacity 为 2 的 n 次方，
那么就可以将这个操作转换为位运算。
```java
static int indexFor(int h, int length) {
    return h & (length-1);
}
```



HashMap 允许插入键为 null 的键值对。但是因为无法调用 null 的 hashCode() 方法，也就无法确定该键值对的桶下
标，只能通过强制指定一个桶下标来存放。HashMap 使用第 0 个桶存放键为 null 的键值对。
```java
    private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
            modCount++;
            addEntry(0, null, value, 0);
        return null;
    }
```
使用链表的头插法，也就是新的键值对插在链表的头部，而不是链表的尾部。
```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    } 
        createEntry(hash, key, value, bucketIndex);
    } 
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    // 头插法，链表头部指向新的键值对
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

6. 扩容-基本原理
设 HashMap 的 table 长度为 M，需要存储的键值对数量为 N，如果哈希函数满足均匀性的要求，那么每条链表的长
度大约为 N/M，因此平均查找次数的复杂度为 O(N/M)。
为了让查找的成本降低，应该尽可能使得 N/M 尽可能小，因此需要保证 M 尽可能大，也就是说 table 要尽可能大。
HashMap 采用动态扩容来根据当前的 N 值来调整 M 值，使得空间效率和时间效率都能得到保证。
和扩容相关的参数主要有：capacity、size、threshold 和 load_factor。

![avatar](1563867424345345.png)<br>
从下面的添加元素代码中可以看出，当需要扩容时，令 capacity 为原来的两倍。
```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    //键值对的数量size大于threshold时进行扩容操作
    if (size++ >= threshold)
    //扩容使用resize()实现
        resize(2 * table.length);
}
```
扩容使用 resize() 实现，需要注意的是，扩容操作同样需要把 oldTable 的所有键值对重新插入 newTable 中，因此
这一步是很费时的。
```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }
    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable);
    table = newTable;
    threshold = (int)(newCapacity * loadFactor);
}

void transfer(Entry[] newTable) {
    Entry[] src = table;
    int newCapacity = newTable.length;
    for (int j = 0; j < src.length; j++) {
        Entry<K,V> e = src[j];
        if (e != null) {
            src[j] = null;
            do {
                Entry<K,V> next = e.next;
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            } while (e != null);
        }
    }
}

```
7. 扩容-重新计算桶下标
在进行扩容时，需要把键值对重新放到对应的桶上。HashMap 使用了一个特殊的机制，可以降低重新计算桶下标的操作。
假设原数组长度 capacity 为 16，扩容之后 new capacity 为 32：<br>
capacity     : 00010000<br>
new capacity : 00100000
对于一个 Key，

它的哈希值如果在第 5 位上为 0，那么取模得到的结果和之前一样；
如果为 1，那么得到的结果为原来的结果 +16。

8. 计算数组容量
HashMap 构造函数允许用户传入的容量不是 2 的 n 次方，因为它可以自动地将传入的容量转换为 2 的 n 次方。<br>

先考虑如何求一个数的掩码，对于 10010000，它的掩码为 11111111，可以使用以下方法得到：
```txt
mask |= mask >> 1 11011000
mask |= mask >> 2 11111110
mask |= mask >> 4 11111111
```

8. 链表转红黑树
从 JDK 1.8 开始，一个桶存储的链表长度大于 8 时会将链表转换为红黑树。