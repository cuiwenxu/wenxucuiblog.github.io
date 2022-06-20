# bitset详解

Java的BitSet使用一个Long（一共64位）的数组中的没一位（bit）是否为`1`来表示当前Index的数存在不。但是BitSet又是如何实现的呢？其实只需要理解其中的两个方法：

- set
- get

## set

先看源代码：

```java
public void set(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    int wordIndex = wordIndex(bitIndex);
    expandTo(wordIndex);

    words[wordIndex] |= (1L << bitIndex); // Restores invariants

    checkInvariants();
}
```

除了判断给的值是否小于0的那两句，我们依次来看一下这个函数的每一句代码。

### wordIndex

第一句就是计算wordIndex，通过`wordIndex`函数获取值。代码如下：

```cpp
private static int wordIndex(int bitIndex) {
    return bitIndex >> ADDRESS_BITS_PER_WORD;
}
```

这里`ADDRESS_BITS_PER_WORD`的值是6，那么最先想到的问题就是：**为什么是6呢？而不是其他值呢？**

答案其实很简单，还记得在最开始提到的：BitSet里使用一个Long数组里的每一位来存放当前Index是否有数存在。

因为在Java里Long类型是64位，所以一个Long可以存储64个数，而要计算给定的参数`bitIndex`应该放在数组（在BitSet里存在`word`的实例变量里）的哪个long里，只需要计算：`bitIndex / 64`即可，这里正是使用`>>`来代替除法（因为位运算要比除法效率高）。而64正好是2的6次幂，所以`ADDRESS_BITS_PER_WORD`的值是6。

通过`wordIndex`函数就能计算出参数`bitIndex`应该存放在`words`数组里的哪一个long里。

### expandTo

```java
private void expandTo(int wordIndex) {
    int wordsRequired = wordIndex+1;
    if (wordsInUse < wordsRequired) {
        ensureCapacity(wordsRequired);
        wordsInUse = wordsRequired;
    }
}
```

从上面已经知道在BitSet里是通过一个Long数组（`words`）来存放数据的，这里的`expandTo`方法就是用来判断`words`数组的长度是否大于当前所计算出来的`wordIndex`（简单的说，就是能不能存的下），如果超过当前`words`数组的长度（记录在实例变量`wordsInUse`里），也即是存不下，则新加一个long数到`words`里(ensureCapacity(wordsRequired)所实现的。)。

### Restores invariants

```java
words[wordIndex] |= (1L << bitIndex); // Restores invariants
```

这一行代码可以说是BitSet的精髓了，先不说什么意思，我们先看看下面代码的输出：

```java
System.out.println(1<<0);
System.out.println(1<<1);
System.out.println(1<<2);
System.out.println(1<<3);
System.out.println(1<<4);
System.out.println(1<<5);
System.out.println(1<<6);
System.out.println(1<<7);
```

输出是：

```undefined
1
2
4
8
16
32
64
128
```

这个输出看出规律没有？就是2的次幂，但是还是不太好理解，我们用下面的输出，效果会更好：

```css
System.out.println(Integer.toBinaryString(1<<0));
System.out.println(Integer.toBinaryString(1<<1));
System.out.println(Integer.toBinaryString(1<<2));
System.out.println(Integer.toBinaryString(1<<3));
System.out.println(Integer.toBinaryString(1<<4));
System.out.println(Integer.toBinaryString(1<<5));
System.out.println(Integer.toBinaryString(1<<6));
System.out.println(Integer.toBinaryString(1<<7));
```

输出是：

```undefined
1
10
100
1000
10000
100000
1000000
10000000
```

从而发现，上面所有的输出里，`1`所在的位置，正好是第1，2，3，4，5，6，7，8（Java数组的Index从0开始）位。而BitSet正是通过这种方式，将所给的`bitIndex`所对应的位设置成1，表示这个数已经存在了。这也解释了`(1L << bitIndex)`的意思（注意：因为BitSet是使用的Long，所以要使用1L来进行位移）。

搞懂了`(1L << bitIndex)`，剩下的就是用`|`来将当前算出来的和以前的值进行合并了`words[wordIndex] |= (1L << bitIndex);这里是或等的逻辑，展开为words[wordIndex] = words[wordIndex]|(1L << bitIndex)，和+=一个逻辑`。

剩下的	`checkInvariants`函数使用了断言，只有在`vm option`中配置了`-ea`时才能起作用。

```java
private void checkInvariants() {
        assert(wordsInUse == 0 || words[wordsInUse - 1] != 0);
        assert(wordsInUse >= 0 && wordsInUse <= words.length);
        assert(wordsInUse == words.length || words[wordsInUse] == 0);
    }

```

**<font color='green'>总体看一下`set`的流程，对于输入的input(必须为正整数)，首先input>>6，相当于除64，取商(input>>6)，根据商的值判断应该用long型数组(words[])中的哪个下标来存储，然后让words[input>>6]和1<<input按位或，就把input放到long型数组的第input位上了</font>**

**<font color='red'>需要注意的一点：Java中的移位操作是循环性的，long型变量在64位操作系统中是64位来存储的，如果将一个long型变量a<<65和a<<1效果是相同的</font>**

## get

搞懂了`set`方法，那么`get`方法也就好懂了，整体意思就是算出来所给定的`bitIndex`所对应的位数是否为`1`即可。先看看代码：

```java
public boolean get(int bitIndex) {
    if (bitIndex < 0)
        throw new IndexOutOfBoundsException("bitIndex < 0: " + bitIndex);

    checkInvariants();

    int wordIndex = wordIndex(bitIndex);
    return (wordIndex < wordsInUse)
        && ((words[wordIndex] & (1L << bitIndex)) != 0);
}
```

计算`wordIndex`在上面set方法里已经说明了，就不再细述。这个方法里，最重要的就只有：`words[wordIndex] & (1L << bitIndex)`。这里`(1L << bitIndex)`也已经做过说明，就是算出一个数，只有`bitIndex`位上为1，其他都为0，然后再和`words[wordIndex]`做`&`计算，如果`words[wordIndex]`数的`bitIndex`位是`0`，则结果就是`0`，以此来判断参数`bitIndex`存在不。

# 运用

搞清楚了原理，就可以很好的运用了，不过这里不讲怎么使用Java的BitSet，网上有很多。我讲一个我在刷`Leetcode`时碰见的一个题，就可以使用BitSet的原理来实现。

题目如下：

```csharp
Given an array containing n distinct numbers taken from 0, 1, 2, ..., n, find the one that is missing from the array.
For example, Given nums = [0, 1, 3] return 2.
Note: Your algorithm should run in linear runtime complexity. Could you implement it using only constant extra space complexity?
```

翻译过来就是：从0到n之间取出n个不同的数，找出漏掉的那个。 注意：你的算法应当具有线性的时间复杂度。你能实现只占用常数额外空间复杂度的算法吗？

这正好可以使用BitSet的思想来实现。

```cpp
public int missingNumberInByBitSet(int[] array) {
    int bitSet = 0;
    for (int element : array) {
        //首先用按位或操作，来实现数组对应下标的元素都置为1
        bitSet |= 1 << element;
    }

    for (int i = 0; i < array.length; i++) {
        //按位与操作，如果哪一位与完之后为0，就表示缺少这个数字
        if ((bitSet & 1 << i) == 0) {
            return i;
        }
    }

    return 0;
}
```

这里是简单实现，使用int，而不是lang，也没有用一个数组，但是核心思想是这样的。

#####运用二：去重顺带排序

`把需要去重的数据放到bitset中，然后将long型数组转化为二进制数组，输出为1的下标即可，缺点是只适用于正整数数组`

```java
public static void main(String[] args) {

        int[] arr={0,3,2,44,55,34,6,2,7,8,10};
        //在初始化bitset时，可以根据所存放的最大数字来指定大小，以避免重复的申请空间，否则默认是64位
        //BitSet bitSet= new BitSet(max+1);
        BitSet bitSet= new BitSet();
        for(int each:arr){
            bitSet.set(each);
        }
        for(long each:bitSet.toLongArray()){
            System.out.println(Long.toBinaryString(each));
            char[] chars=Long.toBinaryString(each).toCharArray();
            for(int i=0;i<chars.length;i++){
                if(chars[i]=='1'){
                    System.out.println(i);
                }
            }
        }

    }
```


