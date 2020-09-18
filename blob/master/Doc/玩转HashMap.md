#### 玩转HashMap

​	HashMap相对于其他的集合具有更高的复杂度，涉及到了数组、链表、红黑树相关知识，HashMap之所以复杂，不仅是因为它复杂的数据结构，而且它不同数据结构之间的转换也是非常复杂。在链表和红黑树进行转换时涉及到阈值问题，初始化时也会涉及到多个初始化参数如DEFAULT_INITIAL_CAPACITY、MAXIMUM_CAPACITY等，具体参数如下。

```java
	//初始容量，初始化时容量。默认值为16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    //扩容极限，最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;

    //扩容因子，当当前容量占据总容量达到这个比例时进行扩容
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //扩容时链表长度为此值时，链表会转换为红黑树结构
    static final int TREEIFY_THRESHOLD = 8;

    //树大小小于 此值时,则会由树重新退化为链表
    static final int UNTREEIFY_THRESHOLD = 6;

    //最小树形化容量阈值，当hash表容量大于此值时才能够转化为红黑树
    static final int MIN_TREEIFY_CAPACITY = 64;
```

构造方法:针对不同的情况，HashMap提供了多种构造方法，我们可以通过传入不同的参数进行初始化，通过控制初始化参数我们能够减少由于扩容而带来的性能问题，也可以通过参数平衡数组每个元素中的节点个数（链表或者红黑树）与数组长度的关系，数组长度和树节点树是一个矛盾点，树节点树越多数组的长度就越短（节点在数组中分布均匀）。扩容因子越大，树的长度可能就越长，操作可能就会越麻烦，反之可能数组空间利用率就越低，可能有些数组元素还是链表甚至还没有元素就需要进行扩容，性能问题就越明显。

```java
//传入初始容量和负载因子，可用于已知大小的集合，通过初始化容量大小能够有效的控制扩容的次数甚至做到不扩容，扩容因子能够提高集合利用率
	public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " + initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " + loadFactor);
        this.loadFactor = loadFactor;
      	//计算容量大小，这里时保证hashmap大小为2的幂次方的地方
        this.threshold = tableSizeFor(initialCapacity);
    }

    //只控制集合初始容量
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    //全部使用默认值
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; //all other fields defaulted
    }

    //传入Map进行初始化
    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }


```

tableSizeFor()根据初始化的值重新计算threshold参数，该值之后会用于数组初始化，这里计算是取大于等于该值的2的最小幂次方。而默认值也是2的幂次方，这样初始化出来的数组就能够保证为2的幂次方，扩容也是乘以2也能够保证为2的幂次方。

算法：通过移位与原值取或操作，这里直接取值二进制数的最高位进行计算即可，低位不会对结果产生影响。第一次计算会将最高位和次高位值处理成1，第二次操作将前四位处理为1，第三次是前八位，第四次前十六位，最后是前32位。即将原值右侧每一位变成1，这样加1操作到每一位都是进一，最终会是2的幂次方。

```java
//计算大于该值的最小的2的幂次方
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

链表转树操作treeify

```java

//链表转成树
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
     	//树的根节点为空，直接将插入的节点作为根节点
    	if (root == null) {
      		x.parent = null;
      		x.red = false;
      		root = x;
    	}
    	else {
          K k = x.key;
          int h = x.hash;
          Class<?> kc = null;
          for (TreeNode<K,V> p = root;;) {
              int dir, ph;
              K pk = p.key;
              if ((ph = p.hash) > h)
                  dir = -1;
              else if (ph < h)
                  dir = 1;
              //比较两个类引用地址算出的值的大小
              else if ((kc == null &&
              (kc = comparableClassFor(k)) == null) ||
              (dir = compareComparables(kc, k, pk)) == 0
                  dir = tieBreakOrder(k, pk);
                  TreeNode<K,V> xp = p;
              if ((p = (dir <= 0) ? p.left : p.right) == null) {
                  x.parent = xp;
                  if (dir <= 0)
                      xp.left = x;
                  else
                      xp.right = x;
                	  //平衡插入
                      root = balanceInsertion(root, x);
              break;
              }
          }
      }
   }
   moveRootToFront(tab, root);
}
```

```java
//红黑树树反转回链表
final Node<K,V> untreeify(HashMap<K,V> map) {
  Node<K,V> hd = null, tl = null;
  for (Node<K,V> q = this; q != null; q = q.next) {
    //树节点转换为链表节点
    Node<K,V> p = map.replacementNode(q, null);
    if (tl == null)
      hd = p;
    else
      tl.next = p;
    tl = p;
  }
  return hd;
}
```



hash计算，这里是将原数值右移16位进行取与运算。hashmap数组是2的幂次方，这样该值二进制树位于数组长度二进制树的左侧位不会进行散列时不会对散列产生影响（直接为倍数），通过hash运算可以将高位的差异扩散到低位。

```java
//hash计算
static final int hash(Object key) {
    int h;
  	//hash计算通过右移位取与计算，能够将高位的影响传递给低位。原因是hashmap要求容量为2的幂函数，高位不会产生余数。对散列影响较小，可能造成数据分布不均匀
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

HashMap比较复杂，其插入和删除操作涉及到扩容，链表与树转换操作。这里主要研究下插入，顺便看下遍历与删除。插入流程：

1.判断table数组是否为空，如果为空调用resize方法进行新建

2.判断数组对应元素是否为空，即之前是否插入过节点

3.判断是否为树结构

4.链表插入，插入后是否达到转换成红黑树的阈值

5.链表转换位红黑树

```java
//插入数据入口
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
//具体插入逻辑
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //数组为空的情况
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length; //-----------------重新计算数组大小----------------------//
    //n-1与hash取与计算，既是取余，散列（hashmap要求数组长度为2的幂次方），散列元素为空情况直接新建一个节点。
    if ((p = tab[i = (n - 1) & hash]) == null)//-----------------数组元素是否位空----------------------//
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
      	//对应数组元素首个链表或者树元素为插入元素直接进行替换
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
      //插入元素为红黑树
        else if (p instanceof TreeNode)//-----------------是否是树----------------------//
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
          	//对应插入的数组元素为链表
            for (int binCount = 0; ; ++binCount) {
              	//循环遍历链表，binCount是为了判断当前链表长度
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                  	//插入节点后，链表达到转化为树的阈值情况
                    if (binCount >= TREEIFY_THRESHOLD - 1) //-----------------重新达到阈值----------------------//
                      	//链表转换为树
                        treeifyBin(tab, hash);
                    break;
                }
              	//当前节点等于插入节点的情况，直接跳出循环，不再插入
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
          	//节点插入后处理（未作处理）
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    //节点插入后处理（未作处理）
    afterNodeInsertion(evict);
    return null;
}

//链表转为树
final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
              	//链表节点转为树节点
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
          	//对树进行处理，使其满足红黑树要求
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
     }
 }
//返回比较类型（必须是实现了Comparable）
static Class<?> comparableClassFor(Object x) {
    if (x instanceof Comparable) {
        Class<?> c; Type[] ts, as; Type t; ParameterizedType p;
        if ((c = x.getClass()) == String.class) // bypass checks
            return c;
        if ((ts = c.getGenericInterfaces()) != null) {
            for (int i = 0; i < ts.length; ++i) {
                if (((t = ts[i]) instanceof ParameterizedType) &&
                    ((p = (ParameterizedType)t).getRawType() ==
                     Comparable.class) &&
                    (as = p.getActualTypeArguments()) != null &&
                    as.length == 1 && as[0] == c) // type arg is c
                    return c;
            }
        }
    }
    return null;
}
//比较两个值
static int compareComparables(Class<?> kc, Object k, Object x) {
    return (x == null || x.getClass() != kc ? 0 :
            ((Comparable)k).compareTo(x));
}

//树节点的插入
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab,
                                       int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)//当前节点的hash大于插入系节点的hash
                    dir = -1;
                else if (ph < h)//当前节点的hash小于插入系节点的hash
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))//当前节点的hash等于插入系节点的hash，且																		   equals相等
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {   //碰撞，hash值相等equals不等
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);//根据存储地址求取hash值进行处理
                }

                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    Node<K,V> xpn = xp.next;
                    TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    xp.next = x;
                    x.parent = x.prev = xp;
                    if (xpn != null)
                        ((TreeNode<K,V>)xpn).prev = x;
                    moveRootToFront(tab, balanceInsertion(root, x));
                    return null;
                }
            }
        }
//左旋平衡,将当前节点的右节点的左节点赋给当前节点的右节点，当前节点的右节点赋值当前节点的父节点，当前节点的右节点当作当前节点的父节点
static <K,V> TreeNode<K,V> rotateLeft(TreeNode<K,V> root,TreeNode<K,V> p) {
    TreeNode<K,V> r, pp, rl;
    if (p != null && (r = p.right) != null) {
        if ((rl = p.right = r.left) != null)
            rl.parent = p;
        if ((pp = r.parent = p.parent) == null)
            (root = r).red = false;
        else if (pp.left == p)
            pp.left = r;
        else
            pp.right = r;
        r.left = p;
        p.parent = r;
    }
    return root;
}
//右旋平衡，与左旋转相反操作
static <K,V> TreeNode<K,V> rotateRight(TreeNode<K,V> root,TreeNode<K,V> p) {
    TreeNode<K,V> l, pp, lr;
    if (p != null && (l = p.left) != null) {
        if ((lr = p.left = l.right) != null)
            lr.parent = p;
        if ((pp = l.parent = p.parent) == null)
            (root = l).red = false;
        else if (pp.right == p)
            pp.right = l;
        else
            pp.left = l;
        l.right = p;
        p.parent = l;
    }
    return root;
}
//平衡插入，树节点的插入，插入后根据插入的数据来进行判断是否平衡，不平衡则进行旋转平衡操作
static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,TreeNode<K,V> x) {
    x.red = true;
    for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
        if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        else if (!xp.red || (xpp = xp.parent) == null)
            return root;
        if (xp == (xppl = xpp.left)) {
            if ((xppr = xpp.right) != null && xppr.red) {
                xppr.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.right) {
                    root = rotateLeft(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateRight(root, xpp);
                    }
                }
            }
        }
        else {
            if (xppl != null && xppl.red) {
                xppl.red = false;
                xp.red = false;
                xpp.red = true;
                x = xpp;
            }
            else {
                if (x == xp.left) {
                    root = rotateRight(root, x = xp);
                    xpp = (xp = x.parent) == null ? null : xp.parent;
                }
                if (xp != null) {
                    xp.red = false;
                    if (xpp != null) {
                        xpp.red = true;
                        root = rotateLeft(root, xpp);
                    }
                }
            }
        }
    }
}

```

resize重新计算数组大小，首先判断元素组是否为空即是否为新建数组，不为空的话直接将元素组扩大（不超过最大容量的，查过最大容量返回最大的整数），如果原数组为空，判断threshold是否大于零，**当数组长度为0时，该值是通过初始化方法进行赋值，是根据计算得到，这里会保证HashMap为2的幂次方**。如果初始化方法不带参数，会直接拿默认值进行初始化。**最后如果原数组不为空代表这是扩容操作，会将原数组的每个元素的每个节点重新计算在数组中的位置。**

扩容计算中e.hash & oldCap取与计算，这里扩容操作是将原数组扩大为原数组2倍。hash值为原数组长度的奇数倍的值都会放到扩容后的高位，偶数倍的值会到低位。因为容量是2的幂次方，只有对原数组取与为1才会是基数倍（这里理解为数组容量为2的幂次方，二进制数中只有一位为1其它的均为0。它左侧为会是的的2的幂次方倍注定为偶数倍，右侧会是余数，只有该位为1才会是奇数倍）。

```java
//重新计算大小，扩容或者新建数组
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    //原始数组存在且长度大于0情况（扩容）
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    //原始数组不存在，threshold存在，构造函数带参数的情况。
    else if (oldThr > 0) 
        newCap = oldThr;
    else {               
        //该情况是针对无参构造函数
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)//-----------------空元素扩容计算-------------------//
                    //新建数组进行散列计算（取余）
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)//-----------------树元素进行扩容后重新计算位置----------------------//
                    //树元素进行扩容后重新计算位置（）
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {     //-----------------链表扩容计算-------------------//
                        next = e.next;
                        //判断是高位数组还是低位数组元素，判断去掉余数后是奇数倍还是偶数倍，以此确定是低位还是高位
                        if ((e.hash & oldCap) == 0) {
                            //低位链表增加长度
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                             loTail = e;
                        }
                        else {
                            //高位链表增加长度
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                       //低位赋值
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        //高位赋值
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

//扩容操作，插入元素时可能会达到数组总容量的扩容比例（扩容因子），此时进行扩容操作（将数组扩大为原先两倍-不考虑扩容极限情况）。此时需要将原数组元素重新计算位置，扩容操作可能会使原红黑树元素高度变低，甚至达到树转链表的节点。这时需要进行树转节点操作。
final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
            TreeNode<K,V> b = this;
            // Relink into lo and hi lists, preserving order
            TreeNode<K,V> loHead = null, loTail = null;
            TreeNode<K,V> hiHead = null, hiTail = null;
            int lc = 0, hc = 0;
            for (TreeNode<K,V> e = b, next; e != null; e = next) {
                next = (TreeNode<K,V>)e.next;
                e.next = null;
                if ((e.hash & bit) == 0) {
                    if ((e.prev = loTail) == null)
                        loHead = e;
                    else
                        loTail.next = e;
                    loTail = e;
                    ++lc;
                }
                else {
                    if ((e.prev = hiTail) == null)
                        hiHead = e;
                    else
                        hiTail.next = e;
                    hiTail = e;
                    ++hc;
                }
            }

            if (loHead != null) {
                if (lc <= UNTREEIFY_THRESHOLD)
                    tab[index] = loHead.untreeify(map);
                else {
                    tab[index] = loHead;
                    if (hiHead != null)
                        loHead.treeify(tab);
                }
            }
            if (hiHead != null) {
                if (hc <= UNTREEIFY_THRESHOLD) //扩容后树元素长度小于树转换成链表的阈值，进行树到链表的转换
                    tab[index + bit] = hiHead.untreeify(map);
                else {//扩容后树元素长度大于树转换成链表的阈值，进行链表到树的转换
                    tab[index + bit] = hiHead;
                    if (loHead != null)
                        hiHead.treeify(tab);
                }
            }
        }

```

HashMap的删除操作，删除操作和树的插入操作类似。不同的时删除操作不会增加元素，就不涉及链表到树的转换，也不会涉及扩容问题。但是会减少元素对应集合的元素数量，这样可能会存在树到链表的转换。



```java
//入口，返回删除的元素，可为空
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }

//节点删除，会先查找该节点，然后进行删除操作
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))//判断是否为删除节点、注意碰撞，要用equals比较
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);//树节点的查找
                else {
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);//链表节点查找
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);//树节点删除
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

//树节点删除操作
final void removeTreeNode(HashMap<K,V> map, Node<K,V>[] tab,
                                  boolean movable) {
            int n;
            if (tab == null || (n = tab.length) == 0)
                return;
            int index = (n - 1) & hash;
            TreeNode<K,V> first = (TreeNode<K,V>)tab[index], root = first, rl;
            TreeNode<K,V> succ = (TreeNode<K,V>)next, pred = prev;
            if (pred == null)
                tab[index] = first = succ;
            else
                pred.next = succ;
            if (succ != null)
                succ.prev = pred;
            if (first == null)
                return;
            if (root.parent != null)
                root = root.root();
            if (root == null || root.right == null ||
                (rl = root.left) == null || rl.left == null) {
                tab[index] = first.untreeify(map);  //插入后树节点转换成链表
                return;
            }
            TreeNode<K,V> p = this, pl = left, pr = right, replacement;
            if (pl != null && pr != null) {
                TreeNode<K,V> s = pr, sl;
                while ((sl = s.left) != null) // find successor
                    s = sl;
                boolean c = s.red; s.red = p.red; p.red = c; // swap colors
                TreeNode<K,V> sr = s.right;
                TreeNode<K,V> pp = p.parent;
                if (s == pr) { // p was s's direct parent
                    p.parent = s;
                    s.right = p;
                }
                else {
                    TreeNode<K,V> sp = s.parent;
                    if ((p.parent = sp) != null) {
                        if (s == sp.left)
                            sp.left = p;
                        else
                            sp.right = p;
                    }
                    if ((s.right = pr) != null)
                        pr.parent = s;
                }
                p.left = null;
                if ((p.right = sr) != null)
                    sr.parent = p;
                if ((s.left = pl) != null)
                    pl.parent = s;
                if ((s.parent = pp) == null)
                    root = s;
                else if (p == pp.left)
                    pp.left = s;
                else
                    pp.right = s;
                if (sr != null)
                    replacement = sr;
                else
                    replacement = p;
            }
            else if (pl != null)
                replacement = pl;
            else if (pr != null)
                replacement = pr;
            else
                replacement = p;
            if (replacement != p) {
                TreeNode<K,V> pp = replacement.parent = p.parent;
                if (pp == null)
                    root = replacement;
                else if (p == pp.left)
                    pp.left = replacement;
                else
                    pp.right = replacement;
                p.left = p.right = p.parent = null;
            }

            TreeNode<K,V> r = p.red ? root : balanceDeletion(root, replacement);//平衡删除

            if (replacement == p) {  // detach
                TreeNode<K,V> pp = p.parent;
                p.parent = null;
                if (pp != null) {
                    if (p == pp.left)
                        pp.left = null;
                    else if (p == pp.right)
                        pp.right = null;
                }
            }
            if (movable)
                moveRootToFront(tab, r);
        }

//平衡删除，参照平衡插入
static <K,V> TreeNode<K,V> balanceDeletion(TreeNode<K,V> root,
                                           TreeNode<K,V> x) {
    for (TreeNode<K,V> xp, xpl, xpr;;)  {
        if (x == null || x == root)
            return root;
        else if ((xp = x.parent) == null) {
            x.red = false;
            return x;
        }
        else if (x.red) {
            x.red = false;
            return root;
        }
        else if ((xpl = xp.left) == x) {
            if ((xpr = xp.right) != null && xpr.red) {
                xpr.red = false;
                xp.red = true;
                root = rotateLeft(root, xp);//旋转平衡
                xpr = (xp = x.parent) == null ? null : xp.right;
            }
            if (xpr == null)
                x = xp;
            else {
                TreeNode<K,V> sl = xpr.left, sr = xpr.right;
                if ((sr == null || !sr.red) &&
                    (sl == null || !sl.red)) {
                    xpr.red = true;
                    x = xp;
                }
                else {
                   if (sr == null || !sr.red) {
                       if (sl != null)
                            sl.red = false;
                        xpr.red = true;
                        root = rotateRight(root, xpr);
                        xpr = (xp = x.parent) == null ?
                            null : xp.right;
                    }
                    if (xpr != null) {
                        xpr.red = (xp == null) ? false : xp.red;
                        if ((sr = xpr.right) != null)
                            sr.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateLeft(root, xp);//旋转平衡
                    }
                    x = root;
                }
            }
        }
        else { // symmetric
            if (xpl != null && xpl.red) {
                xpl.red = false;
                xp.red = true;
                root = rotateRight(root, xp);//旋转平衡
                xpl = (xp = x.parent) == null ? null : xp.left;
            }
            if (xpl == null)
                x = xp;
            else {
                TreeNode<K,V> sl = xpl.left, sr = xpl.right;
                if ((sl == null || !sl.red) &&
                    (sr == null || !sr.red)) {
                    xpl.red = true;
                    x = xp;
                }
                else {
                    if (sl == null || !sl.red) {
                        if (sr != null)
                    		sr.red = false;
                	    xpl.red = true;
                        root = rotateLeft(root, xpl);
                        xpl = (xp = x.parent) == null ?
                            null : xp.left;
                    }
                    if (xpl != null) {
                        xpl.red = (xp == null) ? false : xp.red;
                    if ((sl = xpl.left) != null)
                        sl.red = false;
                    }
                    if (xp != null) {
                        xp.red = false;
                        root = rotateRight(root, xp);//旋转平衡
                    }
                    x = root;
                }
            }
        }
    }
}
```

get方法，用于获取节点。由于存在碰撞的可能行，这里不止需要判断hash是否相等，还需要判断是否存在碰撞的可能行，使用equals方法进行判断

```java
//入口
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
    
//获取节点
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))//比较节点
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);//元素为红黑树进行查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);//链表形式查找
        }
    }
    return null;
}

final TreeNode<K,V> getTreeNode(int h, Object k) {
    return ((parent != null) ? root() : this).find(h, k, null);//树节点进行查找
}

final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
      //hash比较，注意碰撞
        if ((ph = p.hash) > h)
            p = pl;
        else if (ph < h)
            p = pr;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        else if ((kc != null ||
                 (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        else
            p = pl;
    } while (p != null);
    return null;
}
```

HashMap相对于HashTable少了线程安全。HashMap数据结构更加的复杂，HashTable以Entry存放信息，没有红黑树这种复杂结构，HashTable每个方法都带有synchronized关键字，所有的方法都是互斥，能够很好的保证线程安全，但是这样会导致性能比较低，不利于高并发操作。ConcurrentHashMap相对于HashTable它不是直接对方法进行加锁，而是对代码快进行加锁，锁的粒度更加的细，这样每次处理只需要加锁对应的代码快即可，能提升效率。ConcurrentHashMap的数据结构与HashMap类似，也正是这样才是分段加锁成为可能。