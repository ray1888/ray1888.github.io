---
title: Go语言切片技巧
date: 2019-09-03 10:07:38
tags: Go
---


<!-- ---
layout: post
title: 
author: Ray Chan(ray1888)
date: '2019-09-03 10:07:38 +0800'
category: go
summary: Go Slice 
thumbnail: go.png
--- -->



# 添加元素到切片中

##  添加元素到尾部
```
var a []int
//  添加元素(后面的元素都需要解包）
append(a, 0)
// 添加切片(后面的元素都需要解包）
append(a, []int{1,2,3}...)
```

## 添加元素到头部
```
var a []int
//  添加元素(后面的元素都需要解包）  
a = append([]int{1}, a...)
// 添加切片(后面的元素都需要解包）  
a = append([]int{1,2,3}, a...)  
```

## 添加指定元素到切片中
性能较低的方法（会产生中间过渡切片）  
```
var a []int
a = append(a[:i], append([]int{1,2}, a[i:]...)...)  // 在第i个元素插入切片
a = append(a[:i], append([]int{1}, a[i:]...)...)     //  在第i个元素插入元素
```

性能较高的方法
```
//  插入单个元素
a = append(a, 0)
copy(a[i+1:],  a[i:])
a[i] = x 

//  添加多个元素(一个切片）
//  b= []int{1,2,3}
a = append(a,b..)
copy(a[i+len(b):],  a[i:])
copy(a[i:], b) 
```


# 删除切片元素

## 删除结尾元素
```
a = []int{1,2,3}
// 删除单个元素
a = a[:len(a)-1]
// 删除多个元素
a = a[:len(a)-N]
```

## 删除开头元素
```
a = []int{1,2,3}
// 删除单个元素
a = a[1:]
// 删除多个元素
a = a[N:]
```
  
不移动指针的方法(把后面元素往前移动)， 用到了空指针  
```
a = []int{1,2,3}
// 删除单个元素
a = append(a[:0], a[1:]...)  
// 删除多个元素
a =  append(a[:0], a[N:]...)  

//  使用Copy来进行处理
a = []int{1,2,3}
a = a[:copy(a, a[1:])]
a = a[:copy(a, a[N:])]
```

## 删除中间元素
```
a = []int{1,2,3,4,5}
a = append(a[:i], a[i+1:]...)
a = append(a[:i], a[i+N:]...)

a = a[:i+copy(a[i:], a[i+1:])]
a = a[:i+copy(a[i:], a[i+N:])]
```

# 切片的实现原理及使用的注意事项  
切片本质上是对底层数组的一个数据的引用。但是它是一个动态的结构。它的底层结构是这样的  
```
type SliceHeader struct{
    Data uintptr
    Len   int  //  实际已经使用的容量
    Cap   int  //  最大的容量
}
```

<!-- ![goSlice](/assets/img/posts/GoSlice.png) -->
{% asset_img GoSlice.png goSlice %}


可以看出实际上这两个切片都是指向同一个数组的。但是如果当切片进行扩展后，会变成这样。  

<!-- ![goSliceE](/assets/img/posts/GoSliceExtend.png) -->
{% asset_img GoSliceExtend.png goSliceE %}

  
可以看出，因为原来的底层数组因为长度已经不足以Slice进行扩展，因此Slice会先去创建一个新的底层数组，并且把原来的元素加入到新创建的数组中，并且再把新插入的元素插入到新数组中。然后把原来指向的底层数组的Reference Count 减一。  

## 使用的技巧
1. 因为如果append进去切片的时候，len = cap 就会出现内存的申请和数据的复制，这样会使得比较慢。使用时尽量去减少触发内存分配的次数和分配内存的大小。
2. 对于切片来说，即使是空切片，cap=0, len=0, 它实际上也不会是nil, 因此判断一个切片是否为空，应该使用len($slice_name) == 0 来继续判断。

## 切片GC的问题
一个比较经常的情况，对底层数组的某一个内存的引用，导致整个数组无法被GC。

### 变量引用的情况
```
func FindPhoneNumber(filename string) []byte{
	b, _ := ioutil.ReadFile(filename)
	return regexp.MustCompile("[0-9]+").Find(b)
}
```

上面的这段代码里面，因为返回的地方是一个切片引用了b的底层数组，导致b的底层数组会一直在内存中。  

解决上面的问题，可以使用传值把需要用到的值传到一个新的切片里面，这样就能减少依赖。
```
func FindPhoneNumber(filename string) []byte{
	b, _ := ioutil.ReadFile(filename)
	b =  regexp.MustCompile("[0-9]+").Find(b)
    return append([]byte{}, b)
}
```

### 删除变量的情况

```
var a *[]int{1,2,3,4,4,5,6}
a = a[:len(a)-1] // 被删除的最后一个元素仍然被引用
```

保险的处理方法, 把需要删除的元素设置为nil，然后再去进行切片
```
var a *[]int{1,2,3,4,4,5,6}
a[len(a)-1] = nil
a = a[:len(a)-1] // 被删除的最后一个元素仍然被引用
```