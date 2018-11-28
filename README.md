# MTLeafIDTree
美团分布式ID生成系统研读


<pre>
业务系统对ID号的要求
      1）全局唯一性
         不能出现重复的ID号，既然是唯一标识，这一点非常重要
      2）趋势递增
         在主键的选择上，应该尽可能使用有序的主键保证写入性能
      3）单调递增
         保证下一个ID一定大于上一个ID，例如事务版本号，IM增量信息，排序等特殊需求
      5）信息安全
         如果ID是连续的，恶意用户的爬取工作就非常容易了，直接按照顺序下载指定URL。
         所以在一些场景下，会需要ID无规则，不规则
</pre>

<pre>
ID生成系统的要求
      1）平均延迟要尽可能低
      2）可用性5个9
      3）高QPS
</pre>

<pre>
UUID方案
      UUID的标准型式包含32个16进制数，以连字号分为5段。形式为8-4-4-4-12的32个字符
          如:
              550e8400-e29b-41d4-a716-446655440000
      优点：
          性能非常高，本地生成，没有网络消耗
      缺点：
          1）不易存储：UUID太长，16字节128位，通常以36自己长度的字符串标识。
          2）ID作为MySQL主键不合适，MYSQL官方明确建议主键越短越好，InnoDB的表尤其不合适
          3）对MySQL索引不利，作为数据库主键，在InnoDB引擎下，UUID的无序性可能会引起数
             据位置频繁变动，严重影响性能。
          5）基于Mac地址生成UUID的算法可能造成MAC地址泄漏
</pre>

![](https://i.imgur.com/RFFmimR.png)

<pre>
类snowflake方案
      是一种以划分命名空间来生成ID的一种算法，这方案把64-bit分别划成多段，分开来标识机器，
   时间戳等，比如：

      41-bit的时间可以表示（1L<<41）/(1000L*3600*24*365)=69年的时间，10-bit机器可以
      分别表示1024台机器。如果我们对IDC划分有需求，还可以将10-bit分5-bit给IDC，分5-bit
      给工作机器。这样就可以表示32个IDC，每个IDC下可以有32台机器，可以根据自身需求定义。
      12个自增序列号可以表示2^12个ID，理论上snowflake方案的QPS约为409.6w/s，这种分配
      方式可以保证在任何一个IDC的任何一台机器在任意毫秒内生成的ID都是不同的。

      优点：
          1）毫秒数在高位，自增序列在低位，整个ID都是趋势递增的
          2）不依赖数据库等第三方系统，以服务的方式部署，稳定性高，生成ID的性能也是非常高
          3）可以根据自身业务特性分配bit位，非常灵活
      缺点：
          1）强依赖机器时钟，如果机器上始终回拨，会导致发号重复或者服务会处于不可用状态。


      Mongodb的ObjectID可以算作是和snowflake类似方法
             通过   “时间  +  机器码  + pid + inc”共12个字节，通过 4 + 3 + 2 + 3 的
         方式标识成一个24长度的16进制数。
             5720194cf5cc3585bb7e5b32
</pre>

![](https://i.imgur.com/5uJNrZn.png)

![](https://i.imgur.com/IYNzqSD.png)

<pre>
数据库生成
      在MySQL表中，利用给定字段  AUTO_INCREMENT 和 AUTO_INCREMENT_OFFSET来保证ID自增

      每次业务使用下列SQL读写MYSQL得到ID
          REPLACE INTO Tickets64 (stub) VALUES ('a');
          SELECT LAST_INSERT_ID();

      优点：
          1）非常简单，利用现有数据库系统的功能实现，成本小，有DBA专业维护
          2）ID号单调递增，可以实现一些对ID有特殊要求的业务

      缺点：
          1）强依赖DB，当DB异常时整个系统不可用，属于致命问题，配置主从复制可以尽可能增加
             可用性，但是数据一致性在特殊情况下难以保证，主从切换时不一致可能会导致重发错误。
          2）ID发号性能瓶颈限制在单台MYSQL的读写性能。

      对MYSQL性能问题，可以使用如下方案解决：
          在分布式系统中可以多部署几台机器，每台机器设置不同的初始值，且步长和机器数相等，
          比如有两台机器。设置步长为2;
               TicketServer1的初始值为1(1,3,5,7,9,...);
               TicketServer2的初始值为2(2,4,6,8,10,...);

      这种架构貌似能够满足性能的需求，但是有以下几个缺点：
          1）系统水平扩展比较困难
             比如定义好了步长和机器台数之后，如果需要添加机器该怎么办？
             如果线上机器台数很多，维护起来会很麻烦。
          2）ID没有了单调递增的特性，只能趋势递增，这个缺点对于一般业务不是很重要，可以容忍。
          3）数据库压力还是很大，每次获取ID都得读写一次数据库，只能靠堆机器来提高性能。
</pre>

<pre>
Leaf方案
</pre>