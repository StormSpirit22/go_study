# 3.1 数组与切片

## 数组与切片

数组就是一片连续的内存， slice 实际上是一个结构体，包含三个字段：长度、容量、底层数组。

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int // 长度 
    cap   int // 容量
}
```

【引申1】 [3]int 和 [4]int 是同一个类型吗？

不是。因为数组的长度是类型的一部分，这是与 slice 不同的一点。

【引申2】 下面的代码输出是什么？

说明：例子来自雨痕大佬《Go学习笔记》第四版，P43页。这里我会进行扩展，并会作图详细分析。

```go
package main

import "fmt"

func main() {
    slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
    s1 := slice[2:5]
    s2 := s1[2:6:7]

    s2 = append(s2, 100)
    s2 = append(s2, 200)

    s1[2] = 20

    fmt.Println(s1)
    fmt.Println(s2)
    fmt.Println(slice)
}
```

结果：

```
[2 3 20]
[4 5 6 7 100 200]
[0 1 2 3 20 5 6 7 100 9]
```

`s1` 从 `slice` 索引2（闭区间）到索引5（开区间，元素真正取到索引4），长度为3，容量默认到数组结尾，为8。 `s2` 从 `s1` 的索引2（闭区间）到索引6（开区间，元素真正取到索引5），容量到索引7（开区间，真正到索引6），为5。

![img](../../.go_study/assets/images3/slice-1.png)

接着，向 `s2` 尾部追加一个元素 100：

```go
s2 = append(s2, 100)
```

`s2` 容量刚好够，直接追加。不过，这会修改原始数组对应位置的元素。这一改动，数组和 `s1` 都可以看得到。

![img](../../.go_study/assets/images3/slice-2.png)



再次向 `s2` 追加元素200：

```go
s2 = append(s2, 100)
```

这时，`s2` 的容量不够用，该扩容了。于是，`s2` 另起炉灶，将原来的元素复制新的位置，扩大自己的容量。并且为了应对未来可能的 `append` 带来的再一次扩容，`s2` 会在此次扩容的时候多留一些 `buffer`，将新的容量将扩大为原始容量的2倍，也就是10了。

![img](../../.go_study/assets/images3/slice-3.png)

最后，修改 `s1` 索引为2位置的元素：

```go
s1[2] = 20
```

这次只会影响原始数组相应位置的元素。它影响不到 `s2` 了，人家已经远走高飞了。

![img](../../.go_study/assets/images3/slice-4.png)



再提一点，打印 `s1` 的时候，只会打印出 `s1` 长度以内的元素。所以，只会打印出3个元素，虽然它的底层数组不止3个元素。



## 切片作为函数参数

Go 语言的函数参数传递，只有值传递，没有引用传递。

```go
package main

func main() {
    s := []int{1, 1, 1}
    f(s)
    fmt.Println(s)
}

func f(s []int) {
    // i只是一个副本，不能改变s中元素的值
    /*for _, i := range s {
        i++
    }
    */

    for i := range s {
        s[i] += 1
    }
}
```

程序输出：

```
[2 2 2]
```

要想真的改变外层 `slice`，只有将返回的新的 slice 赋值到原始 slice，或者向函数传递一个指向 slice 的指针。

```go
package main

import "fmt"

func myAppend(s []int) []int {
    // 这里 s 虽然改变了，但并不会影响外层函数的 s
    s = append(s, 100)
    return s
}

func myAppendPtr(s *[]int) {
    // 会改变外层 s 本身
    *s = append(*s, 100)
    return
}

func main() {
    s := []int{1, 1, 1}
    newS := myAppend(s)

    fmt.Println(s)
    fmt.Println(newS)

    s = newS

    myAppendPtr(&s)
    fmt.Println(s)
}
```

输出：

```
[1 1 1]
[1 1 1 100]
[1 1 1 100 100]
```

## 切片的容量是怎样增长的

当原 slice 容量小于 `1024` 的时候，新 slice 容量变成原来的 `2` 倍；原 slice 容量超过 `1024`，新 slice 容量变成原来的`1.25`倍。

进行上面一步的处理后，go 还会对 slice 进行一个内存对齐，之后新 slice 的容量是要 `大于等于` 老 slice 容量的 `2倍`或者`1.25倍`。然后再向 Go 内存管理器申请内存，将老 slice 中的数据复制过去，并且将 append 的元素添加到新的底层数组中。

这里要注意的是，append函数执行完后，返回的是一个全新的 slice，并且对传入的 slice 并不影响。

关于 `append`，我们最后来看一个例子，来源于 [Golang Slice的扩容规则](https://jodezer.github.io/2017/05/golangSlice的扩容规则)。

```go
package main

import "fmt"

func main() {
    s := []int{1,2}
    s = append(s,4,5,6)
    fmt.Printf("len=%d, cap=%d",len(s),cap(s))
}
```

输出：

```
len=5, cap=6
```



内存对齐参考

[在 Go 中恰到好处的内存对齐](https://segmentfault.com/a/1190000017527311) （推荐看！）

[【Golang】详解内存对齐](https://segmentfault.com/a/1190000040528007)