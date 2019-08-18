---
layout:     post
title:      "聊一聊字符串内部化"
subtitle:   "String Interning"
date:       2019-08-18
author:     "ms2008"
header-img: "img/post-bg-golang-blog-hero.jpg"
catalog:    true
tags:
    - Golang
    - Lua
    - C
typora-root-url: ..
---

### 缘起

字符串作为一种*不可变*值类型，在多数的语言里，**其底层基本都是个只读的字节数组：一旦被创建，则不可被改写**。正是因为其只读特性，如果有大量相同的字符串需要处理，那么在内存中就会保存多份，显然是非常浪费内存的。

> 对于 C 来说字符串本质上就是 `const char*`；而对于 Lua，虽然字符串并不是以 `\0` 结尾，但是 `TString` 的数据本质上也是一个 `const char*`

**所谓字符串内部化（string interning），就是一种技术手段让相同的字符串在内存中只保留一份**。这样就可以大幅降低内存占用，缩短字符串比较的时间。因为相同的字符串只需要保存一份在内存中，当用这个字符串做匹配时，比较字符串只需要比较地址是否相同就够了，而不必逐字节比较。于是时间复杂度就从 `O(N)` 降低到了 `O(1)`。

> 目前基本上所有主流的语言都使用了这项技术。比如，Lua 5.2 以前所有的字符串会被内部化到一张表中，这张表挂在 global state 结构下，相同的字符串在同一 VM 只会存在一份

而 Go 的字符串，本质上是一个 `reflect.StringHeader`:

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
```

其中 `Data` 指针<u>指向的是一个字符常量的地址，这个地址里面的内容是不可以被改变的，因为它是只读的，但是这个指针可以指向不同的地址</u>。虽然相同的字符串是不同的 `StringHeader`，但是其内部实际上都指向相同的字节数组：

```go
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

func main() {
	str1 := "Hello, World!"
	str2 := "Hello, World!"
	// 这两个地址并不相同
	fmt.Printf("str add: %p, %p\n", &str1, &str2)

	x1 := (*reflect.SliceHeader)(unsafe.Pointer(&str1))
	x2 := (*reflect.SliceHeader)(unsafe.Pointer(&str2))
	// 底层都是指向相同的 []byte
	fmt.Printf("data add: %#v, %#v\n", x1.Data, x2.Data)
}
```

<u>需要注意的是 Go 的 string intern 仅仅针对的是编译期可以确定的字符串常量，如果是运行期间产生的字符串则不能被内部化</u>。比如：

```go
// 可以被 intern
s1 := "12"
s2 := "1"+"2"

// 不能被 intern
s3 := "12"
s4 := strconv.Itoa(12)
```

因为 string 的指针指向的内容是不可以更改的，所以每更改一次字符串，就得重新分配一次内存，之前分配空间的还得由 gc 回收，这是导致 string 操作低效的根本原因。

### Hack it

了解了它的机制之后，我们可以试着来绕过其限制，来完成一个可以内部化所有字符串的实现。首先我们需要一个 pool，把所有的字符串都放到这个 pool 里，只要字符串在这个 pool 里只有一份（例如 Map 就是一个非常好的选择），就可以认为已经被 intern 了。下面是一个老外的实现：

```go
package main

import (
	"fmt"
	"strconv"
)

type stringInterner map[string]string

func (si stringInterner) Intern(s string) string {
	if interned, ok := si[s]; ok {
		return interned
	}
	si[s] = s
	return s
}

func main() {
	si := stringInterner{}
	s1 := si.Intern("12")
	s2 := si.Intern(strconv.Itoa(12))
	fmt.Println(stringptr(s1) == stringptr(s2)) // true
}
```

他在优化了他们的一个服务后，可以看到内存占用下降的非常明显：

![](/img/in-post/string-interning-mem.png)

string intern 除了可以显著降低内存外，还有一个优点就是降低相同字符串的比较时间：

```go
package main

import (
	"strings"
	"testing"
)

type stringInterner map[string]string

func (si stringInterner) Intern(s string) string {
	if interned, ok := si[s]; ok {
		return interned
	}
	si[s] = s
	return s
}

func benchmarkStringCompare(b *testing.B, count int) {
	s1 := strings.Repeat("a", count)
	s2 := strings.Repeat("a", count)
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		if s1 != s2 {
			b.Fatal()
		}
	}
}

func benchmarkStringCompareIntern(b *testing.B, count int) {
	si := stringInterner{}
	s1 := si.Intern(strings.Repeat("a", count))
	s2 := si.Intern(strings.Repeat("a", count))
	b.ResetTimer()
	for n := 0; n < b.N; n++ {
		if s1 != s2 {
			b.Fatal()
		}
	}
}

func BenchmarkStringCompare1(b *testing.B)   { benchmarkStringCompare(b, 1) }
func BenchmarkStringCompare10(b *testing.B)  { benchmarkStringCompare(b, 10) }
func BenchmarkStringCompare100(b *testing.B) { benchmarkStringCompare(b, 100) }

func BenchmarkStringCompareIntern1(b *testing.B)   { benchmarkStringCompareIntern(b, 1) }
func BenchmarkStringCompareIntern10(b *testing.B)  { benchmarkStringCompareIntern(b, 10) }
func BenchmarkStringCompareIntern100(b *testing.B) { benchmarkStringCompareIntern(b, 100) }
```

可以看到被 intern 的字符串对比时间基本是个常数，而未被 intern 的字符串比较呈现出一个 `O(N)` 趋势：

```
BenchmarkStringCompare1-2           	300000000	         4.65 ns/op	       0 B/op	       0 allocs/op
BenchmarkStringCompare10-2          	300000000	         5.49 ns/op	       0 B/op	       0 allocs/op
BenchmarkStringCompare100-2         	200000000	         8.96 ns/op	       0 B/op	       0 allocs/op
BenchmarkStringCompareIntern1-2     	500000000	         3.69 ns/op	       0 B/op	       0 allocs/op
BenchmarkStringCompareIntern10-2    	500000000	         3.65 ns/op	       0 B/op	       0 allocs/op
BenchmarkStringCompareIntern100-2   	500000000	         3.65 ns/op	       0 B/op	       0 allocs/op
```

**实际上 Go 对比两个字符串是否相等，首先会对比其长度，长度不同自然是不同的串，时间复杂度为 `O(1)`；如果长度相同，再对比其底层字节数组地址，地址相同肯定是相同的串，时间复杂度仍然可以认为是 `O(1)`；如果地址不同，则需要逐个对比字节，那么时间复杂度也就退化为了 `O(N)`**。

string intern 作为一种高效的手段，在 Go 内部也有不少应用，比如在 HTTP 中 intern 公用的请求头来避免多余的内存分配：

```go
// commonHeader interns common header strings.
var commonHeader = make(map[string]string)

func init() {
	for _, v := range []string{
		"Accept",
		"Accept-Charset",
		"Accept-Encoding",
		"Accept-Language",
		"Accept-Ranges",
		"Cache-Control",
		// ...
	} {
		commonHeader[v] = v
	}
}
```

如果你在做缓存系统，或者是需要操作大量的字符串，不妨也考虑下 string intern 来优化你的应用。

### 参考文献

- [String interning in Go][1]
- [Strings in Go][3]
- [Go黑技巧][2]

[1]: https://artem.krylysov.com/blog/2018/12/12/string-interning-in-go/
[2]: https://lihaoquan.me/2016/11/19/go-magic.html
[3]: https://go101.org/article/string.html
