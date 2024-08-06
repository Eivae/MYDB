# 到底怎么实现的MVCC的多版本数据？在B+树插入时如果数据已经存在（也就是更新）时会将原来的数据覆盖？



# 面试整理
## TM
>> 通过 TransactionManager（TM）模块对数据库事务状态进行管理，包括进行，提交和回滚；事务完全遵循ACID原则，保证了事务的原子性。

TM定义了一些事务的基本功能，如事务的开启、提交、回滚、事务状态查询（根据事务xid查询）。还提供了创建与打开一个事务管理器的方法：create()方法和open()方法。事务管理器就是一个实现了TM接口的类的实例。<br><br>


TransactionManager 维护了一个 XID 格式的文件，用来记录各个事务的状态。MYDB 中，每个事务都有下面的三种状态（分别用0、1、2表示）：
- active，正在进行，尚未结束
- committed，已提交
- aborted，已撤销（回滚）

MYDB中一个数据库只有一个XID文件，这个文件由两部分组成，一个部分是文件的头部，占8字节，保存了XID文件中事务的个数；头部后面的部分就是事务xid的状态，每个状态占1字节，依次往后排。xid从1开始增加，不同事务的xid不重复。并特殊规定 XID 0 是一个超级事务（Super Transaction）。当一些操作想在没有申请事务的情况下进行，那么可以将操作的 XID 设置为 0（比如要新建一个根节点？）。XID 为 0 的事务的状态永远是 committed。<br><br>


每开启一个新事务，就需要在xid文件最后写入该事务的xid值，其实就是原来xid的个数加1。<br><br>


**事务的ACID特性怎么保证的？** <br><br>


原子性通过加锁来保证？还是用回滚日志来保证？隔离性是通过MVCC，用到了锁来保证读吗？<br><br>


## 引用计数缓存框架
>> 采用 **引用计数缓存框架** 对整个过程的缓存进行管理，在回收时进行刷盘操作，保证了系统的稳定性和可靠性。

为什么要设计一个缓存框架？因为分页管理和数据项（DataItem）管理都用到了缓存，因此设计一个更为通用的框架，所有缓存都用这个框架，方便管理。<br><br>

为什么不用LRU？或者说为什么要选这种实现方式？LRU策略中，资源驱逐不可控，上层模块无法感知。而引用计数策略正好解决了这个问题，只有上层模块主动释放引用（**什么时候才释放？**），缓存在确保没有模块在使用这个资源了，才会去驱逐资源。这样就能防止正在被引用的资源不知不觉被驱逐？但是，缓存满了无法自动释放内存，会直接报错。<br><br>

LRU中，只需要有一个get方法就可以了。每次get时，都先去缓存中查一下有没有，如果有，就直接返回；如果没有，需要回源。但是，当get完了之后，缓存满了，此时缓存将驱逐最近没有被使用的那个资源，但如果<br><br>

LRU的缺陷：
- 缓存污染：当我们批量读取数据时，由于这些数据被访问了一次，这些大量数据会被放入到缓存中的链表的头部，从而把之前链表里的活跃数据给挤掉了，如果这些大量的数据只是访问了一次，之后很长时间都不会访问的话，就导致整个链表被污染了。


每个资源都有一个long型的id，缓存（一个hashmap）中，key为id，value为实际的缓存数据。还有一个hashmap references，它是用来保存每个资源的引用次数的，key为id，value为该资源的引用次数。实际上还有一个hashmap getting，用于保存缓存中某个资源是否正在被获取（？），key为资源id，value为boolean。<br><br>

为什么要用引用计数法？可能就是因为实现起来简单。<br><br>

## DM
>> 在 DataManager（DM）中，分页管理存储数据的 DB 文件和日志文件，同时抽象 DB 文件为 DataItem 对象供上层使用，提供 read 和 insert 方法。

数据库中的InnoDB存储引擎中，逻辑储存结构为：
- 表空间（tablespace）
- 段（segment）
- 区（extent）
- 页（page）
- 行（row）

页是mysql中磁盘和内存交换的基本单位，也是mysql管理存储空间的基本单位。

- 首先，写一个Page接口，这个接口定义了对页面的操作，比如对页面加锁与释放锁、释放页面（即引用计数减1）、标记为脏页、获取页的数据等等。

- 然后，写一个类实现Page接口，即PageImpl类，这个类里面有一个字节数组类型的字段data用于保存该页的实际数据。

- 创建一个PageX类，表示普通页，对它的管理很简单，就是通过一个FSO的变量来完成，FSO就是 Free Space Offset，表示页中自由空间起始位置的偏移量。我们向页中插入数据时就在这个位置插入，然后再更新一下FSO就行了。它里面还有数据库崩溃后重新恢复普通页数据的方法。**另外，这样的页怎么更新中间的某个数据？如果更新前后数据不一样不是更麻烦？**

- 创建一个PageOne类，表示一个特殊页，这个页的作用是用于判断上一次数据库是否正常关闭。数据库启动时会往这个页100\~107字节处填入一个随机字节，数据库正常关闭时将其拷贝到108\~115字节。数据库每次启动时都会检查一下这前后两个位置的数据是否是一致的，如果是一致的，说明数据库时正常关闭的；如果不一致，说明数据库不是正常关闭的，可能中途发生了崩溃，导致没有重写第二个位置的数据，使得下次启动时不一致，从而进行数据恢复的流程。数据库文件的第一页，通常用作一些特殊用途，比如存储一些元数据，用来启动检查什么的。MYDB 的第一页，只是用来做启动检查。

- 往其中填入的内容是一个随机字符串，是用 `SecureRandom` 随机生成的（见下面）

```java
//利用SecureRandom随机生成一个指定长度的字符串
public static byte[] randomBytes(int length) {
    Random r = new SecureRandom();
    byte[] buf = new byte[length];
    r.nextBytes(buf);
    return buf;
}
```


- 在MYDB中，一个页被设为8KB的大小。

- 关于页面缓存，先创建了一个页面缓存的接口，定义了对该缓存的一些操作，比如新创建一个页（也就是缓存中的一个资源）、释放一个资源、根据页号pgno从缓存中得到一个资源等等。在这个接口中，还有创建和打开缓存文件的create()和open()方法，但是，为什么缓存文件为.db文件？这不是数据源吗？其实，创建的.db文件就是数据文件，虽然它是在PageCache接口中创建并打开的，但并不是说这个文件只是内存中的缓存的持久化文件，之所以这样，是因为缓存本身就是数据库与上层的桥梁，上面创建了一个页后也是先写入缓存，再写入磁盘，而不是直接写入磁盘。同样，上层对某个页的修改也是先修改缓存中的页，再在之后由缓存将这个脏页刷入磁盘，而不是直接去操作磁盘。每次创建一个页面缓存都会建立一个该缓存与db文件的文件通道FileChannel fc。

首先，最开始什么都没有。然后，我们要去创建一个数据管理器DM，创建的时候先回创建一个页面缓存，在创建页面缓存的过程中，如果db文件不存在，则需要先创建一个db文件。创建完缓存之后，再创建一个日志。利用页面缓存、日志和TM创建一个DM，再调用dm中的方法创建一个特殊页并刷入db文件中，因此，pageOne是db中第0个（或第1个）页面。这样，一个dm就创建好了。之后就可以用dm中的insert()方法插入数据，要插入的是一个字节数组，显然不能直接放进去，因为db文件中的数据是以页面格式保存的，所以需要先创建一个新页或从已经存在的页中找出空闲空间足够大的页，来保存我们要插入的数据。从db中取出数据时也是直接取出一整页的数据，然后放入缓存中。这样就实现了对db文件的分页管理。<br><br>


PageImpl和PageX的关系是什么？整个系统中的页自始至终都只有Page（或PageImpl）的格式，PageX、PageOne这两个类中并没有存储数据的字节数组，这两个类中的方法都是静态的，它们只是用来帮我们操纵页面数据的。<br><br>





### DM是如何分页管理db文件的？

DM 将文件系统抽象成页面，每次对文件系统的读写都是以页面为单位的。同样，从文件系统读进来的数据也是以页面为单位进行缓存的。要实现这一点，我们先定义了一个页面接口Page，这个接口中有一些对页的操作方法对页面加锁、释放锁、释放页面（即引用计数减1）、标记为脏页等等，然后定义一个Page接口的实现类PageImpl，这个类中有一个字节数组类型的字段，用于存储来自磁盘的真正数据。我们每次从磁盘中读取到的字节数组都会被包装成一个页面的形式，放入缓存的也是这种格式。<br><br>

从形式上来看，db中的页都是一样的，但是从功能上，db中的页又分为普通页和特殊页两种。普通页的格式是这样的，它的起始两个字节保存的是一个无符号的短整型数字，这个数字表示这个页的空闲位置的偏移，它用一个叫做FSO字段来表示，FSO就是Free Space Offset的缩写。除了FSO外，剩下的内容就是实际存储的数据。当我们向页中插入数据时就在FSO的位置插入，然后再更新一下FSO就行了。**另外，这样的页怎么更新中间的某个数据？如果更新前后数据不一样不是更麻烦？**。<br><br>

特殊页是db文件中的第一页，这个页的作用是用于判断上一次数据库是否正常关闭。数据库启动时会往这个页100\~107字节处填入一个随机字节，数据库正常关闭时将其拷贝到108\~115字节。数据库每次启动时都会检查一下这前后两个位置的数据是否是一致的，如果是一致的，说明数据库时正常关闭的；如果不一致，说明数据库不是正常关闭的，可能中途发生了崩溃，导致没有重写第二个位置的数据，使得下次启动时不一致，从而进行数据恢复的流程。（数据库文件的第一页，通常用作一些特殊用途，比如存储一些元数据，用来启动检查什么的。MYDB 的第一页，只是用来做启动检查。）<br><br>

索引是否页算分页管理的内容？<br><br>

在 MySQL 中，页的底层实现方式与存储引擎有关。不同的存储引擎可能会使用不同的数据结构来组织和管理页。例如，InnoDB 存储引擎使用了 B+ 树数据结构来组织和管理索引页和数据页，以提高访问效率。而 MyISAM 存储引擎则使用了堆数据结构来管理数据页，以提高插入和删除数据的性能。<br><br>

Mysql中的页中包含了哪些内容？比如行数据、索引数据以及其他管理信息。




### 日志文件
日志文件标准格式如下，XChecksum 为后续所有日志计算的Checksum，int类型。
```
[XChecksum] [Log1] [Log2] ... [LogN] [BadTail]
```


每条正确日志的格式如下，Size 4字节int，用于标识Data长度，Checksum 4字节int类型。
```
[Size] [Checksum] [Data]
```

其中，Data部分是每条日志真正存储的东西，表示每条操作的具体内容，它的格式如下：
```
[LogType] [XID] [Pgno] [Offset] [Raw]
```

DM 层在每次对底层数据操作（**注意，读取操作并不会落日志，只有插入或修改才会**）时，都会记录一条日志到磁盘上。在数据库崩溃之后，再次启动时，可以根据日志的内容，恢复数据文件，保证其一致性。<br><br>

dm执行insert()方法时，会生成一条插入日志，就是一个字节数组，类型为 `[LogType] [XID] [Pgno] [Offset] [Raw]`，也就是一条插入日志包含了操作类型、执行操作的事务、操作的页和偏移量、新插入的数据，它是一种redo日志。<br><br>

更新操作落日志并不是在DataManagerImpl中，而是在DataItem中，每次更新都会加写锁并释放写锁，在释放写锁前（其实就是在after()方法中），都会写入一条更新日志。<br><br>

当dm执行更新方法时呢？dm好像无法更新数据？还是说更新数据就是把一个数据删了（也不是真删），然后再插入新数据？一个表中的数据不一定就是紧挨着的？按后面索引的内容，好像只要提供一个uid用来找就行了，不用非得在一个页里？如果确实是这样的话，那么更新操作确实可以先删后插入。<br><br>

dm的删除操作就是把valid位置为0.<br><br>


### 恢复策略
写了日志之后在发生数据库崩溃时怎么回复？<br><br>

DM 的日志策略很简单，一句话就是：**在进行插入和更新操作之前，必须先进行对应的日志操作，在保证日志写入磁盘后，才进行数据操作**。MySQL中的重做日志也是先写日志再写磁盘。这样做的好处是即使数据库崩溃了，修改操作也不会丢失。<br><br>

回复操作总的只有两种，重做和回滚。重做比较容易实现，因为只要按照日志当中的信息再做一遍就行了，不过它也是要重新遍历一遍完整的日志文件的，所以说只是简单地实现了一下。但是回滚操作要稍微麻烦些，它需要先遍历整个日志文件，将那些活跃状态的事务挑出来，然后将这个事务的操作都放入一个列表当中，然后创建一个哈希表，key为活跃事务的xid，value为操作列表，这样我们就得到了所有活跃事务的所有操作了。然后回滚的时候就倒着遍历列表，将以前做的操作都撤销，如果是插入操作，则删除这个记录；如果是更新操作，就将值改为旧值。当然，在MYDB中没有真正的删除操作，也就是不会真的将某条记录从磁盘中删去，而是把DataItem中的标志位置为0，表示该数据已经删除。**可是磁盘中存的是DataItem吗？还是Entry？但不应该是一个普通的字节数组吗？如果是普通数组，哪里来的标志位？** <br><br>

为了防止多线程下已提交的线程读取或修改了未提交的事务的数据，使得回滚时连已提交的数据都要回滚，需要做一些规定：正在进行的事务，不会读取或修改其他任何未提交的事务产生的数据。<br><br>

**页面索引**：页面索引缓存了每一页的空闲空间，用于在上层模块进行插入操作时，能够快速找到一个**合适空间**的页面，而无需从磁盘或者缓存中检查每一个页面的信息。




### DataItem

MYDB中的缓存：
- **页面缓存**
- **DataItem缓存**：这个缓存在DM中
- **Entry缓存**：这个缓存在VM中


有两级缓存（不止），在磁盘与dm之间，有一层页面缓存；在dm与上层之间，有一层DataItem缓存。

dataItem 结构如下：
```
[ValidFlag] [DataSize] [Data]
ValidFlag 1字节，0为合法，1为非法
DataSize  2字节，标识Data的长度
```


DM中，使用read()方法读出来的都是DataItem，不是磁盘中保存的页数据。read()方法的读取过程就是，先调用AbstractCache中的get()方法，在get()方法中，又会调用到在DM中重写的getForCache()方法，DM读取来的数据是DataItem本质上讲就是在getForCache()中对读取的数据进行了包装，在这个方法里，它先会根据uid来得到整个页面，然后再根据页面、偏移量、dm等数据来创建一个DataItem对象并返回。所以上层调用read()方法后得到的就是DataItem。<br><br>

**特别需要注意的是：在创建DataItem对象之前，需要先从页面pg中得到它里面的保存数据的数组，用的是Page接口的getData()方法，它返回的是页面中数组的引用（就是一个地址）raw，也就是它并没有新创建一个对象，然后再创建DataItem对象时会用raw来作为DataItem中的SubArray中的raw，换句话说，DataItem和页面缓存中的Pg是共享数据的，对DataItem中的SubArray进行修改，会直接改变页面缓存中的数据，然后把页面缓存刷回磁盘后就完成了磁盘中数据的修改。因此上层虽然操作的是DataItem，但实质上还是在修改页面缓存中的内容。** <br><br>

那么，为什么需要DataItem呢？直接用原本的数据不行吗？另外，DataItem中除了数据外，还有很多东西，它们有什么用？
- DataItem中既有保存数据的SubArray对象，即共享数组（共享的是谁的？页面缓存的？），又有用于保存旧数据的字节数组oldRaw，用于写回滚日志？
- DataItem中有写锁和读锁，这和MySQL中的锁一样吗？加锁是为了保证原子性吗？
- DataItem中还有DM的引用，这是为了释放资源，和PageImpl中有PageCache的引用一样
- 还有一个Page引用，用于表示这个DataItem来源于那一页

向上层提供DataItem而不是Page是因为DataItem有更多的功能，能够满足上层的需要，
- 当上层读取某个数据时都会用读锁锁定，读完之后再释放读锁
- 当上层修改某个数据时也会先用写锁锁定，写完之后再释放写锁
- 每次修改数据之前，都会把旧数据复制到到DataItem中用于保存旧数据的数组，这样

上层模块在获取到 DataItem 后，可以通过 data() 方法的到数据，该方法返回的数组是数据共享的，而不是拷贝实现的，所以使用了 SubArray。

在上层模块试图对 DataItem 进行修改时，需要遵循一定的流程：在修改之前需要调用 before() 方法，想要撤销修改时，调用 unBefore() 方法，在修改完成后，调用 after() 方法。整个流程，主要是为了保存前相数据，并及时落日志。DM 会保证对 DataItem 的修改是原子性的。

read()的时候会用到页面缓存吗？如果DataItem缓存中没有要找的资源，就需要根据uid从磁盘中读取到页面资源，再创建一个DataItem对象并放入缓存，显然，从磁盘读取资源的过程肯定会先去页面缓存中查找，没有才真正去读取磁盘并写入页面缓存。<br><br>

页面缓存中key为页号，在DataItem缓存中key为uid，uid就是页号与偏移量的结合，是一个64位的数，高32位是页号，低32位是偏移量。<br><br>



## VM
**两段锁协议**（Two-Phase Locking Protocol）：
- 指事务必须分成两个阶段对数据进行加锁和解锁（扩展阶段）
- 在释放一个封锁以后，事务不在申请获得其它封锁（收缩阶段）

2PL 确实保证了调度序列的可串行化，但是不可避免地导致了事务间的相互阻塞，甚至可能导致死锁。MYDB 为了提高事务处理的效率，降低阻塞概率，实现了 MVCC。


VM的内容主要有：
- tm
- dm
- 一个Hashmap，key为xid，value为Transaction，它是VM对事务的抽象，它包含了事务的一些属性，比如事务的xid、事务的隔离级别level、创建事务时生成的快照snapshot等等



### MVCC

虽然MVCC实现了多版本，但因为VM中并没有提供update操作，因此就只有一个版本，所以一条记录（Entry）中只有一个版本。**那MVCC的多版本体现在哪？不同版本怎么储存？** 更新操作在TBM中，它好像是先用VM中的delete方法“删掉”旧记录（就是把标志位置为0），然后再将新记录插入。**但是这新旧版本是怎么关联的？** <br><br>

一条Entry的格式如下：
```
[XMIN] [XMAX] [DATA]

XMIN：创建该版本的事务编号
XMAX：删除该版本的事务编号
```

Entry这个类中内容很简单，主要就是一个事务xid和DataItem





### 隔离级别
MYDB中一共有两种隔离级别，即读已提交和可重复读，它们都是通过版本前面的XMIN和XMAX来实现的。不同之处在于读已提交是对它进行读取这个操作时已经提交的事务的数据才能够读取，而可重复读是在事务开始时就已经提交的事务的数据才能读取，在事务开始时还处于活跃状态或者在该事务之后才创建的事务的数据是不能读取的，也就是“看不见的”，在整个事务进行的过程中都是按照这一标准来的。

怎么实现读已提交和可重复读










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


**问题：上层通过传下一个long型整数来表示需要调用的页，这个整数到底是个啥？就是页与页内偏移构成的一个64位数字，用于在内存（还是磁盘？）中唯一定位一个数据** <br>
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

解决版本跳跃的思路也很简单：如果 Ti 需要修改 X，而 X 已经被 Ti 不可见的事务 Tj 修改了，那么要求 Ti 回滚。<br>
<br>




### VM的实现
VM 层通过 VersionManager 接口，向上层提供功能。接口都能像上层提供功能？<br>
<br>

VM 的实现类还被设计为 Entry 的缓存（实现类起到了缓存作用？），所以需要继承 AbstractCache<Entry>。DM 的实现类是 DataItem 的缓存？<br>
<br>

VM继承了 AbstractCache<Entry>，但缓存的功能体现在哪？似乎没看出来？可能是这样理解的，每一个继承了AbstractCache的类都自带了缓存功能，有相应的字段和方法（核心是HashMap<Long, T> cache），因此实际上已经是缓存了。

pc中的缓存键值对是 pgno——Page，DM中的缓存的键值对是uid——DataItem<br>
<br>

DM里面的pc是PageCache，页面缓存，因为PageCacheImpl类继承了AbstractCache，实际缓存时其中的 HashMap<Long, Page> cache，也就是它缓存的是页号与页（页是一个类，包含了数据与方法）；DM中的缓存是 HashMap<Long, DataItem> cache，缓存的是通过pc读取到的page对象，然后将page中的data包装成DataItem，放入缓存并返回，读的时候，如果pc缓存中已经有相应的page了，就直接从缓存中拿，如果没有，就需要从db文件中读取、放入pc缓存并返回给DM，DM再包装成DataItem，放入DM中的缓存并返回。<br>
<br>

也就是说有两级缓存。整体结构是，db文件（磁盘）——pc——DM缓存。下次DM需要读取某个DataItem时，直接利用uid从缓存中读取，如果缓存中没有，再调用getForCache，从pc中拿，而从pc中怎么拿就是pc的事了，pc缓存中有就直接返回，没有就还是从db中读并写入缓存。<br>
<br>

总结下来，VM中的读取删除数据本质上都是借用DM中的方法（毕竟DM是数据管理器，所有的数据操作都要经过他，VM实现的是版本的管理）<br>
<br>

abort 事务的方法则有两种，手动和自动。手动指的是调用 abort() 方法，而自动，则是在事务被检测出出现死锁时，会自动撤销回滚事务；或者出现版本跳跃时，也会自动回滚。<br>
<br>

我们要通过事务xid来删除掉这个资源，所以首先需要这个事务获取到这个资源，然后才能删除，而且，如教程所说，删除一个资源在MYDB里也并不是直接通过删掉对应的磁盘数据来实现，而是通过将Xmax设为当前事务xid来实现，这样这个事务之后或正活跃的事务就“看不见”这个数据了，而看不见的功能是在read方法中实现的。<br>
<br>

如果entry不释放的话就会一直累积，导致占用不必要的内存？就像PageCache也要及时关闭一样？<br>
<br>


异常测试：
```java
// 如果添加后会出现死锁，抛出运行时异常Error.DeadlockException（自定义的运行时异常），那么测试通过
assertThrows(RuntimeException.class, ()->lt.add(1, 2));
```




# Index Manager(IM)

索引管理器IM，为 MYDB 提供了基于 B+ 树的聚簇索引。目前 MYDB 只支持基于索引查找数据，不支持全表扫描。<br>
<br>

在依赖关系图中可以看到，IM 直接基于 DM，而没有基于 VM。索引的数据被直接插入数据库文件中，而不需要经过版本管理。<br>
<br>

- B+ 树非叶子节点上是不存储数据的，仅存储键值，而 B 树节点中不仅存储键值，也会存储数据。
- B+ 树索引的所有数据均存储在叶子节点，而且数据是按照顺序排列的。

如果不存储数据，那么就会存储更多的键值，相应的树的阶数（节点的子节点树）就会更大，树就会更矮更胖，如此一来我们查找数据进行磁盘的 IO 次数又会再次减少，数据查询的效率也会更快。
另外，B+ 树的阶数是等于键值的数量的，如果我们的 B+ 树一个节点可以存储 1000 个键值，那么 3 层 B+ 树可以存储 1000×1000×1000=10 亿个数据。<br>
<br>

>> 参考文章：![B+树](https://www.cnblogs.com/cangqinglang/p/15042752.html)<br>
>> ![](https://zhuanlan.zhihu.com/p/86137284)

有一道 MySQL 的面试题，为什么 MySQL 的索引要使用 B+ 树而不是其它树形结构？比如 B 树？<br>
<br>

简单版本回答：因为 B 树不管叶子节点还是非叶子节点，都会保存数据，这样导致在非叶子节点中能保存的指针数量变少（有些资料也称为扇出），指针少的情况下要保存大量数据，只能增加树的高度，导致 IO 操作变多，查询性能变低。<br>
<br>


### 二叉树索引
二叉树由一个个 Node 组成，每个 Node 都存储在一条 DataItem 中。结构如下：

```
[LeafFlag][KeyNumber][SiblingUid]
[Son0][Key0][Son1][Key1]...[SonN][KeyN]
```

最后的一个 KeyN 始终为 MAX_VALUE，以此方便查找。这个方便体现在哪里？<br>
<br>
答：要理解这个方便性，首先需要注意，这个keyN并不是每一个节点都会有的一个key，而是整个B+树中的最后一个key。如果我们想要搜索B+树中key的范围在left到right之间（均含）的数据，我们当然调用树中的search方法。同样，如果我们想要整个树中的所有数据（就是当没有Where语句时），那left=0，而right则是那个最大的key，但显然，我们不能很方便地得到这个最大的key，然后再search。如果keyN始终为MAX_VALUE的话，只要取right为MAX_VALUE就能得到所有key对应的行数据了，但会不会多了一行空数据（对应keyN）？另外，有了这个最大的keyN，在进行insert操作时，所有的新数据都会插在keyN前面，因为他们的key一定小于MAX_VALUE（keyN）。<br>
<br>

一棵树的高度就是直觉上的那个高度，也就是说包含了根节点，比如一个高度为2的B+树就是一个根节点和下面的叶子结点。<br>
<br>

可以注意到，IM 在操作 DM 时，使用的事务都是 SUPER_XID。<br>
<br>


Tokenizer 类，对语句进行逐字节解析，根据空白符或者上述词法规则，将语句切割成多个 token。对外提供了 peek()、pop() 方法方便取出 Token 进行解析。<br>
<br>

Parser 类则直接对外提供了 Parse(byte[] statement) 方法，核心就是一个调用 Tokenizer 类分割 Token，并根据词法规则包装成具体的 Statement 类（比如Where类、Select类，在TBM中会有具体的方法来利用这些数据结构中的数据，根据这些输入的条件，再调用VM中的方法来实现begin、commit、update、delete、insert等功能，这些功能就是TM接口中的提供给外界的抽象方法）并返回。<br>
<br>


### 字段与表管理

由于 TBM 基于 VM，单个字段信息和表信息都是直接保存在 Entry 中。字段的二进制表示如下：
```
[FieldName][TypeName][IndexUid]
```


FieldName和TypeName，以及后面的表名，都是以字符串的形式储存的，这里规定这些字符串的存储方式，以明确其存储边界。
```
[StringLength][StringData]
```



TypeName 为字段的类型，限定为 int32、int64 和 string 类型。如果这个字段有索引，那个 IndexUID 指向了索引二叉树的根，否则该字段为 0。<br>
<br>

**疑问之处：在parseSelf()方法内多减了个8，不确定是否正确！！！**


一个数据库中存在多张表，TBM 使用链表的形式将其组织起来，每一张表都保存一个指向下一张表的 UID。表的二进制结构如下：
```
[TableName][NextTable]
[Field1Uid][Field2Uid]...[FieldNUid]
```

这里由于每个 Entry 中的数据，字节数是确定的，于是无需保存字段的个数。（**为什么？？？**）<br>
<br>


由于 TBM 的表管理，使用的是链表串起的 Table 结构，所以就必须保存一个链表的头节点，即第一个表的 UID，这样在 MYDB 启动时，才能快速找到表信息。<br>
<br>

MYDB 使用 Booter 类和 bt 文件，来管理 MYDB 的启动信息，虽然现在所需的启动信息，只有一个：头表的 UID。<br>
<br>

Booter 类对外提供了两个方法：load 和 update，并保证了其原子性。update 在修改 bt 文件内容时，没有直接对 bt 文件进行修改，而是首先将内容写入一个 bt_tmp 文件中，随后将这个文件重命名为 bt 文件。以期通过操作系统重命名文件的原子性，来保证操作的原子性。<br>
<br>

需要注意的是，在创建新表时，采用的是头插法，所以每次创建表都需要更新 Booter 文件。这就像哈希表中插入时一样，哈希冲突时把同一个桶中的元素（Entry？）变成链表，这个链表也是采用头插法，这样的话下次插入时就不用遍历到链表最后一位再插入，提高效率。

修改文件名：
```java
Files.move(tmp.toPath(), new File(path+BOOTER_SUFFIX).toPath(), StandardCopyOption.REPLACE_EXISTING);
```

REPLACE_EXISTING：Replace an existing file if it exists.


Table类中parseWhere()方法的核心就是通过where条件中的数据来确定左右边界，再根据B+树中的search()方法来找到符合where条件的数据。一个很需要注意的点，由于是专门利用有索引的字段进行parseWhere，所以该字段的value就是B+树的key，左右边界也是这样的值。<br>
<br>

读取数据（read()方法）用的也只是B+树，在MYDB中，只有通过索引才能读取数据，通过其他字段是无法读取的。同样，插入时（insert()方法）也只是插入到B+树中就完成了，当然，插入需要用到key和uid，key就在输入数据中，输入的一行数据中，索引字段部分的值就是key，利用它以及uid（通过利用VM将输入数据写入磁盘来得到）就可插入到B+树中。<br>
<br>

delete直接借助VM中的delete方法来删除，不用操作B+树了？<br>
<br>

update()方法的思路大概是，先用VM中的delete方法，再用VM中的insert方法，最后再在B+树中插入新数据（原来那个不用删？）<br>
<br>




# 服务端客户端的实现及其通信规则

NYDB 被设计为 C/S 结构，类似于 MySQL。通过 socket 通信<br>
<br>

传输的最基本结构，是 Package：
```java
public class Package {
    byte[] data;
    Exception err;
}
```


编码和解码的规则如下：
```
[Flag][data]
```

编码之后的信息会通过 Transporter 类，写入输出流发送出去。<br>
<br>

为了避免特殊字符造成问题，这里会将数据转成十六进制字符串（Hex String），并为信息末尾加上换行符。这样在发送和接收数据时，就可以很简单地使用 BufferedReader 和 Writer 来直接按行读写了。<br>
<br>

将字节数组或字符串转换为16进制可以用 org.apache.commons.codec.binary 包，也可以将16进制转换为2进制，在pom.xml中引入以下依赖即可：
```xml
<dependency>
  <groupId>commons-codec</groupId>
  <artifactId>commons-codec</artifactId>
  <version>1.15</version>
</dependency>
```

比如，`Hex.encodeHexString(buf, true)+"\n"`<br>
<br>



服务器做的事很简单，就是根据客户端传过来（send）的数据，调用Executor中的execute()方法执行TBM中的那些方法（见TBM接口）：begin、commit、abort、show、create、insert、read、update、delete。TBM中的这些方法的实现又是通过调用VM中的方法以及B+tree中的insert、search等方法来实现的。<br>
<br>

Client类中主要就是一个execute()方法，用于处理输入的命令。Client类不是单独使用的，而是在Shell类（Shell里有一个while(true)循环）中使用，Shell是用来读入用户的输入（用Scanner.nextLine()方法），并调用 Client.execute()<br>
<br>

























