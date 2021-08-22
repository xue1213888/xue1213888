# map底层结构和扩容规则


# map实现原理

这是map底层的数据结构

```golang
type hmap struct {
    count     int    // map中存入的键值对个数
    flags     uint8  // 标识当前map的状态，比如hashWriting=4第4位表示有goroutine正在往map写入
    B         uint8  // buckets的数量，2的B次方
    noverflow uint16 // 溢出桶的使用数量
    hash0     uint32 // hash算法的seed

    buckets    unsafe.Pointer // buckets，指向存储健值对的桶的位置
    oldbuckets unsafe.Pointer // 当需要扩容迁移时，这个oldBuckets指向旧桶的位置，buckets指向新桶的位置
    nevacuate  uintptr        // 当前的迁移进度

    extra *mapextra // 关于溢出桶的一些信息
}

type mapextra struct {

    overflow    *[]*bmap
	oldoverflow *[]*bmap
  
    nextOverflow *bmap // 指向下一个空闲的溢出桶位置
}

type bmap struct {
    tophash [bucketCnt]uint8 // bucketCnt=8，这边记录每一个tophash
}
```

# 键值对存储与hash表。

hash表的存储会有一个buckets数组，用来记录桶的位置，我们对key进行hash算法得到一个khash。

这个khash的低8位用来确定bucket桶的序号。

一般有两种方法：

- 直接与桶的数量取模。khash % m得到序号
- 与桶数量-1进行与运算。khash & （m-1）。

golang采用的是第二种方法，这要要求，桶的数量必须是2的次幂。它的二进制标记位刚好只有一个1，m-1的话刚好就是除了最高位，全是1。

我们现在得到bucket桶的位置了。

# 关于hash冲突

我们得到桶的位置后，不巧的是得到的桶已经存满了，放不下我们了，golang默认一个桶存8个键值对，bucketCtn=8。

这时候出现了hash冲突。

这时候我们还有两种办法应对：

- `开放地址法`：假如我们现在要使用的是二号桶，结果二号桶存满了，那我们就去三号桶找空位。
- `拉链法`：我们在二号桶后面拉一条指向溢出桶的线，在溢出桶内进行读写。

golang采用的是``拉链法``。

hash冲突会影响map的读写效率，一个散列均匀的hash算法是很重要的，适时扩容也是保障读写效率的有效手段。

# map扩容规则

golang在分配创建map时，会根据使用桶的数量创建相应数量的溢出桶，当B>=5，golang会认为map使用溢出桶的概率会很高，就会分配2^(b-4)个溢出桶，溢出桶和bucket在内存中是连续的，作用不同而已。

## 翻倍扩容

golang map的负载因子是6.5。即count/(2^b)>6.5就要进行扩容。扩容后的新桶是旧桶的两倍，这时候buckets指向新桶的位置，oldbuckets指向旧桶的位置，nevacuate表示当前需要迁移的旧桶序号。下一次读写操作时发生在该桶时，会首先进行迁移操作再进行读写。直到旧桶的所有数据都迁移到新桶中。

## 等量扩容

当不满足负载因子，但是的溢出桶使用过多时，会发生等量扩容。他的目的在于减少溢出桶的使用，提高读写效率。

当B小于15，并且使用溢出桶的数量=当前桶的数量了，或者B大于15，溢出桶的数量大于2的15次方就需要等量扩容。

# 当我们插入一个元素时的内部流程

- 修改flags，说明现在在写
- 我们首先对key进行hash算法得到一串hash值
- 用hash值的低位来确定要存入的桶的位置也就是相应的bucket内存位置
- 判断是否在扩容，如果在扩容，先迁移再处理
- 遍历bucket的bmap，
  - 找到空位置，就把tophash写到bmap.tophash数组的第i位，这个就是可以插入键值对的地址
  - 如果在遍历的过程中找到了相同的tophash，那么就判断key是否相等，相等的话，返回value的地址
  - 如果key不相等，那就继续遍历，并且bmap的tophash存满了，就从链在这个bmap后面的溢出桶查找和插入
  - 如果存满了，也没有链接在后面的溢出桶，就申请一个溢出桶链在后面。
- 如果当前不在扩容 && 负载因子达到要求 或者溢出桶数量=新桶数量 或者溢出桶数量>2的15次方。就进行扩容。




