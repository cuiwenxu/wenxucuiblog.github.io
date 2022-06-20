##### 设计思路
我们以存放 Integer 值的 Bitmap 来举例（存放Long类型的存储方式会有所不同），RBM 把一个 32 位的 Integer 划分为高16 位和低16 位，通过高16位找到该数据存储在哪个桶中（高 16 位可以划分 2^16 个桶），把剩余的低 16 位放入该桶对应的 Container 中。

每个桶都有对应的 Container，不同的 Container 存储方式不同。依据不同的场景，主要有 3 种不同的 Container，分别是 ArrayContainer 、 BitmapContainer 、RunContainer。Array Container 存放稀疏的数据，Bitmap Container 存放稠密的数据。若一个 Container 里面的元素数量小于 4096，使用 Array Container 来存储。当 Array Container 超过最大容量 4096 时，会转换为 Bitmap Container。

![container继承关系](https://img-blog.csdnimg.cn/20200906160138953.png#pic_center)
##### Array Container
Array Container 是 Roaring Bitmap 初始化默认的 Container。Array Container 适合存放稀疏的数据，其内部数据结构是一个有序的 Short 数组。数组初始容量为 4，数组最大容量为 4096，所以 Array Container 是动态变化的，当容量不够时，需要扩容，并且当超过最大容量 4096 时，就会转换为 Bitmap Container。由于数组是有序的，存储和查询时都可以通过二分查找快速定位其在数组中的位置。后面会讲解为什么超过最大容量 4096 时变更 Container 类型。
下面我们具体看一下数据如何被存储的，例如，0x00020032（十进制131122）放入一个 RBM 的过程如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906160342259.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
0x00020032 的前 16 位是 0002，找到对应的桶 0x0002。在桶对应的 Container 中存储低 16 位，因为 Container 元素个数不足 4096，因此是一个 Array Container。低 16 位为 0032（十进制为50）, 在 Array Container 中二分查找找到相应的位置插入即可（如上图50的位置）。

相较于原始的 Bitmap 需要占用 16K (131122/8/1024) 内存来存储这个数，而这种存储实际只占用了4byte（桶中占 2 byte，Container中占 2 byte，不考虑数组的初始容量）。

##### Bitmap Container
第二种 Container 是 Bitmap Container。它的数据结构是一个 Long 数组，数组容量==恒定为 1024（1024个long位65536位）==，和上文的 Array Container 不同，Array Container 是一个动态扩容的数组。__Bitmap Container 不用像 Array Container 那样需要二分查找定位位置，而是可以直接通过下标直接寻址。__

由于每个 Bitmap Container 需要处理低 16 位数据，也就是需要使用 Bitmap 来存储需要 8192 byte（2^16/8）, 而一个 Long 值占 8 个 byte，所以数组大小为 1024。因此一个 Bitmap Container 固定占用内存 8 KB。

下面我们具体看一下数据如何被存储的，例如，0xFFFF3ACB（十进制4294916811）放入一个 RBM 的过程如下图所示：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200906160824958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
0xFFFF3ACB 的前 16 位是 FFFF，找到对应的桶 0xFFFF。在桶对应的 Container 中存储低 16 位，因为 Container 中元素个数已经超过 4096，因此是一个 Bitmap Container。低 16 位为 3ACB（十进制为15051）, 因此在 Bitmap Container 中通过下标直接寻址找到相应的位置，将其置为 1 即可（如上图15051的位置）。
##### RunContainer
RunContainer采用RLE算法做压缩，适用于存储连续值，如果您具有值11,12,13,14,15，则将其存储为11,4，其中4表示11后面有4个连续值。
##### ArrayContainer转换为BitmapContainer的阈值
可以看到元素个数达到 4096 （4096*16=65536）之前，Array Container 占用的空间比 Bitmap Container 的少，当 Array Container 中元素到 4096 个时，正好等于 Bitmap Container 所占用的 8 KB。当元素个数超过了 4096 时，Array Container 所占用的空间还是继续线性增长，而 Bitmap Container 的内存空间并不会增长，始终还是占用 8 KB，与数据量无关。所以当 Array Container 超过最大容量 4096 会转换为 Bitmap Container。
##### 源码分析
获取int高16位和低16位
```java
protected static short highbits(int x) {
    return (short) (x >>> 16);
}

protected static short lowbits(int x) {
    return (short) (x & 0xFFFF);
}
```



参考：
http://smartsi.club/better-bitmap-performance-with-roaring-bitmaps.html

##### RoaringBitmap在spark中的应用
RoaringBitmap在MapStatus类中有应用，MapStatus是ShuffleMapTask返回给Scheduler的结果。 包括输出到Reducer的Block的大小。RoaringBitmap在MapStatus中用于记录emptyblock的序号。代码如下
```java
def apply(loc: BlockManagerId, uncompressedSizes: Array[Long]): HighlyCompressedMapStatus = {
    // We must keep track of which blocks are empty so that we don't report a zero-sized
    // block as being non-empty (or vice-versa) when using the average block size.
    var i = 0
    var numNonEmptyBlocks: Int = 0
    var totalSize: Long = 0
    // From a compression standpoint, it shouldn't matter whether we track empty or non-empty
    // blocks. From a performance standpoint, we benefit from tracking empty blocks because
    // we expect that there will be far fewer of them, so we will perform fewer bitmap insertions.
    val emptyBlocks = new RoaringBitmap()
    val totalNumBlocks = uncompressedSizes.length
    val threshold = Option(SparkEnv.get)
      .map(_.conf.get(config.SHUFFLE_ACCURATE_BLOCK_THRESHOLD))
      .getOrElse(config.SHUFFLE_ACCURATE_BLOCK_THRESHOLD.defaultValue.get)
    val hugeBlockSizesArray = ArrayBuffer[Tuple2[Int, Byte]]()
    while (i < totalNumBlocks) {
      val size = uncompressedSizes(i)
      if (size > 0) {
        numNonEmptyBlocks += 1
        // Huge blocks are not included in the calculation for average size, thus size for smaller
        // blocks is more accurate.
        if (size < threshold) {
          totalSize += size
        } else {
          hugeBlockSizesArray += Tuple2(i, MapStatus.compressSize(uncompressedSizes(i)))
        }
      } else {
        emptyBlocks.add(i)
      }
      i += 1
    }
    val avgSize = if (numNonEmptyBlocks > 0) {
      totalSize / numNonEmptyBlocks
    } else {
      0
    }
    emptyBlocks.trim()
    emptyBlocks.runOptimize()
    new HighlyCompressedMapStatus(loc, numNonEmptyBlocks, emptyBlocks, avgSize,
      hugeBlockSizesArray.toMap)
  }
```
