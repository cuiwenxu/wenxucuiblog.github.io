
虽然这种算法能大大地减少存储开销，但是随着数据量的增大，它依然面临着存储上的压力。在本篇文章中将要介绍的 HyperLogLog（下称 HLL）是一种非精确的去重算法，它的特点是具有非常优异的空间复杂度（几乎可以达到常数级别）。
![在这里插入图片描述](https://img-blog.csdnimg.cn/d994abcfa2f849ce8f981428dfe4b985.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
HLL 算法需要完整遍历所有元素一次，而非多次或采样；该算法只能计算集合中有多少个不重复的元素，不能给出每个元素的出现次数或是判断一个元素是否之前出现过；多个使用 HLL 统计出的基数值可以融合。
![在这里插入图片描述](https://img-blog.csdnimg.cn/2f2fcb0c97ee4a7181fcda9ac535814f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/483dd9cf546c408db1661e84a79053b7.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)


HLL 算法有着非常优异的空间复杂度，可以看到它的空间占用随着基数值的增长并没有变化。HLL 后面不同的数字代表着不同的精度，数字越大，精度越高，占用的空间也越大，可以认为 HLL 的空间占用只和精度成正相关。

# HLL算法原理感性认知
![在这里插入图片描述](https://img-blog.csdnimg.cn/df9e45d0975d438caa4a94374397b3db.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

HLL 算法的原理会涉及到比较多的数学知识，这边对这些数学原理和证明不会展开。举一个生活中的例子来帮助大家理解HLL算法的原理：比如你在进行一个实验，内容是不停地抛硬币，记录你连续抛到正面的次数（这是数学中的伯努利过程，感兴趣同学可以自行研究下）；如果你最多的连抛正面记录是3次，那可以想象你并没有做这个实验太多次，如果你最长的连抛正面记录是 20 次，那你可能进行了这个实验上千次。

一种理论上存在的情况是，你非常幸运，第一次进行这个实验就连抛了 20 次正面，我们也会认为你进行了很多次这个实验才得到了这个记录，这就会导致错误的预估；改进的方式是请 10 位同学进行这项实验，这样就可以观察到更多的样本数据，降低出现上述情况的概率。这就是 HLL 算法的核心思想。

# HLL算法具体实现
![在这里插入图片描述](https://img-blog.csdnimg.cn/0e53fe28f8114eecaf348169cfb2cac4.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)


HLL 会通过一个 hash 函数来求出集合中所有元素的 hash 值（二进制表示的 hash 值，就可以理解为一串抛硬币正反面结果的序列），得到一个 hash 值的集合，然后找出该 hash 值集合中，第一个 1 出现的最晚的位置。例如有集合为 [010, 100, 001], 集合中元素的第一个 1 出现的位置分别为 2, 1, 3，可以得到里面最大的值为 3，故该集合中第一个1出现的最晚的位置为 3。因为每个位置上出现1的概率都是 1/2，所以我们可以做一个简单的推断，该集合中有 8 个不重复的元素。

可以看到这种简单的推断计算出来集合的基数值是有较大的偏差的，那如何来减少偏差呢？正如我上面的例子里说的一样，HLL 通过多次的进行试验来减少误差。那它是如何进行多次的实验的呢？这里 HLL 使用了分桶的思想，上文中我们一直有提到一个精度的概念，比如说 HLL(10)，这个 10 代表的就是取该元素对应 Hash 值二进制的后 10 位，计算出记录对应的桶，桶中会记录一个数字，代表对应到该桶的 hash 值的第一个 1 出现的最晚的位置。如上图，该 hash 值的后 10 位的 hash 值是 0000001001，转成 10 进制是 9，对应第 9 号桶，而该 hash 值第一个 1 出现的位置是第 6 位，比原先 9 号桶中的数字大，故把 9 号桶中的数字更新为 6。可以看到桶的个数越多，HLL 算法的精度就越高，HLL(10) 有 1024(210) 个桶，HLL(16)有 65536(216) 个桶。同样的，桶的个数越多，所占用的空间也会越大。
![在这里插入图片描述](https://img-blog.csdnimg.cn/4badfda05f224874bcb49ca644927065.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)
刚才的例子我们省略了一些细节，为了让大家不至于迷失在细节中而忽视了重点，真实的 HLL 算法的完整描述见上图，这边的重点是计算桶中平均数时使用调和平均数。调和平均数的优点是可以过滤掉不健康的统计值，使用算术平均值容易受到极值的影响（想想你和马云的平均工资），而调和平均数的结果会倾向于集合中比较小的元素。HLL 论文中还有更多的细节和参数，这边就不一一细举，感兴趣的同学可以自己阅读下论文。

# HLL评估
![在这里插入图片描述](https://img-blog.csdnimg.cn/cf573a33c83544f5b4ad5daed0844a45.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70)

HLL 的误差分布服从正态分布，它的空间复杂度: O(m log2log2N), Ｎ 为基数, m 为桶个数。这边给大家推导一下它的空间复杂度，我有 264 个的不重复元素(Long. MAX_VALUE)，表达为二进制一个数是 64 位，这是第一重 log2, 那么第一个1最晚可能出现在第 64 位。64 需要 6 个 bit (26=64) 就可以存储，这是第二重 log2。如果精度为 10，则会有 1024 个桶，所以最外面还要乘以桶的个数。由于需要完整的遍历元素一遍，所以它的时间复杂度是一个线性的时间复杂度。      

# Q&A


>Q：为什么分桶越多，精度越高？

>A：可以举一个反例，假如说只有一个桶，那么该桶很容易受极值的影响，如果有多个桶，即使其中一个桶受极值影响，但是会有一个取调和平均数的过程，就把极值影响降到最低了，所以桶越多，精度越高




