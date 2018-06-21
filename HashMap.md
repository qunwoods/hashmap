# HashMap
#java
[Java8系列之重新认识HashMap - ImportNew](http://www.importnew.com/20386.html)
[教你初步了解红黑树 - CSDN博客](https://blog.csdn.net/v_july_v/article/details/6105630)

> HashMap实现了Map接口（接口提供了可选得map操作），支持key和value为空；  
> HashMap VS HashTable：HashMap支持key、value为null且线程不安全，HashTable不支持null、线程安全；  
> HAshMap：无序，不保证元素顺序不变；  

### 性能
	* get和put的时间复杂度为O(1)；
	* capacity & load factor：hash table容量被初始化为capacity，load factor表示hash table的最大饱和度，当hash table中的元素超过 capacity 
	*  load factor，hash table会被重构；

### 关于线程同步
	* HashMap未实现同步，当多个线程同时访问且至少一个线程执行修改时，需要外部实现线程同步；
	
### HashMap数据结构
	* 采用哈希表进行存储，采用链地址法解决冲突：
	* HashMap的底层是一个数组（table），数组中的每一项是一个链表（Node），每个Map.Entry是一个键值对，Node实现了Map.Entry，还包含一个指向下一个元素的引用（链表）；
	* 结构定义
	
![structure](https://github.com/qunwoods/hashmap/blob/master/18FA4CAD-55CA-4D69-92EE-D31F83F5E5B8.png)
	
```java
 /**
     * The table, initialized on first use, and resized as
     * necessary. When allocated, length is always a power of two.
     * (We also tolerate length zero in some operations to allow
     * bootstrapping mechanics that are currently not needed.)
     */
    transient Node<K,V>[] table;

	 /**
     * Basic hash bin node, used for most entries.  (See below for
     * TreeNode subclass, and in LinkedHashMap for its Entry subclass.)
     */
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
```
	* ⚠️在java8中，当链表中的元素超过TREEIFY_THRESHOLD(8个)时，会将链表准换成红黑树，降低搜索的时间复杂度（O(n) -> O(logN)）

### put过程分析
	* 若table为空，初始化数组；
	* 根据hash值计算数组index，若当前index中没有存储元素，直接存入该元素；
	* 若当前index存储了元素，说明产生了冲突，进行冲突处理：
		* 判断当前index中存储元素的结构（链表or红黑树）；
		* 判断为红黑树，调用putTreeVal；
		* 判断为链表：
			* 遍历链表，在链表中搜索key，若当前元素已存在，更新当前元素的值，若不存在，挂在链表末尾；
			* 遍历中判断链表长度是否达到转红黑树的阀值，若已达到则进行转换；
![process](https://github.com/qunwoods/hashmap/blob/master/E7A89B6F-7D7E-4FE6-91BE-1ACF51407214.png)

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
		  // 插入第一个元素时，需要先初始化数组的大小
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // (n - 1) & hash求出index，如果当前index没有存储元素，那么将Node放到index中；
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
		  // 当前位置已存储元素，处理冲突
        else {
            Node<K,V> e; K k;
			  // 如果节点key存在，直接覆盖value
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode) // 判断为空黑树
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else { // 判断为链表
                // 循环遍历链表，找到链表中hash值相同的key（元素已存在）或者遍历到链表末尾
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) { 
                        p.next = newNode(hash, key, value, null);
                        // 判断当前链表长度是否需要转成红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
			  // 根据搜索结果，如果e不为null，说明该元素已存在，直接更新，若为空则挂在链表的末尾；
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold) //如果插入元素后超过扩容阀值，需要进行扩容；
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```

### 扩容
[Java8系列之重新认识HashMap - ImportNew](http://www.importnew.com/20386.html)
- [ ] 扩容相对比较复杂，还没有看懂～

![扩容](https://github.com/qunwoods/hashmap/blob/master/BFBC0CFF-3874-4A52-9637-373CB7049720.png)

### get过程分析
```java
 /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // 根据key计算出存储该元素的table中的index，若当前index为空返回null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
			  // 如果头节点即为要查找的节点，直接返回，否则遍历链表（红黑树），找到即返回，否则返回null
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```

### 删除元素
```java
/**
     * Implements Map.remove and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to match if matchValue, else ignored
     * @param matchValue if true only remove if value is equal
     * @param movable if false do not move other nodes while removing
     * @return the node, or null if none
     */
    final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        // table元素个数大于0，计算当前key的在数组中的index，若table[index]不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
			  // 在链表或者红黑树中找到要被删除的节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            // 要被删除的节点存在，则从链表或者红黑树中删除该节点，为空表示不存在要删除的key；
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)
                    tab[index] = node.next;
                else
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```



