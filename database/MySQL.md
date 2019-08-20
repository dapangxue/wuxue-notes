# MySQL

## 一、事务

事务表示了一组原子操作，这组操作具有原子性，要么全部执行成功，要么回滚到原来的状态，不会存在一部分成功，一部分失败的状态。主要用来处理操作量大，复杂度高的数据，比如删除一个信息管理系统中的一个人员，除了删除个人信息，对应的信箱、文章等等其他信息也要一并删除。

显式事务

```SQL
# 禁止自动提交事务
SET AUTOCOMMIT = 0 
# 开启自动提交
SET AUTOCOMMIT = 1
```

```SQL
# 开启事务
BEGIN

# 相关事务操作
...

# 回滚事务
ROLLBACK

# 提交事务
COMMIT
```



### 1.事务四大特性

+ 原子性
+ 一致性，事务在开始执行之前和结束执行之后，数据库的完整性没有被破坏。
+ 隔离性，数据库允许多个事务并发访问数据库的读写和修改的能力。
+ 持久性，事务一经提交不可修改，即使数据库系统发生崩溃也不会改变已经提交过的数据。

### 2.redo log

说过的事情就要办到。

MySQL事务提交数据后并不是马上将事务更改的页面更新回磁盘，这样做的好处就是

+ 每次事务提交完就直接更新刷新一个完整的数据页太浪费了，假设只修改了一个页面的一个字节，对于Innodb来说刷新数据是以页为单位的，I/O一次16kb只为了更新一个字节的数据，开销太大。
+ 事务所修改的数据的页面可能是多个且不连续的，随机I/O更慢。

所以就有了redo log，称为重做日志。它将事务执行过程中执行语句产生的日志顺序存储下来。本质上redo log记录了事务对数据库做了哪些修改。之后如果系统崩溃重启后，可以根据redo log将没有来得及更新的修改性数据写入对应的数据表。

### 3.undo log

事务执行过程中后悔了，Roll Back

undo log主要是为了应对事务的回滚操作的，本质上就是将回滚时所需的东西都记录下来。

要理解undo log就需要理解Innodb引擎一行数据结构的隐藏列`transaction_id`和`roll_pointer`,transaction_id表示创建（此处的意思不是插入操作）当前这一行数据的事务版本号，roll_pointer是一个指针，指向了对应的undo log。

MySQL对INSERT、DELETE、UPDATE做了不同格式的undo log。

#### INSERT操作对应的undo log

对于插入一条数据，如果希望回滚，那么在undo log中存储插入数据的主键即可，在需要回滚的时候，将undo log中存储的主键对应的数据删除。在undo log日志中存储了`主键所占的存储空间大小`和`真实值`。

在一个事务中的undo no都是从0开始顺序编号的，只要事务没有提交。每生成一条undo log

```SQL
# 显式的开启一个事务
BEGIN;
INSERT INTO undo_demo (id, key1, col) VALUES (1, 'WUXUE', 'DAPANGXUE'),(2, 'HUANGTAO', 'SIPANGZI');
```

此时会在undo log中记录两条数据

+ 第一条数据的undo no为0，记录了主键的存储空间长度和真实的值
+ 第二条数据的undo no为1，也记录了主键的存储空间长度和真实的值

#### DELETE操作对应的undo log

在DELETE操作中，undo log结构存储的旧的trx_id和roll_pointer，在结构中表示为`old trx_id`和`old roll_pointer`。保存了主键各列信息和索引格列信息，这也是它和INSERT undo log的不同点，INSERT undo log是没有保存索引格列信息的，同时在undo log生成的日志编号也是按照当前的事务顺序编号，比如在下面的例子中undo log中的生成的日志编号为2。

```sql
# 显示开启一个事务
BEGIN;

# 插入两条数据
INSERT INTO undo_demo (id, key1, col) VALUES (1, 'WUXUE', 'DAPANGXUE'),(2, 'HUANGTAO', 'SIPANGZI');

# 删除一条数据
DELETE FROM undo_demo WHERE id = 1;
```

然后在undo log中的信息组成了一个链表的形式，称为版本链。

#### UPDATE操作对应的undo log

UPDATE操作需要划分为`更新主键的操作`和`不更新主键`的操作。

更新主键的操作相当于先`delete_mark`这一条数据，再插入一条新数据，由于delete_mark也需要写入undo log，所以对应的undo log中会保存两条undo日志。

不更新主键的操作又划分为就地更新和先删除旧记录，再插入新纪录。

+ 就地更新发生在更新一条数据前后对应的所有列所占的存储空间和之前的数据一样大。
+ 先删除旧记录，再插入新纪录，就是将这条记录真正的删除掉，在创建一条新数据。