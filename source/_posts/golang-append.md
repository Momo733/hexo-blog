---
title: golang内置append实现细节
date: 2019-06-26 11:32:13
tags:
 - append 
 - golang
---

在使用内置函数append()过程中，发现对于他的理解还不是很透彻所以写下这篇文章，直接先看问题，然后再继续深入问题到为什么会是这样的结果。

```
    var d = make([]int, 3)
	d = []int{1, 2,3}	
    println(len(d), cap(d), fmt.Sprintf("%p", d), fmt.Sprint(d)) //3 3 0xc0000541c0 [1 2 3]
	d = append(d[:2], 4)
	println(len(d), cap(d), fmt.Sprintf("%p", d), fmt.Sprint(d)) //3 3 0xc0000541c0 [1 2 4]	
```
<center> 代码块 - 1</center >

在第4行中的结果显示切片d的内容由``[1 2 3]``变成了``[1 2 4]	``和原来的切片相比，容量地址长度都没有任何变化，可见append()直接把切片中第三个值3给赋值成4，和我们平常所理解的append()操作的结果并不一样，那就先分析一波源码。

## 1.silce的结构
切片的源码在``src/suntime/silce.go``，其中定义了切片的结构。
```
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
根据上面的代码，可以知道切片其实就是一个已经分配数组内存地址，加上切片的容量，以及已经使用了的长度的数据结构体，而且cap永远大于等于len。

那么下面的两种slice是有区别的吗？

```
	var slice1 = []int{}
	var slice2 []int
	fmt.Print(slice2 == nil, slice1 == nil)
```
看过上面slice的结构就知道，slice1其实就是一个指向数组的指针不为nil，但是cap和len都是0。而slice2就是定义一个变量类型为切片而已，并没有初始化，所以他的指针是nil，结果自然是不相同的。

那么回到上面的问题``d[:2]``的容量和长度分别是3和2，但是指向的底层的数组指针还是原来的切片申请的数组地址。我们都知道数组的内存空间往往都是连续的一块内存，所以在上面的append()的时候其实是对原来切片的操作。此时append()判断容量是足够，添加一个元素4的时候，就不会选择去扩容，而是直接修改了原本底层数组的数据。

那么现在讨论下面的这个问题，你会觉得又有些迷糊了。
```
	var d = make([]int, 3)
	d = []int{1, 2,3}
	println(len(d), cap(d), fmt.Sprintf("%p", d), fmt.Sprint(d)) //3 3 0xc0000541c0 [1 2 3]
	
	_ = append(d[:2], 1,2)
	println(len(d), cap(d), fmt.Sprintf("%p", d), fmt.Sprint(d)) //3 3 0xc0000541c0 [1 2 3]
	
	d = append(d[:2], 1,2)
	println(len(d), cap(d), fmt.Sprintf("%p", d), fmt.Sprint(d)) //3 3 0xc00007a060 [1 2 1 2]
```
<center> 代码块 - 2</center >

在代码第5行如果把append()的赋值操作去掉切片d的结构没有任何变化，但是如果赋值到切片d这个时候d的地址和内容都发生了变化，这就和append()的内部扩容复制相关了。

## 2.append的扩容
append()的扩容代码也是位于``src/suntime/silce.go``中，源代码如下：
```
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	... //根据不同的情况计算容量的大小

	var p unsafe.Pointer
	if et.kind&kindNoPointers != 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```

可以看到在len小于1024的时候，扩容是按照doublecap的规则来双倍递增，大于等于1024的时候是按照原来的1/4来循环增加容量，直至符合数据所需要的容量要求。而且一旦扩容后可以看到返回的数据都是一个新的``slice{}``，指针p也指向一个新的数组地址，所以说扩容必定是会导致最后的切片地址改变。

那么再回到上个问题，我们可以看到在``代码块 - 2 ``中对于切片d有赋值的时候，d的地址完全改变所以发生了扩容以及赋值。在没有赋值操作的时候，append()其实也是扩容创建了一个新的切片，但是没有赋值，而且还是因为**append()是先计算切片需要保存数据的容量是否足够，然后计算了容量空间之后生成新的切片，再进行值的拷贝，而不是在一边拷贝，一边计算剩余的空间，否则就会修改切片d的数据了**。

## 3.总结
当切片是一个空切片的时候，其实他的指针是有地址的，只是没有任何的容量和数据而已，如果我们在循环中使用append的时候最好不要使用空切片去直接append，因为每次append扩容都会产生计算容量已经值拷贝，申请内存空间，造成内存碎片化，所以最好是在make的时候指定一个cap。

------

Reference：[深入解析 Go 中 Slice 底层实现](https://halfrost.com/go_slice/)