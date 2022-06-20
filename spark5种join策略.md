


# 1.分发方式
## 为什么要分发？
因为是分布式的，待拼接的（join key相同）左右两部分（来自不同表、不同物理机器）数据传输到一台机器上
## 分布式分发方式有哪些
- shuffle，左右两表都根据hash key进行map到reducer之间的数据传输
- 广播，只动小表的数据，把小表数据做成HashRelation，传输到大表所在的机器上

# 2.汇集到同一台机器之后的拼接方式
数据汇集到一台机器上之后，共有3种拼接实现方式。按照出现的时间顺序，分别是
- 1.嵌套循环连接（NLJ，Nested Loop Join ）
- 2.排序归并连接（SMJ，Shuffle Sort Merge Join）
- 3.哈希连接（HJ，Hash Join）
假设，现在有事实表orders和维度表users。其中，users表存储用户属性信息，orders记录着用户的每一笔交易。两张表的Schema如下：
```sql
// 订单表orders关键字段
userId, Int
itemId, Int
price, Float
quantity, Int
 
// 用户表users关键字段
id, Int
name, String
type, String //枚举值，分为头部用户和长尾用户
```
我们的任务是要基于这两张表做内关联（Inner Join），同时把用户名、单价、交易额等字段投影出来。具体的SQL查询语句如下表：
```sql
//SQL查询语句
select orders.quantity, orders.price, orders.userId, users.id, users.name
from orders inner join users on orders.userId = users.id
```
## NLJ的工作原理
对于参与关联的两张数据表，我们通常会根据它们扮演的角色来做区分。其中，__体量较大、主动扫描数据的表，我们把它称作外表或是驱动表；体量较小、被动参与数据扫描的表，我们管它叫做内表或是基表。__那么，NLJ是如何关联这两张数据表的呢？

NLJ是采用“嵌套循环”的方式来实现关联的。也就是说，NLJ会使用内、外两个嵌套的for循环依次扫描外表和内表中的数据记录，判断关联条件是否满足，比如例子中的orders.userId = users.id，如果满足就把两边的记录拼接在一起，然后对外输出。
![在这里插入图片描述](https://img-blog.csdnimg.cn/d27cbe3a608f441eb819beb71eb71b9b.png)
在这个过程中，外层的for循环负责遍历外表中的每一条数据，如图中的步骤1所示。而对于外表中的每一条数据记录，内层的for循环会逐条扫描内表的所有记录，依次判断记录的Join Key是否满足关联条件，如步骤2所示。假设，外表有M行数据，内表有N行数据，那么NLJ算法的计算复杂度是`O(M * N)`。不得不说，尽管NLJ实现方式简单而又直接，但它的执行效率实在让人不敢恭维。

## SMJ的工作原理
正是因为NLJ极低的执行效率，所以在它推出之后没多久之后，就有人用排序、归并的算法代替NLJ实现了数据关联，这种算法就是SMJ。SMJ的思路是先排序、再归并。具体来说，就是参与Join的两张表先分别按照Join Key做升序排序。然后，SMJ会使用两个独立的游标对排好序的两张表完成归并关联。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4b5a5837548a44a7ab9673ddd8572494.png)
SMJ刚开始工作的时候，内外表的游标都会先锚定在两张表的第一条记录上，然后再对比游标所在记录的Join Key。对比结果以及后续操作主要分为3种情况：

- 外表Join Key等于内表Join Key，满足关联条件，把两边的数据记录拼接并输出，然后把外表的游标滑动到下一条记录
- 外表Join Key小于内表Join Key，不满足关联条件，把外表的游标滑动到下一条记录
- 外表Join Key大于内表Join Key，不满足关联条件，把内表的游标滑动到下一条记录
- 
SMJ正是基于这3种情况，不停地向下滑动游标，__直到某张表的游标滑到头，即宣告关联结束__。对于SMJ中外表的每一条记录，由于内表按Join Key升序排序，且扫描的起始位置为游标所在位置，因此`SMJ算法的计算复杂度为O(M + N)`。

不过，SMJ计算复杂度的降低，仰仗的是两张表已经事先排好序。要知道，排序本身就是一项非常耗时的操作，更何况，为了完成归并关联，参与Join的两张表都需要排序。因此，__SMJ的计算过程我们可以用“先苦后甜”来形容。苦的是要先花费时间给两张表做排序，甜的是有序表的归并关联能够享受到线性的计算复杂度。__

## HJ的工作原理
考虑到SMJ对排序的要求比较苛刻，所以后来又有人提出了效率更高的关联算法：HJ。HJ的设计初衷非常明确：把内表扫描的计算复杂度降低至O(1)。把一个数据集合的访问效率提升至O(1)，也只有Hash Map能做到了。也正因为Join的关联过程引入了Hash计算，所以它叫HJ。
![在这里插入图片描述](https://img-blog.csdnimg.cn/be95cd64c53246038c8ccc9d0f8d3dd2.png)
HJ的计算分为两个阶段，分别是Build阶段和Probe阶段。在Build阶段，基于内表，算法使用既定的哈希函数构建哈希表，如上图的步骤1所示。哈希表中的Key是Join Key应用（Apply）哈希函数之后的哈希值，表中的Value同时包含了原始的Join Key和Payload。

在Probe阶段，算法遍历每一条数据记录，先是使用同样的哈希函数，以动态的方式（On The Fly）计算Join Key的哈希值。然后，用计算得到的哈希值去查询刚刚在Build阶段创建好的哈希表。如果查询失败，说明该条记录与维度表中的数据不存在关联关系；如果查询成功，则继续对比两边的Join Key。如果Join Key一致，就把两边的记录进行拼接并输出，从而完成数据关联。

# 分发与拼接方式的组合与性能
![在这里插入图片描述](https://img-blog.csdnimg.cn/5df614549e2e456bb4ec4a0cd4a6d0a4.png)
这5种Join策略，对应图中5个圆角矩形，从上到下颜色依次变浅，它们分别是Cartesian Product Join、Shuffle Sort Merge Join和Shuffle Hash Join。也就是采用Shuffle机制实现的NLJ、SMJ和HJ，以及Broadcast Nested Loop Join、Broadcast Sort Merge Join和Broadcast Hash Join。

从执行性能来说，5种策略从上到下由弱变强。相比之下，CPJ的执行效率是所有实现方式当中最差的，网络开销、计算开销都很大，因而在图中的颜色也是最深的。BHJ是最好的分布式数据关联机制，网络开销和计算开销都是最小的，因而颜色也最浅。此外，你可能也注意到了，并没有Broadcast Sort Merge Join，这是因为Spark并没有选择支持Broadcast + Sort Merge Join这种组合方式。

## 两个问题
### 1.排列组合得到的明明应该是6种组合策略，为什么Spark偏偏没有支持这一种呢？
>要回答这个问题，我们就要回过头来对比SMJ与HJ实现方式的差异与优劣势。
相比SMJ，HJ并不要求参与Join的两张表有序，也不需要维护两个游标来判断当前的记录位置，只要基表在Build阶段构建的哈希表可以放进内存，HJ算法就可以在Probe阶段遍历外表，依次与哈希表进行关联。
当数据能以广播的形式在网络中进行分发时，说明被分发的数据，也就是基表的数据足够小，完全可以放到内存中去。这个时候，相比NLJ、SMJ，HJ的执行效率是最高的。因此，在可以采用HJ的情况下，Spark自然就没有必要再去用SMJ这种前置开销比较大的方式去完成数据关联。
### 2.广播方式下为什么要有BNLJ，而不是只有BHJ?
>因为BNLJ支持不等值连接，而BHJ只支持等值连接
