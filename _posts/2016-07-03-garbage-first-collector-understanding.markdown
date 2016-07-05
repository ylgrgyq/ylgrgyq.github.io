--- 
layout: post
title:  "Garbage First Collector 理解"
date:   2016-07-03 15:39:30 +0800
--- 

# 总览

缩写约定：

| 缩写 | 含义 |
| -----|----- |
| YGC  | Young Generation GC |
| OGC  | 针对 Old Generation 的 GC，对 G1 来说指 Mixed GC |
| FGC  | 针对整个 Heap 的 Full GC |
| STW  | Stop-The-World |

## 特点

1. G1 的 Heap 划分为多个 Region，Young Generation 和 Old Generation 都只是逻辑概念，不是物理上隔离又连续的空间
2. 适合大堆，因为不像 CMS 和 Parallel GC 在对老代进行收集的时候需要将整个老代全部收集，G1 收集老代一次只收集老代的一部分 Region
3. G1 的新老代划分不是固定的，一个新代的 Region 在被回收之后可以作为老代 Region 使用，Young Generation 和 Old Generation 大小也会随着系统运行而调整
4. G1 的新生代收集和 Parallel、CMS GC 一样是并发的 STW 收集，且每次 YGC 会将整个 Young Generation 收集
5. G1 的 Old Generation 收集每次只收集一部分 Old Region，且这部分 Old Region 是和 YGC 一起进行的，所以称为 Mixed GC
6. 和 CMS 一样，G1 也有 fail-safe 的 FGC，单线程且会做 compaction
7. G1 的 Old Generation GC (Mixed GC) 也是自带 compaction 的
8. G1 没有永久代的概念

# Young Generation

G1 的 Young Generation 逻辑上也划分为 Eden 和 Survivor。新生 Object 都是在属于 Eden 的一个 Region 上进行分配，Region 满了之后会从 available region 中再取一个新 Region 标记为 Eden 并继续将新生 object 放在里面。直到标记为 Eden 的 Region 数达到上限。到达上限后，触发 YGC。

## TLAB

既然 “新生对象都分配在 Eden”，而 Eden 是个全局的概念，应用内会申请分配内存创建新生对象的业务线程有很多，如果分配内存操作全部由这些业务线程直接去操作 Eden 就一定会产生竞争，因为属于 Eden 的 Region 是一个一个分配的，一个 Region 占满了才会去分配新的 Region。而竞争的存在就导致要用锁去保护 Eden ，才能保证多线程并发的从 Eden 分配内存不出问题。而由于分配内存这个操作会非常频繁，只是用锁去保护 Eden 会出现大量的线程去抢占这个保护 Eden 的锁。所以有了 TLAB，Thread Local Allocation Buffer， 这么个优化，去减少业务进程对保护 Eden 的锁的竞争。

Eden 中按照一定比例有一部分内存会划拨出来专门给 TLAB 使用，每个线程都有自己的 TLAB，这块内存是线程自己独占的，为的就是线程在分配内存的时候可以直接从 TLAB 上不用加锁的获取内存，只有要分配的内存较大，超出了 TLAB 范围时才需要从 Eden 中以加锁的方式获取内存，或者如果特别大超过了 Region 的 50%，会作为 Humongous Object 专门划拨 Region 存放。

## YGC

随着 Eden 内 Object 越来越多，越来越多的 available region 现在被标记为 Eden 并被占满，当标记为 Eden 的 Region 数达到上限时，会触发 YGC。

每次 YGC 时 G1 从 available region 取一个新 Region 标记为 Survivor，将当前整个 Eden 和老 Survivor 中的 live object 找出来，并根据 live object 熬过的 YGC 次数判断是将其拷贝到这个新的 Survivor 还是拷贝(晋升)到 Old。

Young Generation Object 每熬过一次 GC，age 就增长一岁。G1 会维护一个 age -> object 的 hash 表，将 age 达到目标值的 object，晋升到 Old Generation。

这个目标值一般称为 Tenuring Threshold，是根据 -XX:TargetSurvivorRatio 和 -XX:MaxTenuringThreshold 来动态计算得到的。

## PLAB

除了 TLAB 之外，还有个叫做 PLAB 的东西。YGC 时，live object 需要被拷贝到 Survivor Region 或者晋升到一个 Old Region。拷贝过程是并发的，会有多个 GC 线程一同处理，而目标 Survivor Region 和 Old Region 也是一个 Region 写满之后再分配另一个 available region 继续写。所以这些 GC 线程之间也存在竞争。所以 GC 过程中，会类似 TLAB 一样，从当前正在操作的 Region 上给这些 GC 线程都各自分配一块 Thread Local Buffer，拷贝 live object 时每个 GC 线程都是将 live object 优先拷贝到分配给自己的 Thread Local Buffer 上，这个 Thread Local Buffer 就叫做 PLAB，Promotion Lab，以避免加锁，减少竞争。

## Young Generation 大小

上面看到 YGC 触发时机是在 Eden 被占满时，而 Eden 在 Young Generation 中占比最大，也就是说 Young Generation 的大小会影响到 YGC 触发时间和频率。

有三个量会影响到 Young Generation 大小：

* -XX:G1NewSizePercent 初始 Young Generation 大小，默认 5%
* -XX:G1MaxNewSizePercent Young Generation 最大大小，默认 60%
* -XX:MaxGCPauseMillis GC 最大停顿时间，默认 200ms

MaxGCPauseMillis 会影响到 Young Generation 大小是因为 MaxGCPauseMillis 越小，留给 GC 的 STW 的时间越少，则趋向于减少 Young Generation 大小以减少 YGC STW 时间。每次 YGC 完毕，都会根据上面三个量和 G1 内部的一些统计量去计算 Young Generation 大小，然后实现 Young Generation 扩展或收缩。这个变化会提现在 GC 日志当中:
![2016-06-29 1 12 19](https://cloud.githubusercontent.com/assets/1115061/16544438/b174f442-4138-11e6-9461-867fdebbbc08.png)

上图看到 Eden 从 8008M 降低到 7936M，同样 Survivor 也有类似变化。而总 Heap 大小因为 -Xmx 和 -Xms 参数都调的 14G 所以 YGC 前后不会出现变化。

**注意：**如果 Young Generation 大小被明确规定，比如用 -Xmn 或者 -XX:NewRatio 限制，则 Young Generation 大小就不能根据 GC 实际的 Pause Time 而动态调节了，所以不要使用这类参数。上面 G1NewSizePercent 和 G1MaxNewSizePercent 规定的只是 Young Generation 范围，而不是固定的某个值。

# RSet

G1 也属于分代收集器，G1 是从逻辑上划分 Young Generation 和 Old Generation，没有从物理存储空间上将不同代隔离开 ( Region 可以在 Old 和 Young Generation 之间切换)。分代收集的好处就是将 long-live object 和 short-live object 分开收集，从而不用每次 GC 都扫描整个 Heap，降低 GC 时间。

G1 在一般意义上的分代收集的基础上更进一步，每次 Mixed GC 是整个 Young Generation 的 Region 和一部分 Old Generation Region 参与收集，一次 GC 中参与收集的所有 Region 组成的集合叫做 CSet。那么 CSet 内的 Region 上某个 Object A 被不在 CSet 内 Region 上的另一个 Object B 引用时，A 这个 Object 不能被收集掉。比如说一个在 Young Generation 的对象，被一个 Old Generation 对象引用，YGC 时这个 Young Generation 对象不能被收集掉。

所以，需要为每个 Region 维护一个列表，内容是当前 Region 之外，有哪些 Region 有指向当前 Region 的引用。这个表就是 Remember Sets，也叫 RSet. 没有这个 RSet 的话，拿 YGC 来说，每一次 YGC 在扫描完 Root 之后，都要再扫描一遍当前所有 Old Generation Region 以找出从 Old Generation 指向 Young Generation 的引用。

**注意：**看到 RSet 只会记录别的 Region 对本 Region 的引用，自己 Region 内部的引用无需 RSet 参与记录。

## RSet 内引用构建

既然 RSet 是必须要有的，接下来就看看 RSet 内有哪些引用，这些引用是如何维护的。

因为每次 YGC 都会将整个 Young Generation 都放入 CSet，不存在哪个属于 Young Generation 的 Region 不参与 YGC 的情况。所以对 Heap 上的所有 Region 来说，被 Young Generation 内 Object 的引用指向是不需要记录到 RSet 中的。于是，RSet 内需要维护的引用只有两种：

* Old-to-young refernence
* Old-to-old refernence. 

![2016-07-03 12 23 19](https://cloud.githubusercontent.com/assets/1115061/16544442/d1748c76-4138-11e6-902d-7c37b6bb0843.png)
(图片来自参考文献[1])

看到上面图中，x Region 是 Young Region，y、z 是 Old Region。每个 Region 都有个配套的 RSet，x 的 RSet 有个指向 z 的记录，因为 z 是 Old  且有指向 x 的引用。z 虽然被 x 和 y 两个 Region 上的引用指向，但因为 x 是 Young Region，所以 z 的 RSet 中只有指向 y 的记录。同样的方式分析，y 的 RSet 没有任何记录，因为 y 只有被 x 指向的引用。

## RSet 记录

Region 和 Region 之间的 popular 程度是不同的，有的 Region 有更多的引用指向，有的则会少一些。如果一个 Region 特别 popular，有大量的引用指向这个 Region，该 Region 的 RSet 所要记录的引用也更多，GC 时扫描 RSet 的时间也更长。

为了减少这种特别 popular 的 Region 的 RSet 处理时间(这里不光是能减少 GC 时间，还能减少各 GC 线程之间处理 RSet 时的不均匀性，越均匀越能发会多线程 GC 性能)，RSet 根据所属 Region “popular” 程度的不同，一共分为三种等级，sparse、fine 和 coarse。每个等级都有个 per-region-table (PRT) 用于存储引用信息。

每个 Region 实际又能被细分为最小单个 512 字节的 heap chunk，称为 card。每个 card 都有个根据它地址构造出来的全局唯一 id ，这个唯一 id 不仅是在一个 Region 中唯一，在整个 Heap 中都是唯一的，并且能根据这个 id 立即找到对应的 card。说了半天的 RSet 记录指向 RSet 所属 Region 的引用，实际就是在 RSet 中记录指向这个 Region 引用所在 card 的唯一 id。

当 RSet 处在 sparse 级别，PRT 中每个 entry 直接存引用当前 Region 所在 card 的 id，这种粒度下 RSet 扫描效率最高。当 Region popular 程度上升，指向该 Region 的引用越来越多，直接存 card id 会导致 PRT 过大。当 sparse PRT 内存储引用到达限制后，升级为 fine 级别的 PRT。

fine 级别的 PRT 中每个 entry 不再直接存储 card id，具体存储内容拿下图来说。B 有个指向 A 的引用，当 A 的 RSet 升级到 fine 级别时，A 为 B 单独创建一个 Bitmap，将指向这个 Bitmap 的引用和指向 B Region 的引用一起存入 A 的 PRT 的 entry 中。

![2016-07-01 7 10 10](https://cloud.githubusercontent.com/assets/1115061/16544439/b42716f2-4138-11e6-8c8b-aa80b00ba319.png)

并且在 B 对应的这个 Bitmap 会标识出 B 中哪个 card 有指向 A 的引用。从这里描述能看出来 fine 级别的 PRT 对引用的记录更间接一些，所以扫描的时候相对更慢一些。

当 Region popular 程度继续升高，还是按上图说的，B 指向 A 的引用越来越多，B 对应的 Bitmap 达到上限之后，A fine 级别的 PRT 内 B 相关的 entry 会被删除，取而代之的是使用 coarse 级别的 PRT 来记录 B 指向 A 的引用。

coarse 级别的 PRT 实际就是个 Bitmap，该 Bitmap 上每个 bit 代表当前 Heap 的一个 Region。拿上面例子来说就是将 A 的 coarse PRT 的 Bitmap 中代表 B 的 bit 置位，并且不再记录 B 中到底是哪个 card 含有指向 A 的引用。这也能看出来 coarse 级别 PRT 扫描起来耗时最大，必须扫描整个 B region 才能找到所有指向 A 的引用。

除了上面结构之外，跟 RSet 相关的还有个全局的 card table，也是个基于 Bitmap 的结构，用于在 GC 时扫描 RSet 阶段记录已经扫过的 card，避免重复扫已经扫过的 card。每轮 GC 后这个 card table 会被删除。

下面是一次真实 GC 记录，其中 Update RS, Scan RS 就是处理 RSet 的时间，Clear CT 是清理上面说的全局 card table 的时间。

![2016-07-01 7 34 34](https://cloud.githubusercontent.com/assets/1115061/16544449/f9a589ca-4138-11e6-9954-68f989881d04.png)

## RSet 的更新

为了为每个 Region 维护 RSet，就一定涉及到 Region 内有引用被更新的时候，去更新这个 Region 对应的 RSet。

RSet 在 Parallel Old 和 CMS GC 中也有使用，他们是通过 write barrier 来在 Region 内引用有更新的时候去对应的维护 RSet 的。

```
object.field = some_other_object
```

在执行例如上面语句的时候去更新 intergenerational reference。

G1 是引入了两个 barrier，一个 pre-write barrier 和一个 post-write barrier。其中 pre-write barrier 会在后面叙述 G1 concurrent marking 的时候描述，这里只叙述 post-write barrier 功能和 G1 如何使用这个 barrier 去更新 RSet 。

post-write barrier 在每次写入一个 reference 的时候被调用，应用内修改 reference 的地方肯定很多，所以这个 barrier 性能非常关键，执行的慢了会影响整个系统的运行。所以 G1 的这个 post-write barrier 只做很少的事情：

1. 判断这次 reference 写入是不是个 cross-region 的写入，reference 是否符合 old-to-old 或 old-to-young 的 RSet 修改条件；
2. 如果是 cross-region 的写入，则说明需要更新 RSet，于是将引用所在 card 和被引用的 Region 等信息存入一个叫做 update log buffer 或者 dirty card queue 的地方
3. 如果 update log buffer 写满了，就再申请一个新的 buffer 继续写，写满的 buffer 会放在全局的 list 中

之后，由 concurrent refinement threads 去消费这个 update log buffer，拿到 buffer 后这个 GC refinement thread 会根据 buffer 内的信息，实际完成 RSet 更新工作，包括将 reference 记录在 RSet 中以及 RSet 粒度升级等工作。 

concurrent refinement threads 是持续运行的，并且会随着 update log buffer 积累的数量而动态调节。有三个配置项 -XX:G1ConcRefinementGreenZone, -XX:G1ConcRefinementYellowZone, -XX:G1ConcRefinementRedZone 去控制在有多少积压的 buffer 时，使用多少 refinement threads。目的就是为了保证 refinement threads 一定要尽可能的跟上 update log buffer 产生的步伐。但是这个 refinement threads 不是无限增加的，有个 -XX:G1ConcRefinementThreads 能控制 refinement 线程数上限。

如果一旦出现 refinement threads 跟不上 update log buffer 产生的速度，update log buffer 开始出现积压，mutator threads 即上面修改 reference 的线程就会协助 refinement 线程执行 RSet 的更新工作。这个 mutator threads 实际就是应用业务线程，当业务线程去参与 RSet 修改时，系统性能一定会受到影响，所以需要尽力去避免这种状况。

个人理解这里 mutator threads 去帮助 refinement 线程更新 RSet，不是说 mutator thread 在修改 reference 的时候直接同步的更新 RSet，而还是采用上面异步的方式，只是每次写入一个 job 到 update log buffer，就从 update log buffer 中消费一个 job，从而保证 RSet 更新顺序。

除了 Mutator Thread 和 Concurrent Refinement Thread 之外，GC 时真正处理清理工作的 Worker Thread 也会参与消费 update log buffer。可以看上面那张 YGC 实际日志的图，有个 Update RS 过程，这个过程就是在消费 GC 时 Concurrent Refinement Thread 没有处理完的 Update log buffer。

![2016-07-03 10 21 08](https://cloud.githubusercontent.com/assets/1115061/16544448/f72dab3c-4138-11e6-9e6d-40e212ccc196.png)

看到下图是一个线程快照，能看到有很多 Concurrent Refinement Thread 处在运行中。

![2016-07-01 10 16 17](https://cloud.githubusercontent.com/assets/1115061/16544440/b7d7aff0-4138-11e6-86ef-a2eb93d1092f.png)

# Old Generation

接下来再看看 Old Generation 相关内容。之前说了，熬过一定次数 YGC 的 live object 会被晋升到 Old Generation，于是 Old Generation 内存占用会越来越大，并且晋升到 Old 之后之前本来 live 的 object 可能随着使用也变成 dead object 了，也需要去 GC。

当 Old Generation 空间占用整个 Heap 比例超过目标值(-XX:InitiatingHeapOccupancyPercent, IHOP)后，开始 OGC 过程。

**注意：**CMS 是 Garbage 占到整个 Old Generation 比例超过某个值后开始 OGC。而这里 G1 是 Old Generation Garbage 占整个 Heap 的比例。

G1 的 OGC 也是分为 marking 和 sweeping 两个过程。marking 阶段找到当前 Old Generation Heap 中所有 live 的 object，sweeping 过程将 live object 拷贝到新的 available region 从而留下 Garbage 在老的 Region ，之后直接清理掉这些老的 Region。live object 拷贝到 available region 时 live object 是紧挨着排列的，所以没有碎片。清理过程自带 compat 效果。

G1 OGC 最大的特色就是不是一口气将整个 O 取全部清理，从而减小 Old Generation 大小对清理 Old Generation 时间的影响。类似 CMS 或 Parallel 因为每次针对 Old Generation 的清理都要一口气将 Old Generation 全部清理干净，于是 Old Generation 越大，清理的时间越长，所以在大堆上容易产生超长 GC。

## G1 OGC Marking

触发 OGC 后的下一次 YGC 时开始 initial-mark，STW 且并发的标记所有 Root Object。跟着 YGC 一起是因为 YGC 本就需要标记一次所有 Root object。也正因为 initial-mark 是在 YGC 中进行的，所以 marking 开始的时候只用标记 Old Generation Region 就行了，Young Generation 的 Eden 都被清理完了，Survivor 是算作 live object 存在的。

initial-mark 结束后开始 concurrent root scanning. 因为 initial-mark 就是一次 YGC。YGC 后 live Object 都放在 Survivor Region 中。这个过程就是标记所有 Survivor 内 Object 引用的对象。这个过程跟它名字指示的一样是并发的，唯一限制是必须在下一次 YGC 之前完成，因为下一次 YGC 就会产生新的 Survivor ，很有可能跟当前 Survivor 完全不同。

之后是 concurrent marking。大部分 mark 工作都在这里完成。后面会详细再说。这个过程是并发的，对业务的影响主要是降低业务的 throughput.

concurrent marking 结束后开始 STW 的 remark. 标记所有因为 concurrent marking 阶段 marking 线程和业务线程并发运行而导致的没有标记到的 live object.

remark 完毕后，开始 clean up. 如果 mark 阶段发现有没有任何 object 存货的 region，该 region 在该阶段直接被放入 available regions.

### G1 Concurrent Marking

#### Marking 算法

G1 的这套 Marking 算法借鉴了 Taiichi Yuasa 的 Snapshot-at-the-beginning (SATB) 算法，并进行了一些改进。Marking 的最基本目标就是在 Heap 耗尽之前，完成对整个 Heap 的 marking 工作，从而能够在 Heap 耗尽前开始清理。

SATB 内部会对 Heap 维护一个 Snapshot，标记工作也是在这个 Snapshot 上进行。SATB 保证：

1. 所有在 concurrent marking 阶段开始时 live 的 object ，一定会被 marked and traced；
2. 所有在 concurrent marking 过程中产生**或死掉**的 object 都一定被标记为 live 并且不被 traced ;

SATB 会维护两个 bitmap，preivous 和 next。previous bitmap 存的是上一次完成的 marking 信息，当前 marking 阶段会创建并更新 next bitmap。随着 marking 阶段的进行，next bitmap 会逐渐被完善，当 next bitmap 拥有整个 Heap 的 marking 信息后，next bitmap 会替代 previous bitmap。

在 G1 Region 上，有两个 top-at-mark-start (TAMS) 标记位，一个是标记上一次 marking 阶段使用的 TAMS，也称为 PrevTAMS；另一个用来标记本次 marking 阶段，也称为 NextTAMS。

Marking 过程如下图：
![2016-07-03 12 23 19](https://cloud.githubusercontent.com/assets/1115061/16544442/d1748c76-4138-11e6-902d-7c37b6bb0843.png)
(该图片引自参考文献 [2])

上面图是连续两次 mark 的过程。下面对每一步进行解释：
A: PrevBitmap 和 NextBitmap 都是空的，说明这是个新 Region，没有经历过 marking 阶段。Bottom 和 Top 之间是 Region 当前已经分配的空间。因为没有经历过 marking，PrevTAMS 指向 Bottom。NextTAMS 不管 Region 之前是否经历过 marking，initial marking 的时候都会指向 Top。

B: 在 Remark 阶段结束之后，来到了图中 B 指示的阶段。看到 NextBitmap 上已经被标记了哪些是 live object，没被标记的就是 dead object。NextTAMS 到 Top 之间的 object 是 concurrent marking 阶段，因为业务线程跟 marking 线程并发运行而新产生的 object。按照 SATB 之前说的，这部分 object 全部认为是 live 的。也正是因为这个原因，本次 marking 只针对 PrevTAMS 到 NextTAMS 之间的区域进行标记。

C: Cleanup 阶段，NextBitmap 替换 PrevBitmap，因为 marking 工作已经完成，NextBitmap 已经有了整个 Heap 的信息。**注意：**NextBitmap 和 PrevBitmap 实际都是全局的一个 Bitmap，是标识整个 Heap 的。上图中 Bitmap 看上去跟 Region 绑定只是为了方便看，便于理解。

D: 能到 D 这个阶段，这个 Region 经历两次 initial marking，说明在上一次 marking 后，这个 Region 并没有被收集。后面会说，G1 的 Mixed GC 不是一定要在整个 Heap 上所有 dead object 都被收集干净了才停止，而是只要根据 marking 提供的 dead object 占用的空间在整个 Heap 中占比小于一定值后，就停止收集。所以完全是有可能存在能经历两次甚至更多次 marking 的 Region。从 D 能看到 Top 相对于 C 增长了一些，说明上次 marking 结束后，这个 Region 上又有新晋升上来的 object。跟 A 一样，PrevBitmap、PrevTAMS 保持不变，创建 NextBitmap，NextTAMS 指向 Top。并且看到 PrevBitmap 没有变化，因为不经历 makring 这个 bitmap 是不可能变化的。

E: 跟 B 一样，Remark 结束，Bottom 到 NextTAMS 之间所有 live object 都被标识出来，NextTAMS 到 Top 之间是本轮 concurrent marking 阶段新晋升的 object，直接被标记为 live。**注意：**看到第二次 marking 的时候 mark 的还是 Bottom 到 NextTAMS，上一轮已经被 mark 过的 Bottom 到 PrevTAMS 的还是会参与 marking。

F: 跟 C 一样，NextBitmap 替换 PrevBitmap。NextBitmap 被清理。

从上面看到 Marking 阶段实际就是为了维护 PrevBitmap，有了这个 Bitmap，就能知道一个 Region 上有多少 live object，从而能够根据 dead object 空间占比来排序，找出 GC 效率最高的 Region 来 GC。

#### marking 过程

在了解了 Marking 算法过程之后，再回过头来看一遍并发标记的所有阶段。

##### Initial Marking 

之前说过了，就是 STW 的标记所有 roots 直接指向 object 。

root object 指的就是能被 heap 之外引用的对象，比如 native stack objects，JNI local 或 global object 等。

因为 YGC 的时候也是要 STW 的扫描 roots ，所以 initial-mark 都是 piggybacking 到一个 YGC 上进行的，并且会将每个 Region 的 NextTAMS 设置为各自 Region 的 Top 指向的值。

##### Root Region Scanning 

设置完每个 Region 的 NextTAMS 之后，STW 阶段结束，业务线程被重启运行。initial-marking 阶段拷贝到 Survivor Region 的 object 都被认为是 marking roots，需要在本阶段被 scan 。所有被 Survivor Region 内 Object 引用的 Object 都需要被 mark，认为是 live 的。

Root Region Scanning 必须在下一次 YGC 之前完成，不然 Survivor 又被更新了，本阶段就算是失败了。

##### Concurrent Marking

这个阶段就是并发的标记所有 live object，参与 concurrent marking 的线程数由 -XX:ConcGCThreads 规定，如果没有设置默认就是 -XX:ParallelGCThreads 的四分之一。

之前在描述 RSet 功能的时候说过，G1 引入了两个 write barrier，一个 post-barrier 在 RSet 那里用来每次修改引用的时候维护 RSet，还有一个 pre-write barrier 是在 concurrent marking 这个阶段使用的。

之前说过，SATB 的保证是，在 marking 阶段开始时所有的 live object 都会在 marking 结束的时候被标记出来，所有 marking 过程中新生或死亡的 object 都被认为是 live object。新生的对象因为都会在 NextTAMS 到 Top 之间，所以没有什么需要特殊处理的。但 marking 过程中 dead 的 object 需要特殊处理，这里 pre-write 就是干这个特殊处理的。

比如在 concurrent marking 过程中，业务线程执行如下语句：

```
x.f = y
```

也就是说修改了 x 这个 object 中 f 这个引用，另其指向了 y 。那么 x.f 原本指向的 object 可能死亡了也可能还活着，根据 SATB 的要求，需要将其标记为 live。pre-write 的代码逻辑类似：

```
if (is-marking-active) {
  prev = x.f;
  if (prev != Null) {
    satb_enqueue(prev);
  }
}
```

也就是说如果在 marking 过程中，x.f 的引用发生改变，需要将 x.f 原本指向的 object 放入 satb_enqueuey 以异步的方式将 x.f 原本指向的 Object 标记为 live。

satb_enqueue() 是将 prev 放入一个 thread local buffer，也称为 SATB buffer。SATB buffer 有个初始大小，每个业务线程都会有这么个 buffer。当业务线程的 SATB buffer 被占满后，JVM 会再分配一个新的空 SATB buffer 给这个线程使用，写满的那个 SATB buffer 就放在一个全局的 list 中。

执行 concurrent marking 的线程在 scan object 和 mark live object 过程中，会定时的过来查看这个 global list，从中读出 SATB buffer 然后将对应的 object 标记为 live ，被这个 object 指向的所有 object 最终也会被标记为 live。

concurrent marking thread 在 marking 过程中还会计算 live object 数量。从而为之后的清理过程提供参考数据。

问题：
比如一开始有 x.f = z，之后执行了 x.f = y。为何必须标记 z 为 live，不这么标记可以吗？因为如果 x.f = y 执行后，z 对象确实已经 dead 了，不标记 z 应该有助于提升 GC 效率，因为 z 能在本轮 GC 时直接被 GC 掉，而不是像 SATB 这样留到下一轮 GC。如果执行 x.f = y 后，z 没有 dead，即除了 x 之外，z 还被别的 live 对象引用，那么 x.f = y 执行后也不需要通过 SATB queue 的方式将 z 补充标记为 live。所以，为什么 SATB 非得有这么个保证？

回答：我的理解是，x.f = y 执行后无法确定 z 到底是 live 还是 dead。整个 marking 过程都是并发执行的，即使 x.f = y 执行完的那个瞬间 z 有可能再没别的引用指向了，但也不能说 z 就是 dead，因为下一个时刻可能有另一个语句 x1.f = z 执行，而 x1 这个对象是 live 的且已经被标记过。这种情况下 z 就不是 dead ，且不补充标记 z ，z 被 GC 后就导致 x1 这个对象是不可用的。

那这里接着上面的会带，又有个问题是，为何不能在 x1.f = z 真的执行的时候将 x1 标记为 dirty 重新 trace and marking 一下？或者说，为什么不能再 x1.f = z 这种语句真的执行的时候，使用 prev-write barrier 将 z 这个 Object 放入 satb enqueue 补充标记，而非得在 x.f = y 执行的时候就将 z 放入 satb enqueue 去做补充标记呢？

回答：主要原因还是 marking 过程是并发的，业务代码对 GC 来说是不可预测的。拿上面例子来说，x1.f = z 这个语句可能在本轮 GC 的标记过程中执行，也可能是本轮 GC 结束了执行。假若是通过 pre-write barrier 去标记 z，因为 x1.f = z 语句可能在本轮 GC 结束后才执行，本轮 GC 结束后如果 x.f = y 语句执行时，没有 post-write barrier 将 z 补充标记为 live，z 这时候就已经被 GC 掉了，再这么执行 x1.f = z 就会出问题。

##### Remark

该过程是个并发的 STW 过程。GC 线程会将 SATB buffer 消费干净，并将 buffer 中指定 object 标记为 mark ，也将被这些 object 指向的 object 标记为 live。这也是为什么 Remark 必须是 STW 的，因为如果业务线程持续运行，GC 线程就不可能将 SATB buffer 消费干净。

##### Cleanup

这个阶段之前说过，也是并发执行的，且不会 STW。在这个阶段 Next marking bitmap 会替代之前的 Previous marking bitmap，将 PrevTAMS 设置为 NextTABS 的值。

这个阶段三个最耗时的操作是：

1. 根据每个 Region 的 garbage 占用情况，和 RSet popular 程度评估每个 Region 的 GC 效率，并根据 GC 效率将 Region 排序
2. 发现没有 live object 的 Region 时直接将其清理
3. 对每个 Region 的 RSet 进行清理，比如发现一个 card 中指向当前 Region 的 Object 都 dead 了，就直接清理这个 RSet 内的记录。因为之后无论是 YGC 还是 Mixed GC 都会扫描这个 RSet，将其清理一下有助于提升之后清理过程中 RSet 扫描效率

## G1 OGC Sweeping

remark 阶段完毕后，G1 就完成了对整个 heap 的标记，能知道整个 heap 中有哪些 object 是 live 的。并根据 Region 中可收集 Garbage 大小排序，选出 GC 效率最高的 Region 组成 CSet。CSet 就是 G1 目标收集的 Region 集合。在接下来的几次 YGC 中，会从 CSet 中依次选出 GC 效率最高的 Region 收集，即 Mixed GC. “GC 效率最高” 一般是有两个指标，一个是 Region 内 live object 多少，live object 占空间最少的 Region，GC 效率越高。即 Garbage 越多的 Region，GC 效率越高。这也是 Garbage First 的由来。另一个是 Region 的 "popular" 程度，越 “popular“ 的 Region 就有越多的 Region 含有引用指向这个 Region，其 RSet 扫描和更新操作耗时也越长。RSet 扫描和更新后续会说。

但也不是只要是有 dead object 就会被放入 CSet，而是有个参数去控制放入 CSet 的 Region 的选择。-XX:G1MixedGCLiveThresholdPercent，默认是 85% 即当一个 Region 内 live object 空间占比小于 85% 时，就会被放入 CSet。

YGC 时 CSet 内全部是 Young Generation Region。OGC 时，CSet 内有一部分 Young Generation Region 也有一部分 O 区 Region。

G1 的收集无论是 Old Generation 还是 Young Generation ，都是 live object 拷贝到一个 available region，拷贝过去后在这个新的 region 上每个 object 都是紧挨着排列的，所以没有 fragment.

需要注意的是，Mixed GC 是 G1 最主要的清理内存的阶段，但 mixed GC 要求 marking 阶段必须结束，从而能知道 heap 中有哪些 live object，才能开始清理。如果 marking 阶段没结束 heap 就满了，G1 会先尝试扩大 heap，如果无法扩大，则使用 fail-safe GC 收集内存。

Sweeping 阶段具体由几轮 Mixed GC 组成，每次 Mixed GC 需要收集多少个 Region，需要由两个参数决定：-XX:G1MixedGCCountTgarget 和 -XX:G1HeapWastePercent。

-XX:G1MixedGCCountTarget 限定 Sweeping 阶段连续的 Mixed GC 最大次数。所有 Mixed GC 阶段待收集的 Region 总数，除以 G1MixedGCCountTarget 就是每轮 Mixed GC 最少需要清理的 Region 数。

这里计算的是单次 Mixed GC 最少需要清理的 Region 数，还有一个 -XX:G1OldCSetRegionThresholdPercent 用于控制单次 Mixed GC 最多能收集多少 old regions。

-XX:G1HeapWastePercent 限定 Mixed GC 在 Garbage 占 Heap 空间的百分之多少的时候停止 Mixed GC。也就是说每次 Mixed GC 结束都会计算当前 dead object 占总 Heap 空间比例，当这个比例小于 G1HeapWastePercent 后，就停止 Mixed GC 即使现在还有 dead object 没有收集完也停止收集了。因为剩下的 Region 内可能剩余的 dead object 比例不是很多，收集起来效率很低。

G1MixedGCCountTarget 决定 Mixed GC 最大次数，G1HeapWastePercent 决定 Mixed GC 实际次数。

### Evacuation Failures And FGC

Evacuation 过程中有三种情况会导致 G1 降级为 FGC 去收集内存：

1. YGC 时，无法找到 available region 去存放 survive 的 object；
2. Mixed GC 时，无法找到 available region 去存放 live object；
3. 无法找到足够大的连续的 Region 去存放 Humongous Object

Humongous Object 后续会说。

FGC 是单线程的，完成 mark 、 sweep、compation 工作。单线程的 FGC 效率最低，但也是最安全的。 

# Humongous Object

正因为无论是 Young Generation 还是 Old Generation，在 GC 的时候都会有 object 拷贝。Young Generation 一方面是将 object 从 Eden 拷贝到 Survivor ，另一方面是拷贝晋升的 object 到 Old 区。这种拷贝过程对特别大的 object 来说就很不经济。

G1 中 Region 大小最小是 1MB，最大是 32MB。具体多大会根据 Heap 大小做设置，它是尽力去保证整个 Heap 被划分为大约 2048 个 Region。比如如果 Heap 有 16G，算下来 16G / 2048 = 8MB 即一个 Region 大概是 8MB。当然 2048 个 Region 也不是绝对的，如果 Heap 特别大或者特别小，Region 总数是可以超过或小于 2048。Region 总数也能通过参数精确设置 -XX:G1HeapRegionSize=n。

回到 Humongous Object，G1 中内存占用超过当前单个 Region 50% 的 Object 就叫 Humongous Object，G1 对他们有单独的处理。

Humongous Object 分配时会根据这个 object 大小，在 available regions 中找足够放下这个 object 的连续的数个 region，专门分配给这个 Humongous object 使用。如果找不到这么个连续的 region，G1 会直接使用 fail-safe 的 FGC 来清理并 compact heap。

理解这里不先进行 YGC 或 OGC 的原因是 YGC 和 OGC 很多过程都是 concurrent 的，这个时候 Humongous Object 无法分配内存，无法让应用线程继续运行，必须执行完全的 STW 收集一次内存才行。
![2016-07-03 12 16 36](https://cloud.githubusercontent.com/assets/1115061/16544443/d4082fe2-4138-11e6-99e0-ea98ae50d94c.png)

更细的看上面存放 Humongous Object 的连续的 Region：
![2016-07-03 10 21 08](https://cloud.githubusercontent.com/assets/1115061/16544448/f72dab3c-4138-11e6-9e6d-40e212ccc196.png)
看到连续的 Region 是由 StartsHumongous 和 ContinuesHumongous Region 组成的。

开辟单独的区域存放 Humongous Object 是为了避免 long-live 的大对象在 GC 过程中的拷贝。开辟连续的 Region 只存放一个 Humongous Object 是为了让 G1 对 Humongous Object 更激进的进行收集，只要发现这个 object dead，就能将其所占用的 Regions 全部收集，不用去判断 Region 还有没有别的 object 使用，别的 object 是否还 live。比如在 marking 的 clean up 阶段、 YGC 和 FGC 时，发现 humongous object 没有任何引用，就会立即被收集。

但正因为 Humongous Object 这种独特的分配机制，使得其无法享受到 TLAB 和 PLAB 带来的便利。

# Heap Size 调节

G1 heap 大小可以在 -Xms -Xmx 之间变化。

G1 增大 heap 的时机有：

1. FGC 时会根据应用行为计算预期 heap size，增大 heap；
2. YGC 或 OGC 触发时，G1 计算 GC 花费的时间相对应用运行时间的比例，如果 GC 耗费时间比例过大(可通过 -XX:GCTimeRatio 调节，G1 默认是 9，其它 GC 是 99 也就是说其它 GC会更激进的扩展 heap size)，Heap size 会增加从而减少 GC 发生次数，增大单次 GC 所能收集的内存比例. 
3. 分配内存失败时，在进行 fail-safe GC 之前，会先尝试扩展一下内存;
4. 无法为 Humongous object 找到足够大的连续 region 时，先尝试扩展内存，再 fail-safe;
5. G1 GC 时将 live object 拷贝到一个 available region，如果找不到这么个 available region，会先尝试扩展内存，无法扩展则执行 fail-safe





参考文献：

[1] Charlie Hunt,Monica Beckwith,Poonam Parhar,Bengt Rutisson. Java Performance Companion. Addison-Wesley. ISBN-13: 978-0-13-379682-7

[2] David Detlefs, Christine Flood, Steve Heller, Tony Printezis. Garbage-First Garbage Collection.ISMM’04, October 24–25, 2004, Vancouver, British Columbia, Canada. ACM 1-58113-945-4/04/0010.

[3] Darko Stefanovic,Matthew Hertz, Stephen M. Blackburn,Kathryn S. McKinley,J. Eliot B. Moss†. Older-first Garbage Collection in Practice: Evaluation in a Java Virtual Machine.

[4] Taiichi Yuasa. Real-Time Garbage Collection on General Purpose Machines. Journal of Systems and Software, Volume 11, Issue 3, March 1990, pp. 181-98. Elsevier Science, Inc., New York.

[5] Tony Printezis and David Detlefs. A Generational Mostyly-Concurrent Garbage Collector. Proceedings of the 2nd Internaltional Symposium on Memory Management. ACM, New York, 2000, pp. 143-54. ISBN 1-58113-263-8
