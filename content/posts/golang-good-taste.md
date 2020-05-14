---
title: "Golang代码中的好味道与坏味道"
date: 2020-05-13T18:55:00+08:00
draft: false
author: "Jessppp"
authorLink: "https://blog.kswii.com"
description: "Golang代码中的好味道与坏味道"

tags: ["Golang", "Coding"]
categories: ["Golang"]

lightgallery: true
---

## 前言
当我们在写代码的时候，总会存在一些好味道和坏味道的代码，当你应用体量不是很大的时候，他们之间的差别不会很明显，但是当你的体量达到BAT那种级别的时候，一个小小的细节可能会让你付出惨痛的代价

### 1.Slice的好味道与坏味道

我们看下测试代码

```
package test

import "testing"

func BenchmarkSlice1(b *testing.B) {
	s := make([]int, 0)
	for i := 0; i < b.N; i++ {
		s = append(s, i)
	}
}

func BenchmarkSlice2(b *testing.B) {
	s := make([]int, 0, b.N)
	for i := 0; i < b.N; i++ {
		s = append(s, i)
	}
}
```

测试结果

```
goos: windows
goarch: amd64
BenchmarkSlice1-8       100000000               10.5 ns/op            49 B/op          0 allocs/op
BenchmarkSlice2-8       1000000000               2.52 ns/op            8 B/op          0 allocs/op
PASS
ok      command-line-arguments  4.401s

```

我们可以看到slice初始化的时候,第三个cap参数当在数据量比较大的情况下，性能优势相当明显，这里面的原理涉及到slice底层的数据结构和扩容机制，想了解的小伙伴可以跟下源码了解

### 2.Map的好味道与坏味道

我们看下测试代码

```
package test

import "testing"

func BenchmarkMap1(b *testing.B) {
	m := make(map[int]int)
	for i := 0; i < b.N; i++ {
		m[i] = i
	}
}

func BenchmarkMap2(b *testing.B) {
	m := make(map[int]int, b.N)
	for i := 0; i < b.N; i++ {
		m[i] = i
	}
}
```

测试结果

```
goos: windows
goarch: amd64
BenchmarkMap1-8          7597566               203 ns/op              92 B/op          0 allocs/op
BenchmarkMap2-8         15782391                99.6 ns/op            40 B/op          0 allocs/op
PASS
ok      command-line-arguments  3.624s

```

我们可以看到map初始化的第二个参数当数据量大的时候性能优势也是非常明显，原理和上面slice自动扩容的机制类似，我们这边不做过多讲解

### 3.结构体构造时候的好味道与坏味道

我们看下测试代码

```
package test

import (
	"fmt"
	"testing"
	"unsafe"
)

func TestMem(t *testing.T) {
	type part1 struct {
		a bool
		b int32
		c int8
		d int64
		e byte
	}
	type part2 struct {
		a bool
		e byte
		c int8
		b int32
		d int64
	}
	fmt.Printf("part1 size: %d\n", unsafe.Sizeof(part1{}))
	fmt.Printf("part2 size: %d\n", unsafe.Sizeof(part2{}))
}

```

测试结果

```
part1 size: 32
part2 size: 16
PASS
ok      command-line-arguments  0.196s

```

我们看到part1和part2的内存占用大小相差了一倍，具体是什么原因造成这个现象的呢？有兴趣的同学可以百度一下`golang内存对齐`就可以找到答案

### 4.WaitGroup Add的好味道与坏味道

我们看下测试代码

```
package test

import (
	"sync"
	"testing"
)

func BenchmarkWait1(b *testing.B) {
	var wg sync.WaitGroup
	wg.Add(b.N)
	for i := 0; i < b.N; i++ {
		go func() {
			wg.Done()
		}()
	}
	wg.Wait()
}

func BenchmarkWait2(b *testing.B) {
	var wg sync.WaitGroup
	for i := 0; i < b.N; i++ {
		wg.Add(1)
		go func() {
			wg.Done()
		}()
	}
	wg.Wait()
}

```

测试结果

```
goos: windows
goarch: amd64
BenchmarkWait1-8         5310784               227 ns/op               0 B/op          0 allocs/op
BenchmarkWait2-8         4239832               366 ns/op               0 B/op          0 allocs/op
PASS
ok      command-line-arguments  3.512s

```

我们团队小伙伴在写waitgroup的时候经常会使用第二种方法,其实你在程序中写的每一行代码都会被编译成对应的字节码，所以为什么很多时候会听到说**越精简的代码性能越高**是有一定道理的，我们这个测试结果虽然性能差异不是很大，但能看到方法1确实比方法2少调用了N-1次的wg.Add方法，所以这块相信小伙伴也知道怎么样的用法最优啦