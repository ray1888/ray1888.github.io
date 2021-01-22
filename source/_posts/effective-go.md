---
title: Effective Go Reading
date: 2019-09-16 11:07:38
tags: Go
---


<!-- --- -->
<!-- layout: post
title: Effective Go Reading
author: Ray Chan(ray1888)
date: '2019-09-16 11:07:38 +0800'
category: go
summary: Effective Go 
thumbnail: go.png
--- -->

本文章的目的是为了详细阅读和理解Effective GO所提及到的内容。

# 目录
1. [Method](#Method)  
   1.1 [Pointers vs Values](#PvV)  
2. [Data](#Data)
   2.1 [New vs Make](#NewMake)  
   2.2 [Array](#Array)  
   2.3 [Slice](#Slice)  
   2.4 [Map](#Map)  
   2.5 [Append](#Append)  
3. [Interface](#InterFace)
4. [Error](#Error)
5. [PackageInit](#Init)  
6. [Defer](#Defer)
7. [ShareNote](#ShareNote)

# <a id="Method"><span class="toptitle">Method</span></a>

## <a id="PvV"><span class="secondtitle">Pointers vs Values</span></a>

主要的区别，如果方法是放在类型值上面而不是指针上面的，可以通过指针和普通类型来进行使用。但是对于方法绑定的是指针的类型，只能通过指针来进行使用。

```
package main

import "fmt"

type ByteSlice []byte

func (slice ByteSlice) Append(data []byte) []byte {
	// Body exactly the same as the Append function defined above.
	l := len(slice)
	if l+len(data) > cap(slice) { // reallocate
		// Allocate double what's needed, for future growth.
		newSlice := make([]byte, (l+len(data))*2)
		// The copy function is predeclared and works for any slice type.
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[0 : l+len(data)]
	copy(slice[l:], data)
	return slice
}

func (p *ByteSlice) Append2(data []byte) {
	slice := *p
	// Body as above, without the return.
	l := len(slice)
	if l+len(data) > cap(slice) { // reallocate
		// Allocate double what's needed, for future growth.
		newSlice := make([]byte, (l+len(data))*2)
		// The copy function is predeclared and works for any slice type.
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[0 : l+len(data)]
	copy(slice[l:], data)
	*p = slice
}

func (p *ByteSlice) Write(data []byte) (n int, err error) {
	slice := *p
	// Body as above, without the return.
	l := len(slice)
	if l+len(data) > cap(slice) { // reallocate
		// Allocate double what's needed, for future growth.
		newSlice := make([]byte, (l+len(data))*2)
		// The copy function is predeclared and works for any slice type.
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[0 : l+len(data)]
	copy(slice[l:], data)
	*p = slice
	*p = slice
	return len(data), nil
}

func main() {
	var b ByteSlice
	b = b.Append([]byte{1, 2, 3})
	fmt.Printf("byteSlice 1 is %v", b)
	b.Write([]byte{7, 8, 9})
	fmt.Printf("byteSlice 2 is %v", b)
    /*
    func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
    */
	fmt.Fprintf(&b, "This hour has %d days\n", 7)
    /*
    if use below code , will throw error 
    fmt.Fprintf(b, "This hour has %d days\n", 7)
    */
	b.Append2([]byte{4, 5, 6})
	fmt.Printf("byteSlice 3 is %v", b)
}
```

上面的例子表达的是，假如直接像注释的代码那样，把B传入一个满足io.Writer的指针的方法中（含有Write方法），但是因为我们的自定义类型上面的Write是指针方法，而不是类型方法，所以会出现类型报错的问题。报错如下
```
as type io.Writer in argument to fmt.Fprintf:
	ByteSlice does not implement io.Writer (Write method has pointer receiver)
```

根据官方的描述，原文如下：
```
The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers.

This rule arises because pointer methods can modify the receiver; invoking them on a value would cause the method to receive a copy of the value, so any modifications would be discarded. The language therefore disallows this mistake. There is a handy exception, though. When the value is addressable, the language takes care of the common case of invoking a pointer method on a value by inserting the address operator automatically. In our example, the variable b is addressable, so we can call its Write method with just b.Write. The compiler will rewrite that to (&b).Write for us.
```

翻译一下：
对于接收者类型是指针还是值的规则，值接收者可以 被值或者指针进行调用。而指针方法只能被指针进行调用。

这个规则的产生的原因是因为指针方法可以修改接受者的值。 但是以值的方式继续调用的情况下，go是会自动把值复制一份，然后继续方法的调用，所有对于里面变量得修改都会被丢弃（因为是值传递，除非使用Return +调用的地方有返回值接收）。 因此语言不允许有这种的错误出现。当值是可以获得地址的情况下，语言会自动把值获取指针传入指针调用的方法里面。 


# <a id="Data"><span class="toptitle">Data</span></a>

## <a id="NewMake"><span class="secondtitle">New vs Make</span></a>

### <a id="New"><span class="thirdtitle">New</span></a>
New 是 Go里面的分配内存的方法，但是它只会创建一个Zero值（即创建一个0的空间给对应的内存，并且返回这个变量所占有的内存地址）。

由于由new出来的内存占用为0。这样对于设计你自己的数据结构很有帮助，原因是因为可以默认初始化了0的值，而不用之后再去进行二次的初始化。

官网上的例子：
```
For example, the documentation for bytes.Buffer states that "the zero value for Buffer is an empty buffer ready to use." Similarly, sync.Mutex does not have an explicit constructor or Init method. Instead, the zero value for a sync.Mutex is defined to be an unlocked mutex.

The zero-value-is-useful property works transitively. Consider this type declaration.

type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}

```

但是有些时候直接初始化0值不要足够，需要一个构建者。像这个例子一样
```
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := new(File)
    f.fd = fd
    f.name = name
    f.dirinfo = nil
    f.nepipe = 0
    return f
}
```

因为上面这段代码有比较多的参数，因此我们可以用一个命名的字段来继续初始化
```
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }
    f := File{fd, name, nil, 0}
    return &f
}
```

事实上，直接获取复合文字初始化的结构体的地址，实际上会创建一个全新的实例并且赋值，因此我们可以把上面的最后两行代码合成一行代码  
```
f := File{fd, name, nil, 0}
    return &f
=== 

return &File{fd, name, nil, 0}
```

用于和范围： 适用于创建数组、切片、映射（with the field labels being indices or map keys as appropriate）。

返回值：
一个对应类型的指针

### <a id="Make"><span class="thirdtitle">Make</span></a>

Make 可以用于创建并且返回一个非nil的值。  
适用范围：
切片、映射、channel  

```
make([]int,10,100)  // 这样是创建了一个容量为100，但是填入了10个0的切片。
```

官网上面对make的描述
```
For slices, maps, and channels, make initializes the internal data structure and prepares the value for use.
```

返回值：
一个对应类型的数据

### <a id="MakediffNew"><span class="thirdtitle">Diff</span></a>
区别上面的Make和New的目的是为了，对于切片、映射、channel这三种类型，在底层的实现原理中都必须要先创建一个底层的实现才能被引用到，才能在暴露给语言的使用者上面不会抛出错误。  
所以本质上，new创建出来的可以理解为一个nil的对象，而make创建出来的是一个带有底层数据结构，并且有非nil的对象。  
而且返回值有所不同。
```
package main

import "fmt"

func main(){
	 // allocates slice structure; *p == nil; rarely useful
	var p *[]int = new([]int)     
	// the slice v now refers to a new array of 100 ints 
	var v []int = make([]int, 5) 
	fmt.Printf("p values is %v，%v\n", p, *p==nil)
	fmt.Printf("v values is %v, %v\n", v, v==nil)
	
}

------------------
p values is &[]，true
v values is [0 0 0 0 0], false

```

## <a id="Array"><span class="secondtitle">Array</span></a>

数组在Go的三个特点：
1. 如果直接把数据传入到一个函数中，则是把这个数组的值拷贝一份，然后把拷贝的副本传到函数中进行使用。
2. 数组的长度也是它的类型属性之一，[10]int和[20]int不是等价的。
3. Array都是值。

而且有一个使用的小技巧，如果想要减少传递的数据的量，可以直接传入指针，这样可以免于拷贝多一份中间的数据。  
对于Go来说，因为数组类型支持的方法比较少，而且不能够通过动态去进行特定长度数组创建。因此更加建议的是用Slice来代替数组。  

## <a id="Slice"><span class="secondtitle">Slice</span></a>

可以看我的博客另外一篇的文章Go Slice上面有提及高级的用法，此处不再重复。  

### 二维数组和二维切片

对于二维数组的声明，可以使用这样的方法  
```
type Transform [3][3]float64  // A 3x3 array, really an array of arrays.
type LinesOfText [][]byte     // A slice of byte slices.

text := LinesOfText{
	[]byte("Now is the time"),
	[]byte("for all good gophers"),
	[]byte("to bring some fun to the party."),
}
```

如果使用Make来继续初始化的情况。需要考虑两个不同的使用场景导致的初始化方式的不同。  
第一种：如果内部的一位数组可能发生扩展或者收缩，那么就要单独去分配这个一位数组。
第二种： 如果内部的数组长度不会发生size变化而只会发生值得变化，则可以进行统一得初始化。(这样性能会比较好，因为可以一次性的调用Allocate的系统函数，减少分配内存的开销)
```
// First Method
picture := make([][]uint8, YSize)
for i:= range picture{
    picture[i] = make([]uint8, XSize)
}

// Second Method 
picture := make([][]uint8, YSize)
pixels := make([]uint8, XSize * YSize)

for i := range picture{
    picture[i], pixels = pixels[:XSize], pixels[XSize:]
}

```

## <a id="Map"><span class="secondtitle">Map</span></a>
主要注意当一个Key不存在与一个Map中的情况下，需要使用这种方法来进行判断
```
var tz map[string]int
var ds string = "abc"
if val, ok := tz[ds]; ok{
    return val
}
```

同理在上面的例子上面，如果反过来进行使用，可以用于判断Key是否存在于Map中。
```
var tz map[string]int
var ds string = "abc"
_, exist := tz[ds]
if !exist {
    return "is not exist"
}
```

如果需要删除一个值的情况下,使用delete的函数进行处理
```
var tz map[string]int
var ds string = "abc"
// delete(map, key)
delete(tz, ds)
```

## <a id="Append"><span class="secondtitle">Append</span></a>

Append 可以接受多个参数
```
func append(slice []T, elements ...T) []T

// use case 
x := []int{1,2,3}
x = append(x, 4, 5, 6)
fmt.Println(x)
```

对于如果要把两个数组直接接起来的情况下
```
x := []int{1,2,3}
y := []int{4,5,6}
x = append(x, y...)
fmt.Println(x)
```

# <a id="InterFace"><span class="toptitle">InterFace</span></a>

## Interface

一个结构体可以实现多个接口，只要它实现了那些接口所定义的方法。
```
type Sequence []int

// Methods required by sort.Interface.
func (s Sequence) Len() int {
    return len(s)
}
func (s Sequence) Less(i, j int) bool {
    return s[i] < s[j]
}
func (s Sequence) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

// Copy returns a copy of the Sequence.
func (s Sequence) Copy() Sequence {
    copy := make(Sequence, 0, len(s))
    return append(copy, s...)
}

// Method for printing - sorts the elements before printing.
func (s Sequence) String() string {
    s = s.Copy() // Make a copy; don't overwrite argument.
    sort.Sort(s)
    str := "["
    for i, elem := range s { // Loop is O(N²); will fix that in next example.
        if i > 0 {
            str += " "
        }
        str += fmt.Sprint(elem)
    }
    return str + "]"
}
```

## Coversions

上面所引用的到方法其实是重新实现了fmt包里面的Sprint方法。我们可以用一些方法来为这个提速，在调用Sprint之前把数据转换成[]int类型，因为Sequence的本质上就
是[]int。  

```
// 修改前
func (s Sequence) String() string {
    s = s.Copy()
    sort.Sort(s)
    return fmt.Sprint([]int(s))
}

// 修改后
type Sequence []int

// Method for printing - sorts the elements before printing
func (s Sequence) String() string {
    s = s.Copy()
    sort.IntSlice(s).Sort()
    return fmt.Sprint([]int(s))
}
```

## type assertions

使用Type switch 实际上的操作会把那个变量根据分支的判断的类型来转型。  
下面这段代码是一个例子，目的是如果不是string类型, 调用其String()进行输出。而如果是，直接输出。
```
type Stringer interface {
    String() string
}

var value interface{} // Value provided by caller.
switch str := value.(type) {
case string:
    return str
case Stringer:
    return str.String()
}
```

如果对于使用场景是单分支，只需要判断接口是否为那个类型的实现的情况下，可以使用
```
// 
str, ok := value.(string)
```

注意的是继续类型推断的情况下，必须要填入实际的类型，不能再填入Interface。

##  Generality

如果包里面的一个类型只是为了实现接口并且其他方法不需要进行导出给外部使用的情况下。只需要导出接口就好，这样可以避免对于使用者不感兴趣的方法的实现的复杂度。
如官方的Hash模块，crc32.NewIEEE 和 adler32.New这两个方法都是返回 接口类型Hash.Hash32。

## interface & method

只要这个类型实现了这个接口的所有方法，即可以把这个类型传入来当接口使用.可以理解为简单的依赖翻转。


# <a id="Error"><span class="toptitle">Error</span></a>

## <a id="Defination"><span class="secondtitle">Defination</span></a>
Error 接口在代码里面的定义是这样的。
```
type error interface {
    Error() string
}
```

如果需要实现一个自定义的Error（添加部分与业务相关的信息）
```
// PathError records an error and the operation and
// file path that caused it.
type PathError struct {
    Op string    // "open", "unlink", etc.
    Path string  // The associated file.
    Err error    // Returned by the system call.
}

func (e *PathError) Error() string {
    return e.Op + " " + e.Path + ": " + e.Err.Error()
}
```

并且建议的可行方法，错误信息应该能够表明他们的来源（即发生错误的模块是哪个模块的哪个函数）。  
一般来说，函数调用者如果关心具体的错误信息的话，可以使用一个类型switch来获取具体的错误类型和相信信息。
```
for try := 0; try < 2; try++ {
    file, err = os.Create(filename)
    if err == nil {
        return
    }
    if e, ok := err.(*os.PathError); ok && e.Err == syscall.ENOSPC {
        deleteTempFiles()  // Recover some space.
        continue
    }
    return
}
```

## <a id="Panic"><span class="secondtitle">Panic</span></a>
对于预设以内的错误，返回错误的方式应该是返回错误的信息为多一个参数.
```

func Get() (string, Error){
    return "", nil
}

k, err := Get()
```
但是对于不可恢复的错误，我们不能让程序继续运行。  
Panic()的作用是创建一个Runtime Error并且使得程序无法继续运行。  
Panic可以接受任意长度的参数，并且打印到日志上，
```
func CubeRoot(x float64) float64 {
    z := x/3   // Arbitrary initial value
    for i := 0; i < 1e6; i++ {
        prevz := z
        z -= (z*z*z-x) / (3*z*z)
        if veryClose(z, prevz) {
            return z
        }
    }
    // A million iterations has not converged; something is wrong.
    panic(fmt.Sprintf("CubeRoot(%g) did not converge", x))
}
```

官方对于Panic的态度：程序员应该尽可能的去考虑并且解决所有的异常的情况。
对于真实的代码库上面不建议使用这个方法。

## <a id="Recover"><span class="secondtitle">Recover</span></a>

用于恢复发生Panic的Goroutine。但是必须在panic前面的地方添加一个defer 并且把Recover函数放入其中。
```
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Println("work failed:", err)
        }
    }()
    do(work)
}
```

官方库中处理复杂错误的例子，Regexp
```
// Error is the type of a parse error; it satisfies the error interface.
type Error string
func (e Error) Error() string {
    return string(e)
}

// error is a method of *Regexp that reports parsing errors by
// panicking with an Error.
func (regexp *Regexp) error(err string) {
    panic(Error(err))
}

// Compile returns a parsed representation of the regular expression.
func Compile(str string) (regexp *Regexp, err error) {
    regexp = new(Regexp)
    // doParse will panic if there is a parse error.
    defer func() {
        if e := recover(); e != nil {
            regexp = nil    // Clear return value.
            err = e.(Error) // Will re-panic if not a parse error.
        }
    }()
    return regexp.doParse(str), nil
}
```

即使上面把regexp 的类型变成了nil。但是如果在e判断类型的时候如果不是Error的类型。程序仍然会发生错误，并且崩溃推出。
```
if pos == 0 {
    re.error("'*' illegal at start of expression")
}
```
对于上面的这种re-panic的策略，官方建议是在一个包内进行使用，这样就不会把错误暴露给Client。  
虽然re-panic最终程序还是崩溃了，但是这样可以使得程序具体的错误可以过滤一层，并且找到更加直接的错误的原因。

# <a id="Init"><span class="toptitle">PackageInit</span></a>

可以直接看[译文](https://zhuanlan.zhihu.com/p/34211611)即可。

也可以直接读介绍的[原文](https://medium.com/golangspec/init-functions-in-go-eac191b3860a)

# <a id="Defer"><span class="toptitle">Defer</span></a>

Defer语法是在函数返回前进行调度的清除函数调用。它能够很好地处理多分支返回情况下的释放资源的问题。（类似于Python的With语法）

```
func Contents(filename string) (string, error) {
    f, err := os.Open(filename)
    if err != nil {
        return "", err
    }
    defer f.Close()  // f.Close will run when we're finished.

    var result []byte
    buf := make([]byte, 100)
    for {
        n, err := f.Read(buf[0:])
        result = append(result, buf[0:n]...) // append is discussed later.
        if err != nil {
            if err == io.EOF {
                break
            }
            return "", err  // f will be closed if we return here.
        }
    }
    return string(result), nil // f will be closed if we return here.
}
```

```
官方对于Defer的好处声明：
1. 位置更加接近，更好的可以清晰的看出操作
2. 防止资源忘了关闭导致的泄露问题
```

延迟函数（如果函数是方法，则包括接收方）的参数在延迟执行时而不是在调用执行时进行评估。除了避免担心函数执行时变量会更改值外，这还意味着单个延迟的调用站点可以延迟多个函数的执行。这是一个简单的例子。
```
package main

import (
	"fmt"
)

func main() {
	for i := 0; i < 5; i++ {
    		defer fmt.Printf("%d ", i)
	}
}

-------
output:
4 3 2 1 0 
```
Defer函数的执行吮吸是LIFO（LastInFirstOut)栈的结构。



```
package main

import "fmt"

func trace(s string) string {
    fmt.Println("entering:", s)
    return s
}

func un(s string) {
    fmt.Println("leaving:", s)
}

func a() {
    defer un(trace("a"))
    fmt.Println("in a")
}

func b() {
    defer un(trace("b"))
    fmt.Println("in b")
    a()
}

func main() {
    b()
}
-------------------
output
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

从上面的例子可以看出来，defer函数只能包含一层，如果像上面的代码 defer un(trace()), 那么会先执行trace()，然后再把un()函数压入defer的栈中。

[更加实际的使用场景相关的例子](https://mp.weixin.qq.com/s/_wZQID0VatIlAiH6-x3U6A)

# <a id="ShareNote"><span class="toptitle">ShareNote</span></a>
1. [EffectiveGo](https://golang.org/doc/effective_go.html)
2. [GoInitFunc](https://medium.com/golangspec/init-functions-in-go-eac191b3860a)
3. [GoInitFunc译文](https://zhuanlan.zhihu.com/p/34211611)