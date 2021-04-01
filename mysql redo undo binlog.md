###  mysql

事务中的redo undo

原子性实现原理 undolog



- **逻辑日志**：可以简单理解为记录的就是sql语句 。

- **物理日志**：`mysql` 数据最终是保存在数据页中的，物理日志记录的就是数据页变更 。

  

### redo log

**大部分情况下 Redo是物理日志，记录的是数据页的物理变化**，Redo log的主要作用是用于**数据库的崩溃恢复**，实现了持久性。

Redo log可以简单分为以下两个部分：

- 一是内存中重做日志缓冲 (redo log buffer),是易失的，在内存中
- 二是重做日志文件 (redo log file)，是持久的，保存在磁盘中

### 什么时候写Redo?

- 在数据页修改完成之后，在脏页刷出磁盘之前，写入redo日志。注意的是先修改数据，后写日志
- **redo日志比数据页先写回磁盘**
- 聚集索引、二级索引、undo页面的修改，均需要记录Redo日志。

### Redo的整体流程

下面以一个更新事务为例，宏观上把握redo log 流转过程，如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/5652417-a5f90ca64ed10d4d.png?imageMogr2/auto-orient/strip|imageView2/2/w/662/format/webp)



- 第一步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝

- 第二步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值

- 第三步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加写的方式

- 第四步：定期将内存中修改的数据刷新到磁盘中

  

  

  **redolog 的意义**  预写式日志，通过将操作顺序写入到日志中，减少了写入到数据的磁盘读取。

  

**redo log 格式**

因为innodb存储引擎存储数据的单元是页(和SQL Server中一样)，所以redo log也是**基于页的格式**来记录的。默认情况下，innodb的页大小是16KB(由 innodb_page_size 变量控制)，一个页内可以存放非常多的log block(每个512字节)，而log block中记录的又是数据页的变化。

## undo log

undolog是为了实现事务的原子性 是逻辑日志，在mysqlinnodb引擎中，还用其来实现mvcc

undo log主要记录的是数据的逻辑变化，为了在发生错误时回滚之前的操作，需要将之前的操作都记录下来，然后在发生错误时才可以回滚。

undo日志，只将数据库**逻辑**地恢复到原来的样子，它实际上是做的相反的工作，比如一条INSERT ，对应一条 DELETE，对于每个UPDATE,对应一条相反的 UPDATE,将修改前的行放回去。undo日志用于事务的回滚操作进而保障了事务的原子性

- ### undo log的写入时机

  - DML操作修改聚簇索引前，记录undo日志
  - 二级索引记录的修改，不记录undo日志

  需要注意的是，undo页面的修改，同样需要记录redo日志。



### undo的存储位置

在InnoDB存储引擎中，undo存储在回滚段(Rollback Segment)中,每个回滚段记录了1024个undo log segment，而在每个undo log segment段中进行undo 页的申请



### undo的类型

在InnoDB存储引擎中，undo log分为：

- insert undo log
- update undo log

insert undo log是指在insert 操作中产生的undo log，因为insert操作的记录，只对事务本身可见，对其他事务不可见。故该undo log可以在事务提交后直接删除，不需要进行purge操作。

而update undo log记录的是对delete 和update操作产生的undo log，该undo log可能需要提供MVCC机制，因此不能再事务提交时就进行删除。提交时放入undo log链表，等待purge线程进行最后的删除。



### Binlog

binlog用于记录数据库执行的写入性操作 以二进制的形式保存在磁盘中，binlog是逻辑日志，任何存储引擎的mysql数据库都会记录binlog日志，binlog是通过追加的方式进行写入的，当文件大小达到给定值后，会生成新的文件来保存日志。

