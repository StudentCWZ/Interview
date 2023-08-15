# 第十五章 order by 是怎么工作的？

## 15.1 背景

1. 在你开发应用的时候，一定会经常碰到需要根据指定的字段排序来显示结果的需求。还是以我们前面举例用过的市民表为例，假设你要查询城市是杭州的所有人名字，并且按照姓名排序返回前 1000 个人的姓名、年龄。

2. 假设这个表的部分定义是这样的：

   ```mysql
   CREATE TABLE `t` (
     `id` int(11) NOT NULL,
     `city` varchar(16) NOT NULL,
     `name` varchar(16) NOT NULL,
     `age` int(11) NOT NULL,
     `addr` varchar(128) DEFAULT NULL,
     PRIMARY KEY (`id`),
     KEY `city` (`city`)
   ) ENGINE=InnoDB;
   ```

3. 这时，你的 **SQL** 语句可以这么写：

   ```mysql
   select city,name,age from t where city='杭州' order by name limit 1000;
   ```

4. 这个语句看上去逻辑很清晰，但是你了解它的执行流程吗？今天，我就和你聊聊这个语句是怎么执行的，以及有什么参数会影响执行的行为。

## 15.2 全字段排序

1. 前面我们介绍过索引，所以你现在就很清楚了，为避免全表扫描，我们需要在 city 字段加上索引。

2. 在 **city** 字段上创建索引之后，我们用 **explain** 命令来看看这个语句的执行情况。

   ```mysql
   mysql> explain select city,name,age from t where city='杭州' order by name limit 1000  ;
   +----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+----------------+
   | id | select_type | table | partitions | type | possible_keys | key  | key_len | ref   | rows | filtered | Extra          |
   +----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+----------------+
   |  1 | SIMPLE      | t     | NULL       | ref  | city          | city | 66      | const | 4000 |   100.00 | Using filesort |
   +----+-------------+-------+------------+------+---------------+------+---------+-------+------+----------+----------------+
   1 row in set, 1 warning (0.00 sec)
   ```

3. Extra 这个字段中的 **Using filesort** 表示的就是需要排序，MySQL 会给每个线程分配一块内存用于排序，称为 **sort_buffer** 。

4. 为了说明这个 SQL 查询语句的执行过程，我们先来看一下 city 这个索引的示意图。

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/city%20%E5%AD%97%E6%AE%B5%E7%9A%84%E7%B4%A2%E5%BC%95%E7%A4%BA%E6%84%8F%E5%9B%BE.png" alt="city 字段的索引示意图" style="zoom:50%;" />

5. 从图中可以看到，满足 city='杭州' 条件的行，是从 ID_X 到 ID_(X+N) 的这些记录。

6. 通常情况下，这个语句执行流程如下所示：

   - 初始化 sort_buffer ，确定放入 name、city、age 这三个字段；
   - 从索引 city 找到第一个满足 city='杭州' 条件的主键 id，也就是图中的 ID_X；
   - 到主键 id 索引取出整行，取 name、city、age 三个字段的值，存入 sort_buffer 中；
   - 从索引 city 取下一个记录的主键 id ；
   - 重复步骤 3、4 直到 city 的值不满足查询条件为止，对应的主键 id 也就是图中的 ID_Y；
   - 对 sort_buffer 中的数据按照字段 name 做快速排序；
   - 按照排序结果取前 1000 行返回给客户端。

7. 我们暂且把这个排序过程，称为全字段排序，执行流程的示意图如下所示，下一篇文章中我们还会用到这个排序。

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%85%A8%E5%AD%97%E6%AE%B5%E6%8E%92%E5%BA%8F.png" alt="全字段排序" style="zoom:50%;" />

8. 图中按 name 排序这个动作，可能在内存中完成，也可能需要使用外部排序，这取决于排序所需的内存和参数 **sort_buffer_size** 。

9. **sort_buffer_size**，就是 MySQL 为排序开辟的内存(sort_buffer)的大小。如果要排序的数据量小于 **sort_buffer_size**，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。

10. 你可以用下面介绍的方法，来确定一个排序语句是否使用了临时文件。

    ```mysql
    /* 打开 optimizer_trace ，只对本线程有效 */
    SET optimizer_trace='enabled=on';

    /* @a 保存 Innodb_rows_read 的初始值 */
    select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';

    /* 执行语句 */
    select city, name,age from t where city='杭州' order by name limit 1000;

    /* 查看 OPTIMIZER_TRACE 输出 */
    SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

    /* @b 保存 Innodb_rows_read 的当前值 */
    select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';

    /* 计算 Innodb_rows_read 差值 */
    select @b-@a;
    ```

11. 这个方法是通过查看 OPTIMIZER_TRACE 的结果来确认的，你可以从 **number_of_tmp_files** 中看到是否使用了临时文件。

    ```json
    {
      "filesort_summary": {
        "rows": 4000,
        "examined_rows": 4000,
        "number_of_tmp_files": 12,
        "sort_buffer_size": 32768,
        "sort_mode": "<sort_key, packed_additional_fields>"
      }
    }
    ```

12. number_of_tmp_files 表示的是，排序过程中使用的临时文件数。你一定奇怪，为什么需要 12 个文件？内存放不下时，就需要使用外部排序，外部排序一般使用归并排序算法。可以这么简单理解，**MySQL 将需要排序的数据分成 12 份，每一份单独排序后存在这些临时文件中。然后把这 12 个有序文件再合并成一个有序的大文件。**
13. 如果 **sort_buffer_size** 超过了需要排序的数据量的大小，**number_of_tmp_files** 就是 0 ，表示排序可以直接在内存中完成。
14. 否则就需要放在临时文件中排序。**sort_buffer_size** 越小，需要分成的份数越多，**number_of_tmp_files** 的值就越大。
15. 我们的示例表中有 4000 条满足 city='杭州' 的记录，所以你可以看到 examined_rows=4000，表示参与排序的行数是 4000 行。
16. sort_mode 里面的 **packed_additional_fields** 的意思是，排序过程对字符串做了紧凑处理。即使 name 字段的定义是 varchar(16) ，在排序过程中还是要按照实际长度来分配空间的。
17. 同时，最后一个查询语句 select @b-@a 的返回结果是 4000 ，表示整个执行过程只扫描了 4000 行。
18. 这里需要注意的是，为了避免对结论造成干扰，我把 **internal_tmp_disk_storage_engine** 设置成 MyISAM 。否则，select @b-@a 的结果会显示为 4001 。
19. 这是因为查询 **OPTIMIZER_TRACE** 这个表时，需要用到临时表，而 **internal_tmp_disk_storage_engine** 的默认值是 **InnoDB**。如果使用的是 **InnoDB** 引擎的话，把数据从临时表取出来的时候，会让 **Innodb_rows_read** 的值加 1 。

## 15.3 rowid 排序

1. 在上面这个算法过程里面，只对原表的数据读了一遍，剩下的操作都是在 **sort_buffer** 和临时文件中执行的。但这个算法有一个问题，就是如果查询要返回的字段很多的话，那么 **sort_buffer** 里面要放的字段数太多，这样内存里能够同时放下的行数很少，要分成很多个临时文件，排序的性能会很差。

2. 所以如果单行很大，这个方法效率不够好。

3. 那么，**如果 MySQL 认为排序的单行长度太大会怎么做呢？**

4. 接下来，我来修改一个参数，让 **MySQL** 采用另外一种算法。

   ```mysql
   SET max_length_for_sort_data = 16;
   ```

5. **max_length_for_sort_data**，是 MySQL 中专门控制用于排序的行数据的长度的一个参数。它的意思是，如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法。

6. city、name、age 这三个字段的定义总长度是 36 ，我把 **max_length_for_sort_data** 设置为 16，我们再来看看计算过程有什么改变。

7. 新的算法放入 sort_buffer 的字段，只有要排序的列(即 name 字段)和主键 id 。

8. 但这时，排序的结果就因为少了 city 和 age 字段的值，不能直接返回了，整个执行流程就变成如下所示的样子：

   - 初始化 sort_buffer ，确定放入两个字段，即 name 和 id ；
   - 从索引 city 找到第一个满足 city='杭州' 条件的主键 id，也就是图中的 ID_X；
   - 到主键 id 索引取出整行，取 name、id 这两个字段，存入 sort_buffer 中；
   - 从索引 city 取下一个记录的主键 id ；
   - 重复步骤 3、4 直到不满足 city='杭州' 条件为止，也就是图中的 ID_Y；
   - 对 sort_buffer 中的数据按照字段 name 进行排序；
   - 遍历排序结果，取前 1000 行，并按照 id 的值回到原表中取出 city、name 和 age 三个字段返回给客户端。

9. 这个执行流程的示意图如下，我把它称为 **rowid** 排序。

   <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/rowid%20%E6%8E%92%E5%BA%8F.png" alt="rowid 排序" style="zoom:50%;" />

10. 对比全字段排序流程图你会发现，rowid 排序多访问了一次表t的主键索引，就是步骤 7 。

11. 需要说明的是，最后的结果集是一个逻辑概念，实际上 MySQL 服务端从排序后的 sort_buffer 中依次取出 id ，然后到原表查到 city、name 和 age 这三个字段的结果，不需要在服务端再耗费内存存储结果，是直接返回给客户端的。

12. 根据这个说明过程和图示，你可以想一下，这个时候执行 select @b-@a，结果会是多少呢？

13. 现在，我们就来看看结果有什么不同。

14. 首先，**examined_rows** 的值还是 4000 ，表示用于排序的数据是 4000 行。但是 select @b-@a 这个语句的值变成 5000 了。

15. 因为这时候除了排序过程外，在排序完成后，还要根据 id 去原表取值。由于语句是 limit 1000 ，因此会多读 1000 行。

    ```json
    {
      "filesort_summary": {
        "rows": 4000,
        "examined_rows": 4000,
        "number_of_tmp_files": 10,
        "sort_buffer_size": 32768,
        "sort_mode": "<sort_key, rowid>"
      }
    }
    ```

16. 从 **OPTIMIZER_TRACE** 的结果中，你还能看到另外两个信息也变了。
    - **sort_mode** 变成了 `<sort_key, rowid>`，表示参与排序的只有 name 和 id 这两个字段。
    - **number_of_tmp_files** 变成 10 了，是因为这时候参与排序的行数虽然仍然是 4000 行，但是每一行都变小了，因此需要排序的总数据量就变小了，需要的临时文件也相应地变少了。

## 15.4 全字段排序 VS rowid 排序

1. 我们来分析一下，从这两个执行流程里，还能得出什么结论。

2. 如果 MySQL 实在是担心排序内存太小，会影响排序效率，才会采用 rowid 排序算法，这样排序过程中一次可以排序更多行，但是需要再回到原表去取数据。

3. 如果 MySQL 认为内存足够大，会优先选择全字段排序，把需要的字段都放到 **sort_buffer** 中，这样排序后就会直接从内存里面返回查询结果了，不用再回到原表去取数据。

4. 这也就体现了 **MySQL** 的一个设计思想：**如果内存够，就要多利用内存，尽量减少磁盘访问。**

5. 对于 **InnoDB** 表来说，**rowid** 排序会要求回表多造成磁盘读，因此不会被优先选择。

6. 这个结论看上去有点废话的感觉，但是你要记住它，下一篇文章我们就会用到。

7. 看到这里，你就了解了，**MySQL** 做排序是一个成本比较高的操作。那么你会问，是不是所有的 **order by** 都需要排序操作呢？如果不排序就能得到正确的结果，那对系统的消耗会小很多，语句的执行时间也会变得更短。

8. 其实，并不是所有的 **order by** 语句，都需要排序操作的。从上面分析的执行过程，我们可以看到，MySQL 之所以需要生成临时表，并且在临时表上做排序操作，**其原因是原来的数据都是无序的。**

9. 你可以设想下，如果能够保证从 **city** 这个索引上取出来的行，天然就是按照 **name** 递增排序的话，是不是就可以不用再排序了呢？确实是这样的。

10. 所以，我们可以在这个市民表上创建一个 **city** 和 **name** 的联合索引，对应的 **SQL** 语句是：

    ```mysql
    alter table t add index city_user(city, name);
    ```

11. 作为与 city 索引的对比，我们来看看这个索引的示意图。

    <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/city%20%E5%92%8C%20name%20%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95%E7%A4%BA%E6%84%8F%E5%9B%BE.png" alt="city 和 name 联合索引示意图" style="zoom:50%;" />

12. 在这个索引里面，我们依然可以用树搜索的方式定位到第一个满足 city='杭州' 的记录，并且额外确保了，接下来按顺序取下一条记录的遍历过程中，只要 city 的值是杭州，name 的值就一定是有序的。

13. 这样整个查询过程的流程就变成了：

    - 从索引 (city,name) 找到第一个满足 city='杭州' 条件的主键 id ；

    - 到主键 id 索引取出整行，取 name、city、age 三个字段的值，作为结果集的一部分直接返回；

    - 从索引 (city,name) 取下一个记录主键 id ；

    - 重复步骤 2、3，直到查到第 1000 条记录，或者是不满足 city='杭州' 条件时循环结束。

      <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%BC%95%E5%85%A5(city,name)%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95%E5%90%8E%EF%BC%8C%E6%9F%A5%E8%AF%A2%E8%AF%AD%E5%8F%A5%E7%9A%84%E6%89%A7%E8%A1%8C%E8%AE%A1%E5%88%92.png" alt="引入(city,name)联合索引后，查询语句的执行计划" style="zoom:50%;" />

14. 可以看到，这个查询过程不需要临时表，也不需要排序。接下来，我们用 **explain** 的结果来印证一下。

    ```mysql
    mysql> explain select city,name,age from t where city='杭州' order by name limit 1000  ;
    +----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-------+
    | id | select_type | table | partitions | type | possible_keys  | key       | key_len | ref   | rows | filtered | Extra |
    +----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-------+
    |  1 | SIMPLE      | t     | NULL       | ref  | city,city_user | city_user | 66      | const | 1000 |   100.00 | NULL  |
    +----+-------------+-------+------------+------+----------------+-----------+---------+-------+------+----------+-------+
    1 row in set, 1 warning (0.00 sec)
    ```

15. 从上可以看到，Extra 字段中没有 **Using filesort** 了，也就是不需要排序了。而且由于 (city,name) 这个联合索引本身有序，所以这个查询也不用把 4000 行全都读一遍，只要找到满足条件的前 1000 条记录就可以退出了。也就是说，在我们这个例子里，只需要扫描 1000 次。

16. 既然说到这里了，我们再往前讨论，**这个语句的执行流程有没有可能进一步简化呢？**不知道你还记不记得，我在第 5 篇文章《 深入浅出索引（下）》中，和你介绍的覆盖索引。

17. 这里我们可以再稍微复习一下。**覆盖索引是指，索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据。**

18. 按照覆盖索引的概念，我们可以再优化一下这个查询语句的执行流程。

19. 针对这个查询，我们可以创建一个 city、name 和 age 的联合索引，对应的 SQL 语句就是：

    ```mysql
    alter table t add index city_user_age(city, name, age);
    ```

20. 这时，对于 city 字段的值相同的行来说，还是按照 name 字段的值递增排序的，此时的查询语句也就不再需要排序了。这样整个查询语句的执行流程就变成了：

    - 从索引 (city, name, age) 找到第一个满足 city='杭州' 条件的记录，取出其中的 city、name 和 age 这三个字段的值，作为结果集的一部分直接返回；

    - 从索引 (city, name, age) 取下一个记录，同样取出这三个字段的值，作为结果集的一部分直接返回；

    - 重复执行步骤 2 ，直到查到第 1000 条记录，或者是不满足 city='杭州' 条件时循环结束。

      <img src="https://studentcwz-pic-bed.oss-cn-guangzhou.aliyuncs.com/img/%E5%BC%95%E5%85%A5(city,name,age)%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95%E5%90%8E%EF%BC%8C%E6%9F%A5%E8%AF%A2%E8%AF%AD%E5%8F%A5%E7%9A%84%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png" alt="引入(city,name,age)联合索引后，查询语句的执行流程" style="zoom:50%;" />

21. 然后，我们再来看看 **explain** 的结果。

    ```mysql
    mysql> explain select city,name,age from t where city='杭州' order by name limit 1000  ;
    +----+-------------+-------+------------+------+--------------------+---------------+---------+-------+------+----------+-------------+
    | id | select_type | table | partitions | type | possible_keys      | key           | key_len | ref   | rows | filtered | Extra       |
    +----+-------------+-------+------------+------+--------------------+---------------+---------+-------+------+----------+-------------+
    |  1 | SIMPLE      | t     | NULL       | ref  | city,city_user_age | city_user_age | 66      | const |    1 |   100.00 | Using index |
    +----+-------------+-------+------------+------+--------------------+---------------+---------+-------+------+----------+-------------+
    1 row in set, 1 warning (0.00 sec)
    ```

22. 可以看到，Extra 字段里面多了 **Using index**，表示的就是使用了覆盖索引，性能上会快很多。
23. 当然，这里并不是说要为了每个查询能用上覆盖索引，就要把语句中涉及的字段都建上联合索引，毕竟索引还是有维护代价的。这是一个需要权衡的决定。
