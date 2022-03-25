# 其他

可直接参考：

[逃逸分析对性能的影响](https://geektutu.com/post/hpg-escape-analysis.html)

[sync.Pool 复用对象](https://geektutu.com/post/hpg-sync-pool.html)

[sync.Once 如何提升性能](https://geektutu.com/post/hpg-sync-once.html)

[sync.Cond 条件变量](https://geektutu.com/post/hpg-sync-cond.html)



## bytes.Buffer 与 strings.Builder 的区别

### Buffer 和 Builder底层原理实现

Buffer 数据结构:

```javascript
// A Buffer is a variable-sized buffer of bytes with Read and Write methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
    buf      []byte // contents are the bytes buf[off : len(buf)]
    off      int    // read at &buf[off], write at &buf[len(buf)]
    lastRead readOp // last read operation, so that Unread* can work correctly.
}
```

Builder 数据结构:

```javascript
// A Builder is used to efficiently build a string using Write methods.
// It minimizes memory copying. The zero value is ready to use.
// Do not copy a non-zero Builder.
type Builder struct {
    addr *Builder // of receiver, to detect copies by value
    buf  []byte
}
```

可以看出Buffer和Builder底层都是采用[]byte数组进行装载数据。

Buffer：

```go
// Write appends the contents of p to the buffer, growing the buffer as
// needed. The return value n is the length of p; err is always nil. If the
// buffer becomes too large, Write will panic with ErrTooLarge.
func (b *Buffer) Write(p []byte) (n int, err error) {
    b.lastRead = opInvalid
    m, ok := b.tryGrowByReslice(len(p))
    if !ok {
        m = b.grow(len(p))
    }
    return copy(b.buf[m:], p), nil
}

// To build strings more efficiently, see the strings.Builder type.
func (b *Buffer) String() string {
    if b == nil {
        // Special case, useful in debugging.
        return "<nil>"
    }
    return string(b.buf[b.off:])
}
```

创建好Buffer是一个empty的，off 用于指向读写的尾部。其String()方法就是将字节数组强转为string



Builder:

```go
// Write appends the contents of p to b's buffer.
// Write always returns len(p), nil.
func (b *Builder) Write(p []byte) (int, error) {
    b.copyCheck()
    b.buf = append(b.buf, p...)
    return len(p), nil
}
```

Builder采用append的方式向字节数组后添加字符串。[]byte的内存大小也是以倍数进行申请的，初始大小为 0，第一次为大于当前申请的最大 2 的指数，不够进行翻倍。

如果旧容量小于1024进行翻倍，否则扩展四分之一。（2048 byte 后，申请策略的调整）。

```javascript
// String returns the accumulated string.
func (b *Builder) String() string {
    return *(*string)(unsafe.Pointer(&b.buf))
}
```

其次String()方法与Buffer的string方法也有明显区别。Buffer的string是一种强转，我们知道在强转的时候是需要进行申请空间，并拷贝的。而Builder只是指针的转换。

```javascript
Pointer类型代表了任意一种类型的指针，类型Pointer有四种专属的操作：

任意类型的指针能够被转换成Pointer值
一个Pointer值能够被转换成任意类型的指针值
一个uintptr值能够被转换从Pointer值
一个Pointer值能够被转换成uintptr值
```

也就是说，unsafe.Pointer 可以转换为任意类型，那么意味着，通过unsafe.Pointer媒介，程序绕过类型系统，进行地址转换而不是拷贝。

即*A => Pointer => *B

```javascript
func main() {
    var b = []byte{'H', 'E', 'L', 'L', 'O'}

    s := *(*string)(unsafe.Pointer(&b))

    fmt.Println("b =", b)
    fmt.Println("s =", s)

    b[1] = 'B'
    fmt.Println("s =", s)

    s = "WORLD"
    fmt.Println("b =", b)
    fmt.Println("s =", s)
    
    //b = [72 69 76 76 79]
    //s = HELLO
    //s = HBLLO
    //b = [72 66 76 76 79]
    //s = WORLD
}
```

就像上面例子一样，将字节数组转为unsafe.Pointer类型，再转为string类型，s和b中内容一样，修改b,s也变了，说明b和s是同一个地址。但是对s重新赋值后，意味着s的地址指向了“WORLD”,它们所使用的内存空间不同了，所以s改变后，b并不会改变。

所以他们的区别就在于 bytes.Buffer 是重新申请了一块空间，存放生成的string变量， 而strings.Builder直接将底层的[]byte转换成了string类型返回了回来，去掉了申请空间的操作。所以在极端情况下，拼接字符串的效率，strings.Builder 比 bytes.Buffer 要快。