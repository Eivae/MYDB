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



#### 接口和实现类的一些问题
>> ![Java 中到底是应该用接口类型 还是实现类的类类型去引用对象？](https://blog.csdn.net/summerxiachen/article/details/79733800)

结论：**应该优先使用接口而不是类来引用对象，但只有存在适当的接口类型时，即实现类中全是接口类的方法的实现，没有自己单独的方法。当实现类存在自己的方法时，使用实现类来声明变量。**

如果用的是接口类型来引用对象，而实现类中有自己单独的方法，那么可以通过向下转型来使用实现类的方法。<br>
<br>










