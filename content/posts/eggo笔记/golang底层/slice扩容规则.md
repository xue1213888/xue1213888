---
title: "slice内存布局与扩容规则"
date: 2021-08-22T11:52:05+08:00
tags: ["eggo","golang","底层"]
categories: ["eggo"]
draft: false
---

# 首先我们来看一下Slice的结构11

slice由三部分组成

- `array`-指向底层数组的起始地址
- `len`-记录slice元素现有长度
- `cap`-记录slice的内存长度

我们通过make初始化的一个slice数组，它的返回的就是这样一个type。不是*type

```golang
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

这时候会分配底层数组的内存空间，并将array指向底层数组的起始位置。

假如我们通过new来进行初始化，它返回的就是*type，这时候是没有分配底层数组的，它的array=nil。len和cap都为0。

我们可以通过append为这个slice开辟底层数组的内存空间。

# 我们可以将多个slice指向同一个底层数组

```golang
func main() {
	sliArray := [10]int{1,2,3,4,5,6,7,8,9,10}
	sli1 := sliArray[2:5]
	sli2 := sliArray[:]
	sli3 := sliArray[5:]

	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli1",sli1,len(sli1),cap(sli1))
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli2",sli2,len(sli2),cap(sli2))
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli3",sli3,len(sli3),cap(sli3))

}
// output
// sli1=[3 4 5],len=3,cap=8
// sli2=[1 2 3 4 5 6 7 8 9 10],len=10,cap=10
// sli3=[6 7 8 9 10],len=5,cap=5


```

上面sli1，sli2，sli3的array指向同一个数组的不同位置

sli1从[2:5]代表从index=2的元素开始，到index=5的元素结束，不包含index=5，所以是3，4，5。len=3，cap代表的是底层数组的内存长度，所以cap是从index=2到该数组结束，所以cap=8

sli2，sli3同理

这时候对sli1进行append操作。

```golang

	sli1 = append(sli1,1)
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli1",sli1,len(sli1),cap(sli1))
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli2",sli2,len(sli2),cap(sli2))
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli3",sli3,len(sli3),cap(sli3))


// output
// sli1=[3 4 5 1],len=4,cap=8
// sli2=[1 2 3 4 5 1 7 8 9 10],len=10,cap=10
// sli3=[1 7 8 9 10],len=5,cap=5


```

append会对底层数组进行修改，因为添加元素后它还在cap的范围内。

这时候sli2，sli3相应位置的元素也发生了变化。

如果我们append添加元素后，超过了cap范围，他就会重新分配底层数组内存空间，并将原数据拷贝到新的位置。

```golang
	sli1 = append(sli1,1,2,3,4,5)
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli1",sli1,len(sli1),cap(sli1))
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli2",sli2,len(sli2),cap(sli2))
	fmt.Printf("%s=%d,len=%d,cap=%d\n","sli3",sli3,len(sli3),cap(sli3))

// output
// sli1=[3 4 5 1 1 2 3 4 5],len=9,cap=16
// sli2=[1 2 3 4 5 1 7 8 9 10],len=10,cap=10
// sli3=[1 7 8 9 10],len=5,cap=5

```

这时候不会影响sli2，和sli3的内容。

我们可以通过reflect包里面的SliceHeader来获取array指向的底层数组的地址，来验证我们上面的结论。发现确实在append超出cap范围后，指向底层数组的地址发生了变化。

```golang
func pointer(sli []int)int{
	hdr := (*reflect.SliceHeader)(unsafe.Pointer(&sli))
	return int(hdr.Data)
}

func main() {
	sliArray := [10]int{1,2,3,4,5,6,7,8,9,10}
	sli1 := sliArray[2:5]
	fmt.Println("start:",pointer(sli1))
	sli1 = append(sli1,1)
	fmt.Println("append-1:",pointer(sli1))
	sli1 = append(sli1,1,2,2,3,4,5,6,5)
	fmt.Println("append-2:",pointer(sli1))
}

// output
// start: 824634322576
// append-1: 824634322576
// append-2: 824634482688

```

# 这下到了我们的slice扩容规则

slice扩容规则分一下几个步骤

- 当append后长度小于oldCap，那么不进行扩容
- 当append后的长度大于oldCap*2，那么久扩容到那么多就好
- 当append后长度大于oldCap，并且小于2倍，那么新的newCap需要进行一下几个判断

1. len > oldCap && oldCap < 1024 ， newCap = oldCap * 2 ，这是预估的cap
2. len > oldCap && oldCap > 1024 ， newCap = oldCap * 1.25 ，这是预估的cap
3. 我们的到预估的newCap后，需要根据单个数据元素内存大小oneMem来判断需要多少内存needMem
4. 因为内存管理模块里面管理的内存是有一定规格的，内存管理模块分配给我们我们最接近我们需要的最小内存relMem
5. 然后用分配到的内存，除以我们的单个数据元素内存大小，得到最后的cap=relMem/oneMem

```golang
func main(){
	s1 := []int{1,2}
	s1 = append(s1,1,2,3)
	fmt.Println(cap(s1))
}
```

看上面例子

1. 一开始len=2，cap=2
2. append后，len=5，newCap=5
3. 64位操作系用1个int占8个字节，所以需要分配5*8=40字节的内存
4. 系统规格最接近的为48字节的内存，48/8=6。所以实际扩容后的cap=6

# Over

