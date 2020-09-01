# ConcurrentHashMap

## 空间分配

> sizeCtl在不同数值下的多个含义：
1. 默认为0，用来控制table的初始化和扩容操作；
2. -1代表table正在初始化，
3. -N表示有N-1个线程正在进行扩容操作
4. 其余情况：如果table未初始化，则表示table要初始化的大小。
如果table初始化完成，表示table的容量，默认是table大小的0.75倍（n-(n>>>2)）
5. ConcurrentHashMap在初始化时（构造函数），
如果传入了合法的初始容量，构造函数会将该数值赋给sizeCtl，
而不是为哈希表分配内存，只有当哈希表被某个线程第一次put时，才会进行内存的分配。

> initTable函数
1. 该函数是在哈希表被第一次put的时候调用的。
2. 由于可能同一时刻多个线程都在向哈希表put，并且哈希表还没有初始化，
因此都会调用该函数，该函数首先会根据sizeCtl的值是否为-1，判断当前是否有线程正在初始化哈希表，
如果有的话，则当前线程先让出CPU。如果当前没有线程正在初始化哈希表，则通过CAS，实现原子性地
获取对当前哈希表初始化的权力，并可以为当前哈希表分配物理内存。
```
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            /**
             * 如果表为空，或者表的长度为0，则一直循环，
             * 如果sizeCtl＜0，则说明当前哈希表正在初始化，
             * 则需要当前线程让出CPU时间，
             * 否则，进行CAS判断（传入当前对象，和当前对象在内存中的偏移量（SIZECTL）
             * 并将偏移量处的值和期望值，此时为sc去比较，如果相等，
             * 就把目标值-1赋值给offset位置的值，方法返回true），即，如果sizeCtl的大小没有被其他线程改变，
             * 则当前线程可以获得对sizeCtl的改变权力，则将sizeCtl的值赋值为-1，表示当前有线程在修改此表
             */
            if ((sc = sizeCtl) < 0)
                Thread.yield(); // lost initialization race; just spin
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                /**
                 * 当前线程初始化哈希表，
                 * 先判断，当前哈希表是否为空，或者当前哈希表的长度是否为0，
                 * 如果是的话，则获取原sizeCtl的大小，
                 * 如果原来就>0，则说明在创建哈希表的时候，向构造函数中传递了初始大小，
                 * 如果原来<0，则说明able正在初始化，
                 * 或者-N表示有N-1个线程正在进行扩容操作，
                 * 这2中情况则将哈希表的大小设置为默认的大小，
                 * 新建Node数组，数组的大小为当前容量的大小，
                 * 将当前的table替换为新建好的table，并修改sc为（n-(n>>>2)），即表容量的0.75倍
                 *
                 * 最后，将sizeCtl设置为(n-(n>>>2))
                 */
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        //双重验证
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        //将sc的值改变为n-0.25n，也就是0.75n
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
## 基本数据结构

> ForwardingNode
1. ForwardingNode节点是一种临时节点，在扩容进行中才会出现，hash值固定为-1，
并且它不存储实际的数据，如果旧数组的一个hash桶中全部的节点都迁移到新数组中，
旧数组就在这个hash桶中放置一个ForwardingNode。读操作或者迭代读时碰到ForwardingNode
时，将操作转发到扩容后的新的table数组上去执行，写操作碰见它时，则尝试帮助扩容。

```
```


## 基本方法

> tabAt函数
```
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
        /**
         * 获取哈希表指定位置的节点，
         * 通过getObjectVolatile方法可以直接获取指定内存的数据，
         * 保证了每次拿到的数据都是最新的
         */
        return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
    }
```
1. table既然是用volatile修饰的，那么为什么还要用native方法去查看内存？
* 答：table确实是volatile修饰的，但是这只能表示数组的引用是内存可见的，
但是数组每个位置指向的内容并不保证是内存可见的，所以，如果直接去用下标
访问数组中的某个元素不能保证访问的是最新的值。因此需要使用native方法，
直接取内存中取，这样一定是最新的数据。

> 

