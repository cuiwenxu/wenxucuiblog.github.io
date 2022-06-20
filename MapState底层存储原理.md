MapState是一个接口，使用idea来看一下其实现类的关系图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201111151946939.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE2MjQxNTc=,size_16,color_FFFFFF,t_70#pic_center)
MapState的实现类有如下三个：

- ImmutableMapState：只读的mapstate，可以调用其get、contain 等方法，调用put、remove方法时会报错
- UserFacingMapState：mapstate的包装类，可以妥当的将empty state展示为一个empty map。当用户调用getMapState时会返回UserFacingMapState实例，比如调用UserFacingMapState的keys方法，如果为空则调用emptyState.keySet()，其中emptyState是一个Java中的Map实例
```java
@Override
	public Iterable<Map.Entry<K, V>> entries() throws Exception {
		Iterable<Map.Entry<K, V>> original = originalState.entries();
		return original != null ? original : emptyState.entrySet();
	}

	@Override
	public Iterable<K> keys() throws Exception {
		Iterable<K> original = originalState.keys();
		return original != null ? original : emptyState.keySet();
	}

	@Override
	public Iterable<V> values() throws Exception {
		Iterable<V> original = originalState.values();
		return original != null ? original : emptyState.values();
	}

	@Override
	public Iterator<Map.Entry<K, V>> iterator() throws Exception {
		Iterator<Map.Entry<K, V>> original = originalState.iterator();
		return original != null ? original : emptyState.entrySet().iterator();
	}
```
- InternalMapState：真正包含存储逻辑的类，InternalMapState有三个实现类，**当backend为 MemoryStateBackend、FsStateBackend 两种时， StateBackend 在任务运行期间都会将 State 存储在内存中，此时用到的实现类是HeapMapState。当backend为RocksDBStateBackend 时，用到的实现类为RocksDBMapState。**

接下来详细讨论HeapMapState的存储，HeapMapState是把UK和UV存储到StateTable中，StateTable有两个子类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201111202311684.png#pic_center)

具体使用的是CopyOnWriteStateTable，CopyOnWriteStateTable底层使用的经典数据结构为CopyOnWriteStateMap，其具体实现原理可参照
源码解析 | 万字长文详解 Flink 中的 CopyOnWriteStateTable
https://mp.weixin.qq.com/s?__biz=MzU3Mzg4OTMyNQ==&mid=2247488598&idx=1&sn=5c85804f16ec9c3b7a210ed4c9c69fb5&chksm=fd3b9a14ca4c1302253dabd24c9ab316366bf092e22f76547600761b3e0ed8f412ecd663a271&scene=21#wechat_redirect
