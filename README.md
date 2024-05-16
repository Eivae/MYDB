# MYDB

MYDB 是一个 Java 实现的简单的数据库，部分原理参照自 MySQL、PostgreSQL 和 SQLite。实现了以下功能：

- 数据的可靠性和数据恢复
- 两段锁协议（2PL）实现可串行化调度
- MVCC
- 两种事务隔离级别（读提交和可重复读）
- 死锁处理
- 简单的表和字段管理
- 简陋的 SQL 解析（因为懒得写词法分析和自动机，就弄得比较简陋）
- 基于 socket 的 server 和 client

# 开始运行


在修改了本地路径后，执行仍然失败，原因是-D后面要有空格！！！<br>
<br>

```shell
mvn exec:java -D exec.mainClass="top.guoziyang.mydb.backend.Launcher" -D exec.args="-create D:\mydb"
```

同样，后面的类似命令都要加空格。<br>
<br>



# Transaction Manager(TM)

>> TM 通过维护 XID 文件来维护事务的状态，并提供接口供其他模块来查询某个事务的状态。

这部分的思路：
- 1、先写一个接口 TransactionManager ，在这个接口中定义一些接口，再定义一些方法，如创建TM对象和XID文件的方法
- 2、在 TransactionManagerImpl 中检查XID文件是否合法，并实现接口中各个的抽象方法
- 3、写begin方法，创建一个新事务，在XID文件中修改头部和增加尾部。
- 4、写其他重写的方法
- 5、写测试方法

第一步里的创建TM对象的方法不是在TransactionManagerImpl中调用的，而是在调用这个方法的类中用到的，这个方法是为了创建TM的，即实现类的实例的，如果放在实现类里，则只有创建实例了才能调用了，这就矛盾了，除非在实现类里用static修饰这个方法。但还是放接口里好一些，这样每个实现了这个接口的类都能够创建TM了，而且create方法返回的就是一个TransactionManagerImpl对象，凡在TransactionManagerImpl里就会出现调用类里的方法返回类的对象的情况，有点怪异。）

### XID 文件
下面主要是规则的定义了。<br>
<br>

在 MYDB 中，每一个事务都有一个 XID，这个 ID 唯一标识了这个事务。事务的 XID 从 1 开始标号，并自增，不可重复。并特殊规定 XID 0 是一个超级事务（Super Transaction）。当一些操作想在没有申请事务的情况下进行，那么可以将操作的 XID 设置为 0。XID 为 0 的事务的状态永远是 committed。<br>
<br>

在MySQL中，XID是由InnoDB存储引擎生成和管理的。<br>
<br>

TransactionManager 维护了一个 XID 格式的文件，用来记录各个事务的状态。MYDB 中，每个事务都有下面的三种状态：
- active，正在进行，尚未结束
- committed，已提交
- aborted，已撤销（回滚）

XID 文件给每个事务分配了一个字节的空间，用来保存其状态。同时，在 XID 文件的头部，还保存了一个 8 字节的数字，记录了这个 XID 文件管理的事务的个数。于是，事务 xid 在文件中的状态就存储在 (xid-1)+8 字节处，xid-1 是因为 xid 0（Super XID） 的状态不需要记录。<br>
<br>

上面那段话我是这样理解的，XID文件只有一个，然后这个文件由两大部分组成，一个部分是文件的头部，占8字节，保存了XID文件中事务的个数；头部后面的部分就是事务xid的状态，每个状态占1字节，依次往后排。xid-1后，xid=1的事务在8字节处（类似数组，从0开始计数？也就是第9个字节？）<br>
<br>



# Data Manager(DM)

Data Manager 是MYDB中最底层的模块<br>
<br>

>> DM 直接管理数据库 DB 文件和日志文件。DM 的主要职责有：1) 分页管理 DB 文件，并进行缓存；2) 管理日志文件，保证在发生错误时可以根据日志进行恢复；3) 抽象 DB 文件为 DataItem 供上层模块使用，并提供缓存。

归纳起来，DM功能有两点：
- 上层模块和文件系统之间的一个抽象层，向下直接读写文件，向上提供数据的包装
- 另外就是日志功能

可以注意到，无论是向上还是向下，DM 都提供了一个缓存的功能，用内存操作来保证效率。<br>
<br>


思路：
- 1、先实现一个引用计数策略的缓存（因为无论向上还是向下，都需要缓存。将这个缓存类写在common包里）


## 引用计数缓存框架
为什么用引用计数缓存而不用LRU？<br>
<br>

回源：指将缓存中的数据刷会数据源（磁盘）？<br>
<br>


## 数据页的缓存与管理

数据库中的InnoDB存储引擎中，逻辑储存结构为：
- 表空间（tablespace）
- 段（segment）
- 区（extent）
- 页（page）
- 行（row）

页是mysql中磁盘和内存交换的基本单位，也是mysql管理存储空间的基本单位。

本节主要内容就是 DM 模块向下对文件系统的抽象部分。DM 将文件系统抽象成页面，每次对文件系统的读写都是以页面为单位的。同样，从文件系统读进来的数据也是**以页面为单位进行缓存的**。<br>
<br>


### 页面缓存
上一节我们已经实现了一个通用的缓存框架，那么这一节我们需要缓存页面，就可以直接借用那个缓存的框架了。但是首先，需要定义出页面的结构。**注意这个页面是存储在内存中的**，与已经持久化到磁盘的抽象页面有区别。<br>
<br>

页面缓存首先是缓存，只不过缓存的数据是页面，因此，在实现页面缓存之前，我们需要先实现页面（这个页面是储存在内存中的，因为缓存是在内存中，所以缓存中的页面也应该是在内存中）。页面结构在软件包Page中<br>
<br>

**脏页**：缓存中和磁盘数据不一致的页称为脏页。<br>
<br>

脏页会在合适的时候写回磁盘。<br>
<br>

**注意：如果一个类继承了父类，那么当这个类进行实例化的时候也会对父类进行实例化，使用的方法默认为父类的默认构造方法，如果父类中写了有参构造方法而没写无参构造方法，那么无参构造方法就被覆盖了，也就是说编译器不会再自动添加无参构造方法了，所以在子类的构造方法中必须显示地写出父类的构造方法 `super(args)`。** <br>
<br>


理清这一小节的两个包page和pageCache中的类的关系：
- 1、page 接口中的Page定义了页面结构，里面包含了一些抽象方法，定义了页面结构具有的一些功能，比如锁、是否为脏页、页号、页中的数据等等

- 2、pageImpl 实现了page接口，除了Page中的一些内容外，还包含了一个 PageCache 的字段，它用于在拿到 Page 的引用时可以快速对这个页面的缓存进行释放操作（也就是pageIml里的release方法要用到）。

- 3、PageCache 接口定义了页面缓存应该具有的一些功能（同第1点类似），比如创建新的页、得到页、释放页等，另外，还定义了 create 和 open 两个方法，用于外部创建或打开一个数据库文件对应页面缓存？

- 4、pageCacheImpl 实现了 PageCache接口，并继承了引用计数缓存框架 AbstractCache。如何把数据源中的数据放入缓存？利用 AbstractCache 中的 getForCache() 方法，释放则用 releaseForCache 方法，因此在 pageCacheImpl 中还要实现这两个抽象方法。


**问题：上层通过传下一个long型整数来表示需要调用的页，这个整数到底是个啥？** <br>
<br>
应该就是页号，在innodb存储引擎中，一个区有1M，一页有16K，因此有64页，页号用int来表示应该就足够了，是0-63吗？每个区的页号都是单独的？<br>
<br>

AtomicInteger pageNumbers可以转换为int：`pageNumbers.intValue()`。如果原来的值超过了int的范围，就会溢出。<br>
<br>


#### 接口和实现类的一些问题
>> ![Java 中到底是应该用接口类型 还是实现类的类类型去引用对象？](https://blog.csdn.net/summerxiachen/article/details/79733800)

结论：**应该优先使用接口而不是类来引用对象，但只有存在适当的接口类型时，即实现类中全是接口类的方法的实现，没有自己单独的方法。当实现类存在自己的方法时，使用实现类来声明变量。**

如果用的是接口类型来引用对象，而实现类中有自己单独的方法，那么可以通过向下转型来使用实现类的方法。<br>
<br>

一般用父类之类的来做引用，可以使得它的包容性更强？比如要返回 List<List<Integer>> ，那么 List<Integer> 既可以是 List<Integer> tmp = new ArrayList<>() ，又可以是 LinkedList<Integer> tmp = new LinkedList<>(); 因为他们都实现了List接口？（这部分写在LeetCode笔记里）<br>
<br>




## 日志文件与恢复策略

### 日志文件读写
MYDB 提供了崩溃后的数据恢复功能。DM 层在每次对底层数据操作时，都会记录一条日志到磁盘上。在数据库奔溃之后，再次启动时，可以根据日志的内容，恢复数据文件，保证其一致性。<br>
<br>

日志的二进制文件，按照如下的格式进行排布：

```
[XChecksum][Log1][Log2][Log3]...[LogN][BadTail]
```

其中 XChecksum 是一个四字节的整数，是对后续所有日志计算的校验和。Log1 ~ LogN 是常规的日志数据，BadTail 是在数据库崩溃时，没有来得及写完的日志数据，这个 BadTail 不一定存在。

每条日志的格式如下：

```
[Size][Checksum][Data]
```

其中，Size 是一个四字节整数，标识了 Data 段的字节数。Checksum 则是该条日志的校验和。<br>
<br>


### 恢复策略

DM 为上层模块，提供了两种操作，分别是插入新数据（I）和更新现有数据（U）。至于为啥没有删除数据，这个会在 VM 一节叙述。<br>
<br>

DM 的日志策略很简单，一句话就是：
>> 在进行 I 和 U 操作之前，必须先进行对应的日志操作，在保证日志写入磁盘后，才进行数据操作。


一些规定：
>> 规定1：正在进行的事务，不会读取其他任何未提交的事务产生的数据。（防止级联回滚与 commit 语义冲突）
>> 规定2：正在进行的事务，不会修改其他任何未提交的事务修改或产生的数据。

有了这两条规定，并发情况下日志的恢复也就很简单了：
- 重做所有崩溃时已完成（committed 或 aborted）的事务
- 撤销所有崩溃时未完成（active）的事务






## 页面索引与 DM 的实现

### 页面索引
页面索引缓存了每一页的空闲空间，用于在上层模块进行插入操作时，能够快速找到一个**合适空间**的页面，而无需从磁盘或者缓存中检查每一个页面的信息。<br>
<br>

在 select 方法中，注意到，被选择的页，会直接从 PageIndex 中移除，这意味着，同一个页面是不允许并发写的。<br>
<br>


### DataItem
DataItem 是 DM 层向上层提供的数据抽象。上层模块通过地址，向 DM 请求到对应的 DataItem，再获取到其中的数据。<br>
<br>





总结来看，整个DM中有几种缓存？一个是PageCache，也是用在上层模块与数据库文件之间，但这个DataItem又是什么？和PageCache有什么关系？DataItem只是一种对从db文件读取到的数据的抽象，面向上层模块，它并不是一种缓存，应该说，缓存中存储的就是DataItem类的数据？<br>
<br>

另外，页面索引是一种缓存吗？它和普通的PageCache又是什么关系？页面索引应该不是一种缓存，它并没有继承 AbstractCache 类，它只是把所有页面信息（有多少页？）存储在一个列表数组中，程序运行时这个数组放在内存中，因此速度比较快（好像跟缓存也挺像？）<br>
<br>

PageCache有多大？在create时就已经指定了。创建（create）的时候会指定缓存内存大小，打开（open）的时候同样也要指定内存大小，这样，缓存的最大资源数就确定了。在create或open时指定的db文件并不是缓存，get时如果缓存中没有就会从这个db文件中去取。应该说，缓存就是上层模块与磁盘的db文件之间的桥梁，要在db文件中新建一个页，也是通过调用缓存的 newPage() 方法才能创建，在这个方法中，会将新页写入磁盘（db文件）中去。同样，release() 方法也会讲缓存中的数据写入磁盘。在MYDB中，好像是一个db文件就会有一个页面缓存，因为在create和open时会指定路径，从而只关联到一个文件。实际的数据库中，缓存应该是对所有文件都是用的？<br>
<br>


long uid 是由两部分构成，高位4字节保存页号pgno，低位四字节（其实是最低位两字节）保存偏移量offset。创建uid可能就是为了return的时候能够一次性返回两个量，比如DataManagerImpl中的insert方法返回的就是uid，要不然要返回两个量不方便。<br>
<br>

**疑问：为什么parseDataItem()方法会从raw中读取size()？是否错误？**

如果一个类中的字段为private，那么该类的对象不能调用？比如 dm.pageOne。







# Version Manager(VM)

## 记录的版本与事务隔离
>> VM 基于两段锁协议实现了调度序列的可串行化，并实现了 MVCC 以消除读写阻塞。同时实现了两种隔离级别。

Version Manager 是 MYDB 的事务和数据版本的管理核心。<br>
<br>

2PL：两段锁协议<br>
<br>

2PL 确实保证了调度序列的可串行化，但是不可避免地导致了事务间的相互阻塞，甚至可能导致死锁。MYDB 为了提高事务处理的效率，降低阻塞概率，实现了 MVCC。<br>
<br>

首先明确记录和版本的概念。<br>
<br>

DM 层向上层提供了数据项（Data Item）的概念，VM 通过管理所有的数据项，向上层提供了**记录（Entry）** 的概念。上层模块通过 VM 操作数据的最小单位，就是记录。VM 则在其内部，为每个记录，维护了多个**版本（Version）**。每当上层模块对某个记录进行修改时，VM 就会为这个记录创建一个新的版本。<br>
<br>

虽然理论上，MVCC 实现了多版本，但是在实现中，VM 并没有提供 Update 操作，对于字段的更新操作由后面的表和字段管理（TBM）实现。所以在 VM 的实现中，一条记录只有一个版本。<br>
<br>

一条记录存储在一条 Data Item 中，但由于我们需要能对记录做一些操作（即记录的一些功能，或方法），我们需要用一个类（Entry 类）来维护其结构，就像页面缓存不只是将页面与页号放在一个HashMap中，而是创建了一个类，这个类中真正存放数据的就是这个HashMap，但这个类还提供了对这个HashMap进行操作的功能。<br>
<br>


我们规定，一条 Entry 中存储的数据格式如下：
>> [XMIN] [XMAX] [DATA] <br>
<br>
>> XMIN 应当在版本创建时填写，而 XMAX 则在版本被删除，或者有新版本出现时填写。<br>
<br>
>> XMAX 这个变量，解释了为什么 DM 层不提供删除操作，当想删除一个版本时，只需要设置其 XMAX，这样，这个版本对每一个 XMAX 之后的事务都是不可见的，也就等价于删除了。
<br>

在一个Entry类中，字段有三个：
- 数据项dataItem：用于存放上面那个格式的数据
- VM：vm中有dm？
- uid：dm中由页号和offset组成的uid，用于从页中读取数据


VM需要TM和DM。


### 事务的隔离级别
- 读提交：最低的事务隔离程度，只能读取已经提交事务产生的数据。**能防止级联回滚与 commit 语义冲突，但会导致不可重复读和幻读**。
- 可重复读：无法读取
  - 在本事务后开始的事务的数据
  - 本事务开始时还是 active 状态的事务的数据



## 死锁检测与VM的实现（这部分不太懂）

MVCC 的实现，使得 MYDB 在撤销或是回滚事务很简单：只需要将这个事务标记为 aborted 即可。根据前一章提到的可见性，每个事务都只能看到其他 committed 的事务所产生的数据，一个 aborted 事务产生的数据，就不会对其他事务产生任何影响了，也就相当于，这个事务不曾存在过。<br>
<br>

读提交是允许版本跳跃的，而可重复读则是不允许版本跳跃的。<br>
<br>

解决版本跳跃的思路也很简单：如果 Ti 需要修改 X，而 X 已经被 Ti 不可见的事务 Tj 修改了，那么要求 Ti 回滚。












