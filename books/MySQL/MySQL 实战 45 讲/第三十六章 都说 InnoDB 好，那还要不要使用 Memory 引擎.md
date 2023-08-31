# 第三十六章 都说 InnoDB 好，那还要不要使用 Memory 引擎

## 36-1 背景

1. 我在上一篇文章末尾留给你的问题是：两个 group by 语句都用了 order by null ，为什么使用内存临时表得到的语句结果里，0 这个值在最后一行；而使用磁盘临时表得到的结果里，0 这个值在第一行？
2. 今天我们就来看看，出现这个问题的原因吧。

## 36-2 内存表的数据组织结构

1. 为了便于分析，我来把这个问题简化一下，假设有以下的两张表 t1 和 t2 ，其中表 t1 使用 Memory 引擎， 表 t2 使用 InnoDB 引擎。

2. 然后，我分别执行 `select * from t1` 和 `select * from t2` 。

3. 可以看到，内存表 t1 的返回结果里面0在最后一行，而 InnoDB 表 t2 的返回结果里 0 在第一行。

4. 出现这个区别的原因，要从这两个引擎的主键索引的组织方式说起。

5. 表 t2 用的是 InnoDB 引擎，它的主键索引 id 的组织方式，你已经很熟悉了：InnoDB 表的数据就放在主键索引树上，主键索引是 B+ 树。所以表 t2 的数据组织方式如下图所示：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E4%B8%A4%E4%B8%AA%E6%9F%A5%E8%AF%A2%E7%BB%93%E6%9E%9C-0%E7%9A%84%E4%BD%8D%E7%BD%AE.png" alt="两个查询结果-0的位置" style="zoom:50%;" />

6. 可以看到，内存表 t1 的返回结果里面 0 在最后一行，而 InnoDB 表 t2 的返回结果里 0 在第一行。

7. 出现这个区别的原因，要从这两个引擎的主键索引的组织方式说起。

8. 表 t2 用的是 InnoDB 引擎，它的主键索引 id 的组织方式，你已经很熟悉了：InnoDB 表的数据就放在主键索引树上，主键索引是 B+ 树。所以表 t2 的数据组织方式如下图所示：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%A1%A8t2%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%84%E7%BB%87.png" alt="表t2的数据组织" style="zoom:50%;" />

9. 主键索引上的值是有序存储的。在执行 `select *` 的时候，就会按照叶子节点从左到右扫描，所以得到的结果里，0 就出现在第一行。

10. 与 InnoDB 引擎不同，Memory 引擎的数据和索引是分开的。我们来看一下表 t1 中的数据内容。

    <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%A1%A8t1%20%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%84%E7%BB%87.png" alt="表t1 的数据组织" style="zoom:50%;" />

11. 可以看到，内存表的数据部分以数组的方式单独存放，而主键id索引里，存的是每个数据的位置。主键id是hash索引，可以看到索引上的key并不是有序的。

12. 在内存表 t1 中，当我执行 `select *` 的时候，走的是全表扫描，也就是顺序扫描这个数组。因此，0 就是最后一个被读到，并放入结果集的数据。

    可见，InnoDB 和 Memory 引擎的数据组织方式是不同的：

    - InnoDB 引擎把数据放在主键索引上，其他索引上保存的是主键 id 。这种方式，我们称之为**索引组织表**(Index Organizied Table)。
    - 而 Memory 引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，我们称之为**堆组织表**(Heap Organizied Table)。

13. 从中我们可以看出，这两个引擎的一些典型不同：

    - InnoDB 表的数据总是有序存放的，而内存表的数据就是按照写入顺序存放的；

    - 当数据文件有空洞的时候，InnoDB 表在插入新数据的时候，为了保证数据有序性，只能在固定的位置写入新值，而内存表找到空位就可以插入新值；

    - 数据位置发生变化的时候，InnoDB 表只需要修改主键索引，而内存表需要修改所有索引；

    - InnoDB 表用主键索引查询时需要走一次索引查找，用普通索引查询的时候，需要走两次索引查找。而内存表没有这个区别，所有索引的地位都是相同的。

    - InnoDB 支持变长数据类型，不同记录的长度可能不同；内存表不支持 Blob 和 Text 字段，并且即使定义了 varchar(N) ，实际也当作 char(N) ，也就是固定长度字符串来存储，因此内存表的每行数据长度相同。

14. 由于内存表的这些特性，每个数据行被删除以后，空出的这个位置都可以被接下来要插入的数据复用。比如，如果要在表 t1 中执行：

    ```mysql
    delete from t1 where id=5;
    insert into t1 values(10,10);
    select * from t1;
    ```

15. 就会看到返回结果里，id=10 这一行出现在 id=4 之后，也就是原来 id=5 这行数据的位置。

16. 需要指出的是，表 t1 的这个主键索引是哈希索引，因此如果执行范围查询，比如

    ```mysql
    select * from t1 where id<5;
    ```

17. 是用不上主键索引的，需要走全表扫描。你可以借此再回顾下第 4 篇文章的内容。那如果要让内存表支持范围扫描，应该怎么办呢 ？

## 36-3 hash 索引和 B-Tree 索引

1. 实际上，内存表也是指 B-Tree 索引的。在 id 列上创建一个 B-Tree 索引，SQL 语句可以这么写：

   ```mysql
   alter table t1 add index a_btree_index using btree (id);
   ```

2. 这时，表 t1 的数据组织形式就变成了这样：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E8%A1%A8t1%E7%9A%84%E6%95%B0%E6%8D%AE%E7%BB%84%E7%BB%87--%E5%A2%9E%E5%8A%A0B-Tree%E7%B4%A2%E5%BC%95.png" alt="表t1的数据组织--增加B-Tree索引" style="zoom:50%;" />

3. 新增的这个 B-Tree 索引你看着就眼熟了，这跟 InnoDB 的 b+ 树索引组织形式类似。

4. 作为对比，你可以看一下这下面这两个语句的输出：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E4%BD%BF%E7%94%A8B-Tree%E5%92%8Chash%E7%B4%A2%E5%BC%95%E6%9F%A5%E8%AF%A2%E8%BF%94%E5%9B%9E%E7%BB%93%E6%9E%9C%E5%AF%B9%E6%AF%94.png" alt="使用B-Tree和hash索引查询返回结果对比" style="zoom:50%;" />

5. 可以看到，执行 `select * from t1 where id < 5` 的时候，优化器会选择 B-Tree 索引，所以返回结果是 0 到 4 。 使用 force index 强行使用主键 id 这个索引，id=0 这一行就在结果集的最末尾了。

6. 其实，一般在我们的印象中，内存表的优势是速度快，其中的一个原因就是 Memory 引擎支持 hash 索引。当然，更重要的原因是，内存表的所有数据都保存在内存，而内存的读写速度总是比磁盘快。

7. 但是，接下来我要跟你说明，为什么我不建议你在生产环境上使用内存表。这里的原因主要包括两个方面：

   - 锁粒度问题；

   - 数据持久化问题。

## 36-4 内存表的锁

1. 我们先来说说内存表的锁粒度问题。

2. 内存表不支持行锁，只支持表锁。因此，一张表只要有更新，就会堵住其他所有在这个表上的读写操作。

3. 需要注意的是，这里的表锁跟之前我们介绍过的 MDL 锁不同，但都是表级的锁。接下来，我通过下面这个场景，跟你模拟一下内存表的表级锁。

   |               session A                |               session B                |     session C     |
   | :------------------------------------: | :------------------------------------: | :---------------: |
   | update t1 set id=sleep(50) where id=1; |                                        |                   |
   |                                        | select * from t1 where id=2;(wait 50s) |                   |
   |                                        |                                        | show processlist; |

4. 在这个执行序列里，session A 的 update 语句要执行 50 秒，在这个语句执行期间 session B 的查询会进入锁等待状态。session C 的 show processlist 结果输出如下：

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%86%85%E5%AD%98%E8%A1%A8%E7%9A%84%E8%A1%A8%E9%94%81--%E7%BB%93%E6%9E%9C.png" alt="内存表的表锁--结果" style="zoom:50%;" />

5. 跟行锁比起来，表锁对并发访问的支持不够好。所以，内存表的锁粒度问题，决定了它在处理并发事务的时候，性能也不会太好。

## 36-5 数据持久性问题

1. 接下来，我们再看看数据持久性的问题。

2. 数据放在内存中，是内存表的优势，但也是一个劣势。因为，数据库重启的时候，所有的内存表都会被清空。

3. 你可能会说，如果数据库异常重启，内存表被清空也就清空了，不会有什么问题啊。但是，在高可用架构下，内存表的这个特点简直可以当做 bug 来看待了。为什么这么说呢？

4. **我们先看看 M-S 架构下，使用内存表存在的问题。**

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/M-S%20%E5%9F%BA%E6%9C%AC%E6%9E%B6%E6%9E%84.png" alt="M-S 基本架构" style="zoom:50%;" />

5. 我们来看一下下面这个时序：

   - 业务正常访问主库；

   - 备库硬件升级，备库重启，内存表t1内容被清空；

   - 备库重启后，客户端发送一条 update 语句，修改表 t1 的数据行，这时备库应用线程就会报错找不到要更新的行。

6. 这样就会导致主备同步停止。当然，如果这时候发生主备切换的话，客户端会看到，表t1的数据“丢失”了。

7. 在上图中这种有 proxy 的架构里，大家默认主备切换的逻辑是由数据库系统自己维护的。这样对客户端来说，就是网络断开，重连之后，发现内存表数据丢失了。

8. 你可能说这还好啊，毕竟主备发生切换，连接会断开，业务端能够感知到异常。

9. 但是，接下来内存表的这个特性就会让使用现象显得更诡异了。由于 MySQL 知道重启之后，内存表的数据会丢失。所以，担心主库重启之后，出现主备不一致，MySQL 在实现上做了这样一件事儿：在数据库重启之后，往 binlog 里面写入一行 DELETE FROM t1 。

10. **如果你使用是下图所示的双M结构的话：**

    <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%8F%8C%20M%20%E7%BB%93%E6%9E%84.png" alt="双 M 结构" style="zoom:50%;" />

11. 在备库重启的时候，备库 binlog 里的 delete 语句就会传到主库，然后把主库内存表的内容删除。这样你在使用的时候就会发现，主库的内存表数据突然被清空了。

12. 基于上面的分析，你可以看到，内存表并不适合在生产环境上作为普通数据表使用。

13. 有同学会说，但是内存表执行速度快呀。这个问题，其实你可以这么分析：

    - 如果你的表更新量大，那么并发度是一个很重要的参考指标，InnoDB 支持行锁，并发度比内存表好；

    - 能放到内存表的数据量都不大。如果你考虑的是读的性能，一个读 QPS 很高并且数据量不大的表，即使是使用 InnoDB ，数据也是都会缓存在 InnoDB Buffer Pool 里的。因此，使用 InnoDB 表的读性能也不会差。

14. 所以，**我建议你把普通内存表都用 InnoDB 表来代替。**但是，有一个场景却是例外的。

15. 这个场景就是，我们在第 33 和 34 篇说到的用户临时表。在数据量可控，不会耗费过多内存的情况下，你可以考虑使用内存表。

16. 内存临时表刚好可以无视内存表的两个不足，主要是下面的三个原因：

    - 临时表不会被其他线程访问，没有并发性的问题；

    - 临时表重启后也是需要删除的，清空数据这个问题不存在；

    - 备库的临时表也不会影响主库的用户线程。

17. 现在，我们回过头再看一下第 33 篇 join 语句优化的例子，当时我建议的是创建一个 InnoDB 临时表，使用的语句序列是：

    ```mysql
    create temporary table temp_t(id int primary key, a int, b int, index(b))engine=innodb;
    insert into temp_t select * from t2 where b>=1 and b<=2000;
    select * from t1 join temp_t on (t1.b=temp_t.b);
    ```

18. 了解了内存表的特性，你就知道了， 其实这里使用内存临时表的效果更好，原因有三个：

    - 相比于 InnoDB 表，使用内存表不需要写磁盘，往表 temp_t 的写数据的速度更快；

    - 索引 b 使用 hash 索引，查找的速度比 B-Tree 索引快；

    - 临时表数据只有 2000 行，占用的内存有限。

19. 因此，你可以对第 33 篇文章的语句序列做一个改写，将临时表 t1 改成内存临时表，并且在字段b上创建一个 hash 索引。

    ```mysql
    create temporary table temp_t(id int primary key, a int, b int, index (b))engine=memory;
    insert into temp_t select * from t2 where b>=1 and b<=2000;
    select * from t1 join temp_t on (t1.b=temp_t.b);
    ```

    ![使用内存临时表的执行效果](https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E4%BD%BF%E7%94%A8%E5%86%85%E5%AD%98%E4%B8%B4%E6%97%B6%E8%A1%A8%E7%9A%84%E6%89%A7%E8%A1%8C%E6%95%88%E6%9E%9C.png)

20. 可以看到，不论是导入数据的时间，还是执行 join 的时间，使用内存临时表的速度都比使用 InnoDB 临时表要更快一些。
