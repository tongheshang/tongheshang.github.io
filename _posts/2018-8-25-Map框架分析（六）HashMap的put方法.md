---

layout: post

title: Map框架分析（六）HashMap的put方法

date: 2018-8-24

tags: Java基础

---

本文基于JDK 1.8。在阅读源码的过程中，发现自己很多地方不能自洽，应该是对源码的理解有很大问题，本文自做记录不作参考，切勿以本文作参考！

### 相关知识点
- [Map](https://www.heshengbang.tech/2018/06/Map框架分析-二-Map接口分析/)
	- 内部接口
	- 方法
- [AbstractMap](https://www.heshengbang.tech/2018/06/Map框架分析-三-AbstractMap抽象类分析/)
	- 内部类
	- 方法
- HashMap
	- [HashMap中的内部类](https://www.heshengbang.tech/2018/06/Map框架分析-四-HashMap的内部类/)
		- [HashMap的内部类TreeNode](https://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)
	- HashMap中的方法和成员变量
		- [HashMap中的成员变量](https://www.heshengbang.tech/2018/06/Map框架分析-十-HashMap中的成员变量/)
		- [HashMap中的方法](https://www.heshengbang.tech/2018/06/Map框架分析-五-HashMap的方法/)
            - [HashMap的put方法](https://www.heshengbang.tech/2018/06/Map框架分析-九-HashMap的内部类TreeNode/)
            - [HashMap的resize方法](https://www.heshengbang.tech/2018/06/Map框架分析-七-HashMap的resize方法/)
            - [HashMap的树化与反树化](https://www.heshengbang.tech/2018/06/Map框架分析-八-HashMap的树化与反树化/)

### 概述
  本文原起自今年6月份写HashMap的合集，是其中一篇，笔者当时写到一半就弃了，挖坑未填心中一直不安，近一个月又遭遇各种事情，心气难平。昨天思索再三，决定重新开始夯实基础，既然夯实基础那过去挖的坑必然要填上。HashMap这个数据结构在Java这门语言中的重要性不言而喻，因此直接写吧，有不明情况的同学，可以参考以上链接。  
  作为数据结构，HashMap自然包含存数据和取数据这两个基本功能。对Map略有了解的同学就应该知道，HashMap中最基本的元素时Entry，HashMap是Entry的集合，其本身就是Java集合框架中的一个异类，但是如果从Entry这个粒度来看，HashMap其实和List/Set等数据结构没区别。如同所有的集合框架类一样，HashMap是一个可变大小的集合，它存储的是Entry，俗称键值对，它允许使用者直接使用键值对的形式存取元素，而不是像一般的集合类那样整存整取。既然是存取两种操作，HashMap就有不同的实现方式，put就是最基本的存操作。HashMap还有其他一些存操作(putAll/putIfAbsent)，但都是通过put中的方法来实现的。下面展示的是put的源码：
```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
  从源码中可以看出，put方法没有执行任何操作，它只是将自己的参数传到另外一个私有方法putVal()中去。这个putVal()在HashMap的源码中出现过四次，分别是put()/putAll()/putIfAbsent()/readObject()中调用了该方法。四次虽然不多，但是对于整个HashMap实现的功能来说却至关重要。下面将展示putVal方法的源码，在阅读源码前请明确以下几点内容：
- 本文基于JDK8，因此HashMap的结构是哈希数组+链表+红黑树
- 了解HashMap的内部类Node/TreeNode的基本成员变量、方法及方法对应要实现的功能
- 链表的树化

以下是HashMap的putVal()方法的源码：
```java
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
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
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
  putVal()方法有五个参数，第一个是key的哈希值，第二第三个分别是key/value不用细说，最后两个boolean型的onlyIfAbsent、evict。onlyIfAbsent如果为true则表示不修改HashMap中已经存在的key对应的value，如果为false，则直接用给的参数value替换已经存在的value。evict如果为false，则表示HashMap当前处于创建模式。

### put方法详细步骤
- 获取key的哈希值，源码如下：
```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
代码的基本逻辑是：
	- 判断key是否为null，如果为null，哈希值就为0
	- 调用key自身的hashcode()方法，该方法是继承自Object的native方法，每个对象都有
	- 将key的哈希值无符号右移16位，左边用0补位得到值
	- 将上一步的值与key的哈希值取异或运算，获得一个32位的哈希值
		- 高16位的值在异或的运算下不会改变，因为补位0与(0/1)做异或，值仍是(0/1)
		- 低16的结果是高16位与低16位的异或结果
		- 这种求哈希的方式是jdk8新增的

- 如果哈希桶数组为null或者长度为0，则进行扩容
- 获取到哈希桶数组的大小n
- 通过&操作取模，这个模的结果即key在哈希桶数组上的索引位置。这段代码源码如下：
```java
    if ((p = tab[i = (n - 1) & hash]) == null)
                tab[i] = newNode(hash, key, value, null);
```
这段代码的基本逻辑如下：
	- 通过哈希桶数组容量n-1与哈希值取模，模的结果一定是0 ~ n-1。至此，在哈希桶数组上获得一个索引值。
	- 判定索引值对应的哈希桶是否为空
	- 如果哈希桶为null就创建一个新的节点作为该索引的哈希桶
		- 新创建的节点的哈希值、key、value均被指定
		- 新节点的next节点为null，因为它后面暂时还没有链表节点需要去连接
		- 该处创建的节点为Node节点，区别于树节点TreeNode

- 索引对应的哈希桶p不为空，则进入下一个大的代码模块
- 判定如果哈希桶p对应的哈希值与参数哈希值相等，并且，p的key与参数key是同一个对象(==)或者值相等(equals)，将p的引用保留。代码如下：
```java
    if (p.hash == hash &&
    	((k = p.key) == key ||(key != null && key.equals(k))))
        e = p;
```

- p的哈希值与参数哈希值不相等，或者p的key与参数key既不是同一个对象也不是值相等则判定节点p是否为一个树节点，代码如下：
```java
    else if (p instanceof TreeNode)
        e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
```
这段代码首先判定了p节点是否为树节点，如果为树节点就调用TreeNode的putTreeVal方法将put的key/value传进去，需要说明以下几点：
	- this是指当前这个HashMap实例
	- tab是指HashMap实例的哈希桶数组
	- 完成TreeNode的putTreeVal方法后，会返回一个TreeNode，不过由于TreeNode继承自LinkedHashMap.Entry，而LinkedHashMap.Entry又继承自HashMap.Node，所以e可以接收TreeNode这个返回值
	- TreeNode#putTreeVal方法会在后面详细展开写，这里就不讲里面的具体实现步骤

- 参数key/value在哈希桶数组上的位置，既不在数组上，也不在树节点上，那只能是在哈希桶数组上连接的链表上
- 循环遍历哈希桶连接的链表，将参数哈希值和参数key和链表上的节点的哈希值和参数key进行比较，如果找到就跳出循环，如果找不到就创建节点，如果创建节点后链表大小超过树化临界值就将链表转换为红黑树，代码如下：
```java
	for (int binCount = 0; ; ++binCount) {
		if ((e = p.next) == null) {
			p.next = newNode(hash, key, value, null);
			if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
				treeifyBin(tab, hash);
				break;
			}
		if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
			break;
		p = e;
	}
```
这段代码基本的逻辑如下：
	- 开启一次没有结束条件的遍历（在JDK8之前，这样写会有死循环的风险）
	- 判定哈希桶连接的链表的节点是否为null，如果为null就使用给定参数创建一个新的Node节点，并将新节点的引用赋值给哈希桶的next，使其相互关联起来
        - 判定创建节点后，哈希桶链接的链表大小是否超过了树化临界点，JDK在此处给出了一个难得的注释，“-1 for 1st”，意思是-1是因为遍历是从0开始的
        - 如果创建节点后<b>大于等于</b>树化临界值就将链表转化为红黑树，传入参数为哈希桶数组和哈希值，然后跳出循环
	- 如果哈希桶连接的链表的节点不为null，就比较其和参数的哈希值和key是否相等，判定条件与上面一致，不详述
		- 如果相等就跳出循环
	- 这段循环代码执行结束后，key/value被插入或已存在的位置依然将被记录下来

- key/value根据key找到一个已经存在的位置
- put调用的putVal()onlyOfAbsent永远为false，则节点将肯定被赋值为参数value，这段代码如下：
```java
    if (e != null) { // existing mapping for key
		V oldValue = e.value;
		if (!onlyIfAbsent || oldValue == null)
			e.value = value;
		afterNodeAccess(e);
		return oldValue;
	}
```
关于这段代码有以下几点需要注意：
	- e != null说明key在当前这个HashMap实例中已经存在，即调用containsKey会返回true
	- onLyIfAbsent为false，即无论怎样，新的value都将替换旧value
	- afterNodeAccess是留给LinkedHashMap的接口
	- 旧的value将被返回

- 如果上面没有return并结束putVal方法就证明key/value是通过新建节点插入到HashMap实例的，这通常会意味着：
    - 将HashMap的结构发生了改变，因此结构被修改次数+1
    - 将HashMap当前大小+1，如果大小超过了扩容阈值就扩容
	- return null;结束putVal方法，也结束整个put方法


以上是整个put方法最直接的步骤，与其强相关的还有两个方法，分别是：`TreeNode#putTreeVal`和`treeifyBin`，将在接下来详细解释其基本步骤及逻辑。

### TreeNode#putTreeVal
- TreeNode是HashMap的内部类，继承自LinkedHashMap.Entry，而LinkedHashMap.Entry又继承自HashMap.Node，这么做的目的是方便LinkedHashMap继承和扩展HashMap的功能，如果觉得绕，可以暂时认为TreeNode继承自HashMap.Node
- `TreeNode#putTreeVal`是TreeNode的成员方法，意即插入新的树节点到红黑树上，在方法上面有JDK给出的注释"Tree version of putVal."，该注释表达的意思非常明显。
- 方法的源码如下：
```java
	final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v) {
            Class<?> kc = null;
            boolean searched = false;
            TreeNode<K,V> root = (parent != null) ? root() : this;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph; K pk;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((pk = p.key) == k || (k != null && k.equals(pk)))
                    return p;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0) {
                    if (!searched) {
                        TreeNode<K,V> q, ch;
                        searched = true;
                        if (((ch = p.left) != null &&
                             (q = ch.find(h, k, kc)) != null) ||
                            ((ch = p.right) != null &&
                             (q = ch.find(h, k, kc)) != null))
                            return q;
                    }
                    dir = tieBreakOrder(k, pk);
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
```
方法的基本逻辑就是遍历红黑树，比对参数key/value和树节点的哈希值和Key，如果相同就表示找到了，返回节点的引用，如果没有找到就插入节点，然后调整树结构

- 方法的参数有五个
	- `HashMap<K,V> map` 是指红黑树所在的HashMap实例
	- `Node<K,V>[] tab` 是指红黑树头结点所在的哈希桶数组
	- h是指参数key对应的哈希值，哈希值的算法上面有，这里不赘述
	- K是指参数key
	- V是指参数value

- 详细步骤
	- 获取节点所在红黑树的根节点，代码如下：
	```java
    	TreeNode<K,V> root = (parent != null) ? root() : this;
    ```
      这段代码最终的目的是获取节点所在的红黑树的根节点，它为了达成这个目的，用三目运算符来完成操作，判断这个节点是否有父节点：
		- 如果有父节点，则它不是自己所在的红黑树的根节点，则调用root方法去向上搜索父节点
		- 如果它的父节点为null，则它自己就可能是所在的红黑树的根节点。
	- 无论上一步的具体情况如何，最终都可以获取到节点所在的红黑树的根节点。以根节点为起点开始遍历红黑树
	- 树节点的哈希值和参数哈希值比较，代码如下：
	```
    	int dir, ph; K pk;
		if ((ph = p.hash) > h)
			dir = -1;
		else if (ph < h)
			dir = 1;
		else if ((pk = p.key) == k || (k != null && k.equals(pk)))
			return p;
    ```
    这段代码实现的主要逻辑是：  
		- 树节点的哈希值比参数哈希值大，则dir = -1，这意味着接下来，遍历会进入树节点的左子树
		- 树节点的哈希值比参数哈希值小，则dir = 1，这意味着接下来，遍历会进入树节点的右子树
		- 倘若树节点的哈希值和参数哈希值大小相等，同时树节点的key和参数key要么是同一个对象（==），要么值相等（equal），则此时是找到了参数key/value中的key已存在HashMap中，即containKey返回true，此时直接返回找到的树节点
    - 排出上面三种情况，其实还有另外一种情况，但这种情况比较复杂，所以单独拎出来说，代码如下：
    ```
        else if ((kc == null && (kc = comparableClassFor(k)) == null) || (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true;
                if (((ch = p.left) != null && (q = ch.find(h, k, kc)) != null) || ((ch = p.right) != null && (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }
    ```
	当程序运行到这段代码时，只能说明一个问题，树节点的哈希值和参数哈希值相等，但是树节点的key和参数key又不是同一个类型的变量（因此，既无法进行==比较，也无法进行equal比较），因此采用了以下逻辑：  
		- 如果参数key的类型没有实现Comparable接口，进入条件代码块
		- 如果参数key的类型实现了Comparable接口
			- 判断树节点的类型是否实现了Comparable接口，如果没实现则进入条件代码块，如果实现了则参数key和树节点key进行比较，相等则进入代码块，不相等则dir会得到一个非0数字，下一步根据这个非0数字决定继续遍历左子树还是右子树
		- 进入条件体内有两种情况：
			- 参数key没有实现Comparable接口
            - 树节点key没有实现Comparable接口
		- 在上述两种情况满足其一或者两者皆满足的情况下，根据参数哈希值，参数key，参数key的类型，在树节点的左子树中执行一次搜索，没找到就在右子树中执行搜索，如果找到符合条件的节点就返回
		- 如果上一步依然没有找到节点，此时就只有采用红黑树寻找比较中的最后的手段
			- 如果参数key和树节点任意一个为null，就采用System类的`identityHashCode`方法获取到一个哈希值，为null的key的哈希值为0，因此会比较出一个结果
			- 或者它们俩的类名相等，也采用System类的`identityHashCode`方法获取到一个哈希值，该方法是native的，不同的Object一定会求出不同的哈希值，然后对哈希值进行比较，返回比较结果
			- 如果从树节点key和参数key的类名比较出了大小，则直接返回该比较结果
		- 上一步的结果，如果小于等于0就转向左子树遍历，如果大于等于0就转向右子树继续遍历
		- 需要说明的是，如果程序运行到这段代码的时候，参数哈希值和树节点的哈希值是相等的，但是key无法正确比较，因此这种情况下，很大概率是需要新建一个树节点，然后对红黑树进行再平衡调整
    - 当上一步没有返回节点并结束程序时，这意味着程序将转向左子树或者右子树继续遍历，但是在继续遍历前需要判断子树是否为null，如果为null就需要创建节点而不是遍历，如果不为null就进行下一轮遍历比较。代码如下：
    ```java
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
    ```
    这段代码里面的基本逻辑是：
		- 判定该继续遍历的是左子树还是右子树
		- 子树的节点是否为null
			- 如果不为null，就执行新一轮的比较
		- 如果为null，就先创建树节点
		- 完善节点的前（pre）后(next)关系
		- 完善新建节点与父节点的关系，父节点与新建节点的关系
		- 将新建树节点平衡插入到红黑树中
		- 将根节点移动到树节点的链表关系的最前面
		- 返回null

- 分析完代码逻辑后，我们基本搞清楚了putTreeVal()方法的逻辑，知道它有两种返回：①key相同并已经存在的树节点，②新建树节点会返回null
- 根据我们分析的putVal()方法的逻辑来看，返回的结果最终会被用于判断内含参数key的节点是否已经存在，节点存在的情况下是否需要用新值替换旧值。
- 该方法强相关的三个树节点的成员方法是：`moveRootToFront()`，`balanceInsertion` 如果有时间建议好好再去分析这两个方法的逻辑，如果不想自己分析，可参考本文顶部的连接中TreeNode的相关内容

### treeifyBin
- treeifyBin作用是将<b>大于等于</b>树化临界值的链表转换为红黑树，方法之上有JDK给出的注释：
> Replaces all linked nodes in bin at index for given hash unless table is too small, in which case resizes instead.  
这段注释可以理解为，使用TreeNode替换给定哈希值在哈希桶数组中对应的哈希桶中的Node，除非哈希桶数组太小，因为那样还会导致扩容。

- 方法源码如下：
```java
	final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
        else if ((e = tab[index = (n - 1) & hash]) != null) {
            TreeNode<K,V> hd = null, tl = null;
            do {
                TreeNode<K,V> p = replacementTreeNode(e, null);
                if (tl == null)
                    hd = p;
                else {
                    p.prev = tl;
                    tl.next = p;
                }
                tl = p;
            } while ((e = e.next) != null);
            if ((tab[index] = hd) != null)
                hd.treeify(tab);
        }
    }
```
可以看出方法逻辑不是特别扶助，一共也就二十行不到。方法一共有两个参数，一个是哈希桶数组，另一个是哈希值。通过哈希值定位到哈希桶数组上的具体哈希桶，然后通过哈希桶的头结点，对连接在其上的节点进行操作。

- 详细步骤
	- 判定哈希桶数组是否为null或者哈希桶数组的当前大小，是否小于最小树化临界点，如果满足其中一个条件则进行扩容。源码如下：
	```java
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
                resize();
    ```
    个人觉得，这里的第一个判断条件是完全没必要的，至少在HashMap中调用treeifyBin的地方都曾判断过哈希桶数组的大小，并根据是否为空，决定是否扩容。但从另一个角度来说，作为JDK多做一次校验不过是几行机器码，不会影响执行效率，但是可以避免某些极端情况下的风险，也方便以后的扩展（treeifyBin有final修饰符，没有扩展的可能性/xk）。
		- `MIN_TREEIFY_CAPACITY`在HashMap中被定义为64 —— `static final int MIN_TREEIFY_CAPACITY = 64;`
		- 这部分代码中提到的resize()方法在HashMap中也是一个重量级的方法函数，这里不展开写，后面会有专门的篇章来写它，开发者如果觉得十分好奇，可以先自己去看看源码，试着理解一下，如果觉得逻辑复杂（实际上并不，只不过使用了众多默认初始量），可以辅以我文章顶部的链接服用。
	- 上一部分代码执行完毕后，哈希桶数组肯定不会为null
	- 判定参数哈希值，在参数哈希桶数组对应的索引位置是否为null，如果为null则结束方法，如果不为null，则进行程序下一步执行
	- 这一步将参数哈希值在哈希桶数组上对应的哈希桶的节点捞出来，以它为头结点，开始遍历，将所有链表节点转换为树节点，源码如下：
	```
	do {
    	TreeNode<K,V> p = replacementTreeNode(e, null);
        if (tl == null)
        	hd = p;
        else {
        	p.prev = tl;
            tl.next = p;
        }
        tl = p;
    } while ((e = e.next) != null);
    ```
		- 这段代码先将哈希桶中的节点转换为树节点。前面有提到过，TreeNode实际上是继承自Node，因此这部转换其实毫无难度，就是将Node的属性值赋给TreeNode继承自Node的属性。
		```java
        TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
        	return new TreeNode<>(p.hash, p.key, p.value, next);
        }
        ```
		- 判断转换后的树节点是否有pre节点：
			- 如果没有就将其当做根节点
			- 如果有pre节点，就将转换后的树节点和pre节点建立关联关系（即互相将其指为prev/next节点）
		- 将转换后的树节点暂存起来，作为下一次遍历的pre节点
		- 链表通过next，找到下一个节点，继续将Node转换为TreeNode
	- 程序执行完上一部分代码后，以Node方式组织成的链表，已完全转换为TreeNode并以prev/next关联起来，并且也找到了TreeNode最前面的节点。值得注意的是，此时的它们虽然都是TreeNode但是它们都以链表的形式存在，并没有构建出树形结构。
	- 判断TreeNode形成的链表中最前面的那个节点，是否为null
		- 如果为null，结束方法执行
		- 如果不为null，就调用它的treeify方法

- 从源码中可以看出，该方法并没有执行真正的构建红黑树的操作，而只是简单的将Node链表转换为TreeNode链表，然后用TreeNode的头结点调用treeify方法去构建红黑树。
- 和treeifyBin强相关的两个方法分别是`resize`，`TreeNode#treeify`，其中`TreeNode#treeify`完成了将TreeNode链表转换为红黑树的操作，相对更重要一点，有兴趣的可以去了解一下。

### 总结
- 至此，HashMap的put方法已经逐行捋了一遍，我屏蔽了这个过程中调用的方法逻辑，这些方法相对重要，需要单独去讲，而不是作为put方法的一部分来说。另外，两个方法对于putVal()方法来说过于重要，必须在这篇里面讲清楚，否则完全无法讲好HashMap的put方法。
	- `resize()`
	- `TreeNode#putTreeVal()`
		- `balanceInsertion()`
		- `moveRootToFront()`
	- `treeifyBin()`
		- `TreeNode#treeify()`

- HashMap的put方法中逻辑比较复杂的是`TreeNode#putTreeVal()`，这里面通过参数哈希值和参数key去寻找合适的位置将参数key和参数value，参数hash放到正确位置去，是通过对参数hash以及参数key的比较。但是某些情况下，hash值可以相等，但是key却不一定相等，不止不相等，甚至是无法进行比较。在这种情况下，JDK想方设法去比较参数Key和节点key，①判定是否实现了Comparable接口 ② 通过key的类名字进行比较。总之，按照一个逻辑最终要比较出key的一个大小还是相等，这样才能决定key是否相等，还是放到哪个位置去是合适的。