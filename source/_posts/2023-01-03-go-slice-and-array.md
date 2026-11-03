---
title: Go 的 Slice 与数组
date: 2023-01-03
categories:
  - go
tags:
  - Go
  - Slice
  - 数组
---

数组和切片看起来都像“一串同类型元素”，但 Go 把它们设计成了两种完全不同的东西：数组是值类型，长度属于类型的一部分；切片是一个描述底层数组片段的引用式结构，长度可变。混淆这两者，是很多越界、意外共享和内存占用问题的来源。

本文先对比两者的语义，再拆开切片的描述符、扩容和别名行为，最后给出常见坑与工程建议。运行时细节以 Go 1.25.12 为准，属于实现而非语言规范。

## 1. 数组：长度是类型的一部分

数组类型的写法是 `[N]T`，`N` 是编译期常量：

```go
var a [3]int          // 类型 [3]int
b := [3]int{1, 2, 3}
c := [...]int{1, 2, 3} // 由元素个数推导，类型仍是 [3]int

// a = b 合法，因为类型相同
// var d [4]int; a = d 编译错误：类型不同
```

`[3]int` 和 `[4]int` 是两个不同的类型，不能互相赋值，这直接体现了“长度是类型的一部分”。

数组是值语义：赋值、作为参数传递、作为返回值，都会复制整个数组。数组越大，复制成本越高。

```go
func modify(arr [3]int) {
    arr[0] = 100 // 改的是副本
}

func main() {
    a := [3]int{1, 2, 3}
    modify(a)
    fmt.Println(a) // [1 2 3]
}
```

如果想在函数里修改原数组，需要传指针 `*[3]int`，或者干脆改用切片。

数组也有它的优势：

- **可比较**。元素类型可比较时，数组可以用 `==`、`!=`，因此能作为 map 的键；切片不能。
- **内存布局确定**。数组内联在结构体或变量中，没有额外指针间接，适合固定大小的缓冲区、哈希值、坐标等场景。

数组转切片只需要切片表达式：

```go
arr := [5]int{1, 2, 3, 4, 5}
s := arr[:]      // 引用 arr 的底层存储
s[0] = 100
fmt.Println(arr[0]) // 100
```

注意 `s` 没有复制数组，它指向 `arr` 本身。

## 2. 切片：三个字段的描述符

规范对切片的定义是：切片是底层数组中一段连续区间的描述符（descriptor）。常见的实现是三个机器字：

```go
type slice struct {
    array unsafe.Pointer // 指向底层数组
    len   int            // 长度
    cap   int            // 容量
}
```

`reflect.SliceHeader` 把同样的信息暴露为 `Data`、`Len`、`Cap` 三个字段。切片变量本身很小，复制切片只复制这三个字段，不会复制元素。

```go
s := make([]int, 3, 5)
fmt.Println(len(s), cap(s)) // 3 5
```

`make([]T, len, cap)` 中 `cap` 可以省略，省略时等于 `len`；`cap` 必须不小于 `len`。长度是当前可访问元素个数，容量是从起始位置到底层数组末尾还能容纳的元素个数。

切片的零值是 nil：

```go
var s []int      // nil，len 0，cap 0
s = append(s, 1) // append 会分配
```

nil 切片和空切片 `[]int{}` 在 `len`、`cap`、`range` 上表现一致，但两者不完全等价：

- nil 切片只能和 nil 比较，不能和空切片比较；
- `encoding/json` 中 nil 切片编码为 `null`，空切片编码为 `[]`；
- 如果 API 要区分“没有数据”和“空集合”，这个差异会体现在协议上。

## 3. 切片表达式与容量传播

对数组或切片做切片：

```go
a := [5]int{1, 2, 3, 4, 5}
s := a[1:3] // len = 3-1 = 2，cap = 5-1 = 4
```

规则是 `len = high - low`，`cap = cap(原操作数) - low`。也就是说，从中间切出来的切片，容量会一直延伸到原底层数组末尾，而不是到 `high` 为止。

这个“容量延伸到末尾”既是 `append` 高效的原因，也是别名问题的根源：

```go
base := []int{0, 1, 2, 3, 4}
sub := base[1:3]         // len 2, cap 4
sub = append(sub, 99)    // 容量够，直接写入 base[3]
fmt.Println(base)        // [0 1 2 99 4]
```

如果希望限制后续 `append` 的影响范围，用**完整切片表达式** `a[low:high:max]` 显式限制容量：

```go
sub := base[1:3:3] // len 2, cap 2
sub = append(sub, 99) // 容量不足，分配新数组，base 不受影响
```

`max` 只能出现在完整形式里，且必须满足 `low <= high <= max <= cap(a)`。

## 4. append 与扩容

`append` 返回新的切片值，是否复用底层数组取决于剩余容量：

```go
s := make([]int, 0, 4)
s = append(s, 1, 2) // 复用底层数组
s = append(s, 3, 4, 5) // 超过 cap，分配新数组并复制
```

必须把返回值赋回去：切片是按值传递的描述符，`append` 可能产生新的 `array`、`len`、`cap`，原变量不会自动更新。

```go
func bad(s []int) {
    s = append(s, 1) // 调用方看不到
}

func good(s []int) []int {
    return append(s, 1)
}
```

容量不足时，运行时会计算新容量。Go 1.25.12 的 `runtime/slice.go` 中：

- 需要的新长度超过两倍旧容量时，直接取新长度；
- 旧容量小于 256 时，按两倍增长；
- 更大的容量则以约 1.25 倍增长（`newcap += (newcap + 3*256) >> 2`），让大小容量之间的过渡更平滑；
- 最终还会按内存分配器的 size class 向上取整，所以 `cap` 往往大于按公式算出的值。

这段逻辑属于实现细节：不同 Go 版本可能调整增长率，代码不应依赖具体的 `cap` 数值。能依赖的只有规范保证：`append` 后 `len` 增加，容量足够时复用数组，否则重新分配。

一个实践含义是：明确知道要装多少元素时，用 `make([]T, 0, n)` 预分配，可以避免多次扩容和复制。

## 5. 共享底层数组与内存滞留

切片之间可以共享底层数组，这是设计目标，也是最容易出问题的地方。

```go
data := make([]byte, 1<<20)
header := data[:16]
// header 很小，但它让整个 1MB 的底层数组无法被回收
```

只要还有一个切片引用了底层数组的某一段，整个底层数组就都存活。常见的处理方式是复制出真正需要的小切片：

```go
small := make([]byte, 16)
copy(small, data[:16])
// 或者 Go 1.21+：
// small := bytes.Clone(data[:16])
```

同理，从大 slice 里筛选结果时，用 `append` 到新切片（或 `slices.Clone`）而不是反复复用原数组的片段，能避免长期持有大对象。

另一个方向的风险是**意外改写**：两个切片共享数组，一方 `append` 到容量范围内，另一方会看到变化。跨模块传递切片时，如果需要隔离，就应当复制；如果能接受共享，就要在文档里写清楚。

## 6. copy、range 与遍历语义

`copy(dst, src)` 复制的元素个数是 `min(len(dst), len(src))`，不会扩容：

```go
dst := make([]int, 2)
n := copy(dst, []int{1, 2, 3})
fmt.Println(n, dst) // 2 [1 2]
```

`copy` 允许源和目标重叠，内部会正确处理，因此可以用它做元素搬移（如删除中间元素后前移）。

`for range` 遍历切片时，range 表达式在循环开始前求值一次，循环使用的是当时的切片描述符：

```go
s := []int{1, 2, 3}
for i, v := range s {
    if i == 0 {
        s = append(s, 4) // 不会让循环变成 4 次
    }
    fmt.Println(i, v)
}
```

同时，`v` 是元素的副本：

```go
for _, v := range s {
    v *= 2 // 改的是副本，s 不变
}

for i := range s {
    s[i] *= 2 // 这样才能修改元素
}
```

如果元素是结构体或含指针的小对象，复制成本需要纳入考虑；数据量大时用索引访问往往更直接。

## 7. 多维切片

二维切片是“切片的切片”，每一行需要单独初始化：

```go
rows, cols := 3, 4
grid := make([][]int, rows)
for i := range grid {
    grid[i] = make([]int, cols)
}
```

不能只写 `make([][]int, rows)` 就直接 `grid[0][0] = 1`——内层切片还是 nil，会 panic。

如果矩阵尺寸固定且追求更好的局部性，可以用一维切片模拟二维：

```go
grid := make([]int, rows*cols)
at := func(r, c int) int { return r*cols + c }
grid[at(1, 2)] = 7
```

这样只有一次分配、内存连续，缓存友好，代价是需要自己维护索引换算。

## 8. 常见坑与建议

1. **把数组当切片用**：函数参数写 `[N]T` 会复制整个数组，且不同长度不兼容。需要动态长度就用切片。
2. **忽略 `append` 的返回值**：`append(s, x)` 不赋回，可能什么都没发生。
3. **在函数内 append 后期待调用方可见**：切片描述符按值传递，长度变化不会回传；要么返回新切片，要么传 `*[]T`。
4. **误判容量**：`s[i:j]` 的 `cap` 是 `cap(s)-i`，不是 `j-i`；需要隔离就用三索引切片。
5. **长期持有小切片**：小切片可能钉住大底层数组，必要时 `copy` 或 `slices.Clone`。
6. **并发写同一底层数组**：切片本身没有同步语义。多个 goroutine 写同一数组的不同区间是数据竞争，需要加锁或分区。
7. **依赖具体容量**：扩容策略是实现细节，不要写出依赖 `cap` 精确值的逻辑。

日常操作优先用标准库 `slices` 包（Go 1.21+）：`slices.Clone`、`slices.Equal`、`slices.Contains`、`slices.Sort`、`slices.Delete` 等，比手写循环更不容易出错。

## 9. 小结

数组是定长、值语义、可比较的连续存储；切片是带 `array/len/cap` 的描述符，指向一段可共享的底层数组。切片的高效来自“复用底层数组”，它的复杂度也来自同一件事：容量会传播、append 可能原地写、子切片可能延长大对象寿命。

记住三个判断点：**是否需要动态长度（选切片）、是否需要值语义或可比较（选数组）、共享底层数组是否可接受（决定是否复制）**，大部分设计与排查都能落在这三点上。

## 10. 参考资料

- [The Go Programming Language Specification：Array types](https://go.dev/ref/spec#Array_types)、[Slice types](https://go.dev/ref/spec#Slice_types)、[Slice expressions](https://go.dev/ref/spec#Slice_expressions)、[Appending to and copying slices](https://go.dev/ref/spec#Appending_and_copying_slices)
- [Go Slices: usage and internals](https://go.dev/blog/slices-intro)
- [Go 1.25.12 runtime/slice.go（growslice 与 nextslicecap）](https://github.com/golang/go/blob/go1.25.12/src/runtime/slice.go)
- [reflect.SliceHeader](https://pkg.go.dev/reflect#SliceHeader)
- [slices 包文档](https://pkg.go.dev/slices)
