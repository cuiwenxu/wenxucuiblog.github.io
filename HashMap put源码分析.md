##### HashMap如何确定哈希桶数组索引位置
确定哈希桶数组索引位置本质上就是三步：取key的hashCode值、高位运算、取模运算。
直接看源码
```java
static final int hash(Object key) {
        int h;
        //第一步取hashcode，key.hashcode记为h
        //第二步做高位运算 h^(h >>> 16)
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
        
        //第三步key对数组长度取模，即key.hashcode % (node<k,v>[].length)，jdk1.8中把这一操作直接放在putVal方法中，并且使用(n - 1) & hash来代替%运算，因为当数组长度为2的n次方时，(n - 1) & hash等价于hash%n,但是位运算效率更高。代码如下

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        //省略部分代码
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //省略部分代码
}
```



```java
   /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 判断HashMap的Node数组是否为空
        if ((tab = table) == null || (n = tab.length) == 0)
            // 如果为空，则初始化数组
            n = (tab = resize()).length;
        // 判断HashMap的Node数组的hash位置是否为空
        if ((p = tab[i = (n - 1) & hash]) == null)
            // 如果为空直接插入一个节点
            tab[i] = newNode(hash, key, value, null);
        // 如果不为空
        else {
            Node<K,V> e; K k;
            // 当前Node的Key和新插入的Key是否相等
            if (p.hash == hash &&
                    ((k = p.key) == key || (key != null && key.equals(k))))
                // 直接覆盖
                e = p;
            // 当前Node是否为红黑树的TreeNode
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 当前Node是否为单向链表的Node
            else {
                // 遍历单向链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 如果哈希冲突（哈希碰撞）的数量大于等于8，将单向链表转换为红黑树
                        // 当执行treeifyBin(tab, hash);的时候，也不一定必须转换成红黑树
                        // 如果一个Node的单向链表的长度小于64，扩容
                        // 如果一个Node的单向链表的长度大于等于64，才将此单向链表转换成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 当前Node的Key和新插入的Key是否相等
                    if (e.hash == hash &&
                            ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 判断新插入的Node是否已存在，如果已存在根据onlyIfAbsent是否用新值覆盖旧值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        // 扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
