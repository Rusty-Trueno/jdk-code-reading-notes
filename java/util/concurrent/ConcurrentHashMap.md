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

> ConcurrentHashMap的扩容
1. 整体思路：
* 计算每个线程可以处理的桶区间，默认为16。
* 初始化临时变量nextTable，扩容2倍。
* 死循环，计算下标，完成总体判断。
* 如果桶内有数据，则同步地进行节点的转移，通常链表会拆成高位节点和低位节点2中类型。
```
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        /**
         * 哈希表的扩容
         */
        int n = tab.length, stride;
        /**
         * 如果当前是多核cpu，则将stride赋值为（哈希表的长度/8）/cpu的核心数，
         * 否则将stride赋值为当前哈希表的长度，并判断stride的大小是否小于16，
         * 如果小于16，则就使用16，这里的目的是让每个CPU处理的桶一样多，
         * stride可以理解为“步长”，有n个位置是需要进行迁移的，
         * 将这n个任务分为多个任务包，每个任务包有stride个任务
         */
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range
        if (nextTab == null) {            // initiating
            /**
             * 初始化nextTab，
             * 创建原哈希表长度为原来长度2倍的哈希数组，
             * 并将nextTab指向这个新建的哈希数组
             */
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                /**
                 * 如果扩容失败，sizeCtl使用int的最大值
                 */
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            /**
             * 更新成员变量
             * transferIndex指向的是原数组最后的位置
             */
            nextTable = nextTab;
            transferIndex = n;
        }
        /**
         * 新的哈希表的长度
         */
        int nextn = nextTab.length;
        /**
         * 创建一个fwd节点，即正在被迁移的节点，
         * 这个构造方法会生成一个Node，这个Node的哈希值为MOVED，
         * 后面，当原数组中i位置处的节点完成迁移工作后，
         * 就会将位置i处设置为这个fwd节点，用来告诉其他线程该位置处理过了，
         * 因此，它相当于是一个标志。
         */
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        /**
         * advance指的是做完了一个位置的迁移工作，可以准备做下一个位置的了
         */
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                /**
                 * 如果可以“前进”
                 */
                int nextIndex, nextBound;
                /**
                 * 对i减一，判断是否≥bound（正常情况下，如果大于bound不成立，
                 * 说明该线程上次领取的任务已经完成了，那么，需要在下面继续领取任务）
                 * 如果对i减一≥bound（还需要继续做任务），或者完成了，修改推进状态
                 * 为false，不能推进了。任务成功后修改其推进状态为true。
                 */
                if (--i >= bound || finishing)
                    advance = false;
                /**
                 * 将transferIndex赋值给nextIndex，
                 * 一旦transferIndex≤0，则说明原数组的所有位置都有相应的线程去处理了
                 */
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    /**
                     * 通过CAS将transferIndex与nextIndex去比较，
                     * 如果二者相等，则当前线程获得对transferIndex修改的权力，
                     * 将其修改为nextBound，如果nextIndex比“步长”stride要大，
                     * 则将nextBound设置为nextIndex-stride，否则将nextBound设置为0（可见，是从后往前进行的转移）
                     */
                    //更新这次迁移的边界（当前线程可处理的最小下标）
                    bound = nextBound;
                    //将i设置为当前位置-1（初次对i赋值，这个就是当前线程可以处理的当前区间的最大下标）
                    i = nextIndex - 1;
                    //停止前进，退出while循环
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                /**
                 * 判断扩容是否结束
                 */
                int sc;
                if (finishing) {
                    /**
                     * 如果完成了扩容，
                     * 将nextTable置空，
                     * 将table更新为扩容之后的哈希表
                     * 更新阈值为扩容后大小的0.75倍
                     */
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                /**
                 * 如果扩容尚未结束，但是已经无法领取区间了，
                 * 则当前线程需要尝试退出方法
                 */
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    /**
                     * 在helpTransfer中，可知，每当有线程参与迁移时，都会将sizeCtl+1，
                     * 现在，当前线程已经完成了迁移，因此，需要尝试将sizeCtl-1，
                     * 这里采用CAS的方式，避免多个线程同时对sizeCtl修改导致的冲突
                     */
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            /**
             * >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>下面的逻辑是还没有完成转移任务的情况
             */
            /**
             * 如果i处是空的，
             * 则放入刚刚初始化的ForwardingNode节点占位，
             * 说明该位置上的节点已经被迁移过了，
             * 如果放置成功了，可以继续前进，失败了则不能
             */
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            /**
             * 如果当前位置是一个ForwardingNode，代表该位置已经迁移过了
             */
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                /**
                 * 如果当前位置已经存在着节点，
                 * 则需要进行转移操作。
                 * 对哈希数组当前位置节点加锁，
                 * 开始处理数组该位置处的迁移工作，
                 * 加锁是为了防止其他线程putVal的时候向当前位置的链表插入数据
                 */
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        /**
                         * 如果当前i下标处的节点和f相同，
                         */
                        Node<K,V> ln, hn;
                        //头结点的哈希值＞0，说明是链表的Node节点
                        if (fh >= 0) {
                            /**
                             * 和HashMap类似，重新哈希后的位置，要么是最高位为0，要么是最高位为1
                             */
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                /**
                                 * 从链表的头结点开始遍历，
                                 * 依次根据节点的哈希值计算出，
                                 * 节点在新的哈希数组中是高位还是低位，
                                 */
                                int b = p.hash & n;
                                if (b != runBit) {
                                    /**
                                     * 如果当前节点和runBit对应的节点不在同一个位置（一个是高位，一个是低位），
                                     * 则将runBit重新赋值为当前节点的位置b，并将lastRun指向当前节点p
                                     */
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                /**
                                 * 如果原链表的最后一个节点是低位节点，
                                 * 则将ln指向该节点，并将hn指向空
                                 */
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                /**
                                 * 如果原链表的最后一个节点是高位节点，
                                 * 则将hn指向该节点，并将ln指向空
                                 */
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                /**
                                 * 从原哈希表当前位置的第一个节点开始遍历，
                                 * 一直遍历到lastRun节点，因为lastRun节点后面的节点，
                                 * 要么都是高位节点，要么都是低位节点
                                 * （因为上面对lastRun节点的更新，只有和上一个节点不属于同一类型的才发生，
                                 * 所以，lastRun节点后面的节点都是和lastRun节点同一类型的，
                                 * 因此，也就没必要继续调整链表结构了）
                                 */
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                /**
                                 * 如果当前节点是低位节点，
                                 * 则新建一个节点，采用头插法，将该节点的后继节点指向ln，
                                 * 同样，如果该节点是高位节点，也采用头插法，
                                 * 将该节点指向hn。
                                 */
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            /**
                             * 分别将高位的头结点，低位的头结点，
                             * 插入到新的哈希表中。
                             * 并在旧哈希表的i位置插入一个fwd节点，用于占位，
                             * 表明该节点已经被迁移过了，其他线程看到，则不会再迁移了
                             * 设置advance为true，可以继续前进
                             */
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {
                            /**
                             * 如果f是红黑树节点，
                             */
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                /**
                                 * 遍历链表
                                 */
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    /**
                                     * 如果当前节点是低位节点，
                                     * 则将当前节点以尾插法插入到lo链表上，
                                     * 并记录低位节点的个数
                                     */
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    /**
                                     * 同样， 如果当前节点是高位节点，
                                     * 则将当前节点以尾插法插入到hi链表上，
                                     * 并记录高位节点的个数
                                     */
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            /**
                             * 如果低位节点的个数≤6，则将低位节点链表转换为普通节点的链表，
                             * 否则的话，如果高位节点个数非零（说明原链表被分割了），
                             * 则创建一个新的树，如果高位节点个数为零（说明原链表保持不变），
                             * 则直接用原来的树t。
                             *
                             * 高位节点的判断完全类似
                             */
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            //将低位节点链表放进新的表
                            setTabAt(nextTab, i, ln);
                            //将高位节点链表放进新的表
                            setTabAt(nextTab, i + n, hn);
                            //将旧的哈希表的相应位置设置成占位符（说明当前位置的节点已经被转移过了）
                            setTabAt(tab, i, fwd);
                            //继续向后推进
                            advance = true;
                        }
                    }
                }
            }
        }
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

