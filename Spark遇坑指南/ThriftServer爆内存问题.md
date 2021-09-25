# 问题的来源
近期发现有个Sql，会把thrift server搞得OOM。其实查出来的数据也并不多，但是thrift server就是会爆内存。

# Spark的driver和executors交互的机制
![avatar](https://i2.wp.com/blog.knoldus.com/wp-content/uploads/2019/12/Image-1-1.png?w=810&ssl=1)
driver需要去计算logical plan和physical plan以给worker node执行，所以需要把所有需要的文件扫描一遍，把File Status缓存一把。

# 问题所在
由于该Sql没有使用分区，而是直接使用where语句，所以会全标扫描，导致所有的FileStatus都缓存在了thrift server里，用jmap一看，都是这货占的内存。查看hadoop下这个表的存储情况，大为震撼，每个分区下面都有600多个文件，数据存了半年有3000多个分区。这么一来，自然就会导致内存爆了。

# 问题分析
其实我们的并发度设置得并不高，只有100，为什么最后存了那么多的小文件呢？看了下存这个表的代码，又是大为震撼，它分了6次存一张表的东西。

# 问题解决
双管齐下：
1. 使用代码合并这些小文件
2. 修改存表代码，union成一张表再存储，而不是分6次存储。