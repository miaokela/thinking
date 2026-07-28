# Go Runtime 五大核心文件源码深度分析

> 分析对象为 Go 1.22+ 的 `runtime` 包源码，覆盖 `slice.go`、`map.go`、`chan.go`、`proc.go`、`mgc.go` 五个文件，共 12,528 行。

---

## 📁 文件总览

| 文件 | 行数 | 职责 |
|---|---|---|
| `slice.go` | 401 | slice 底层数组分配与扩容 |
| `map.go` | 1,911 | 哈希表实现（bucket、渐进式扩容、迭代） |
| `chan.go` | 935 | channel 发送/接收/关闭的完整实现 |
| `proc.go` | 7,312 | GMP 调度器、goroutine 生命周期、抢占 |
| `mgc.go` | 1,969 | GC 三色标记-清除的主控逻辑 |

---

## 1. slice.go — slice 的"心脏"

### 1.1 核心结构（第 15-19 行）

```go
type slice struct {
    array unsafe.Pointer  // 指向底层数组
    len   int
    cap   int
}
```

只有 3 个字段、24 字节。**这就是为什么 slice 传参是"值拷贝但看起来像引用"——你拷贝的是这 24 字节的描述符，不是底层数组。**

还有一个运行时内部专用的变体（第 22-26 行）：

```go
type notInHeapSlice struct {
    array *notInHeap
    len   int
    cap   int
}
```

用于运行时内部不被 GC 管理的内存。

### 1.2 `makeslice`（第 101-117 行）—— `make([]T, len, cap)` 的真相

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    mem, overflow := math.MulUintptr(et.Size_, uintptr(cap))
    if overflow || mem > maxAlloc || len < 0 || len > cap {
        // 先检查 cap，但报错时报 len——更友好的错误信息
        mem, overflow := math.MulUintptr(et.Size_, uintptr(len))
        if overflow || mem > maxAlloc || len < 0 {
            panicmakeslicelen()
        }
        panicmakeslicecap()
    }
    return mallocgc(mem, et, true)  // 直接调用内存分配器
}
```

关键点：
- `make([]T, len, cap)` 本质上就是**算好大小 → 调 `mallocgc` 分配内存 → 返回指针**
- 第三个参数 `true` 表示"需要清零"
- 先检查 `cap`，但错误信息报 `len`——源码注释（第 106-108 行）解释了原因：当用户写 `make([]T, bignumber)` 时，`cap` 是隐式提供的，报 `len` 更清晰
- `makeslice64`（第 119-131 行）是 64 位版本，增加了溢出检查

还有一个 `makeslicecopy`（第 38-90 行），用于 `slices.Grow` 等场景——分配新内存后立即拷贝旧数据，避免两次分配。

### 1.3 `growslice`（第 177-286 行）—— `append` 扩容的核心

这是 slice 最重要的函数。调用约定很"怪"——它接收 `newLen`（新长度）和 `num`（新增元素数），然后自己算 `oldLen = newLen - num`。源码注释（第 159-163 行）解释了：

> growslice's odd calling convention makes the generated code that calls this function simpler. In particular, it accepts and returns the new length so that the old length is not live (does not need to be spilled/restored).

**扩容流程：**

**第一步：计算新容量**（第 200 行）

```go
newcap := nextslicecap(newLen, oldCap)
```

**第二步：按元素大小特化处理**（第 209-243 行）

```go
switch {
case et.Size_ == 1:
    // 直接用字节数，无需乘法
    capmem = roundupsize(uintptr(newcap), noscan)
case et.Size_ == goarch.PtrSize:
    // 编译器会优化成位移
    capmem = roundupsize(uintptr(newcap)*goarch.PtrSize, noscan)
case isPowerOfTwo(et.Size_):
    // 用位移代替乘法
    shift := uintptr(sys.TrailingZeros64(uint64(et.Size_))) & 63
    capmem = roundupsize(uintptr(newcap)<<shift, noscan)
default:
    // 通用乘法
    capmem = roundupsize(capmem, noscan)
}
```

4 种分支对应不同的性能优化级别。

**第三步：内存对齐修正**（`roundupsize`）

实际分配大小会被对齐到内存分配器的 size class（如 8B、16B、32B...），这意味着实际容量可能比你要求的大。

**第四步：分配新内存并拷贝**（第 262-283 行）

```go
if !et.Pointers() {
    p = mallocgc(capmem, nil, false)           // 无指针：不清零
    memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)  // 只清零尾部
} else {
    p = mallocgc(capmem, et, true)              // 有指针：需要清零（GC 扫描）
    if lenmem > 0 && writeBarrier.enabled {
        bulkBarrierPreWriteSrcOnly(...)          // 写屏障
    }
}
memmove(p, oldPtr, lenmem)                      // 拷贝旧数据
```

### 1.4 `nextslicecap`（第 289-321 行）—— 扩容公式

```go
func nextslicecap(newLen, oldCap int) int {
    newcap := oldCap
    doublecap := newcap + newcap
    if newLen > doublecap {
        return newLen  // 需要的比两倍还大，直接用 newLen
    }

    const threshold = 256
    if oldCap < threshold {
        return doublecap  // 小 slice：翻倍
    }
    for {
        // 大 slice：平滑过渡，从 2x 渐降到 1.25x
        newcap += (newcap + 3*threshold) >> 2
        if uint(newcap) >= uint(newLen) {
            break
        }
    }
    if newcap <= 0 {
        return newLen
    }
    return newcap
}
```

**这就是 1.18 之后的"平滑扩容"策略**：
- `cap < 256`：翻倍（2x）
- `cap >= 256`：每次增长 `newcap += (newcap + 768) / 4`，即从 2x 平滑过渡到 1.25x
- 源码注释（第 301-303 行）明确说了：Transition from growing 2x for small slices to growing 1.25x for large slices.

### 1.5 `slicecopy`（第 355-392 行）—— `copy()` 内置函数

```go
func slicecopy(toPtr unsafe.Pointer, toLen int, fromPtr unsafe.Pointer, fromLen int, width uintptr) int {
    n := fromLen
    if toLen < n { n = toLen }
    if width == 0 { return n }
    size := uintptr(n) * width
    if size == 1 {
        *(*byte)(toPtr) = *(*byte)(fromPtr)  // 单字节特化，约 2x 加速
    } else {
        memmove(toPtr, fromPtr, size)  // 通用情况用 memmove
    }
    return n
}
```

### 1.6 `reflect_growslice`（第 332-348 行）—— reflect 包的专用入口

```go
func reflect_growslice(et *_type, old slice, num int) slice {
    num -= old.cap - old.len
    new := growslice(old.array, old.cap+num, old.cap, num, et)
    // growslice 假设调用者会覆盖 [oldLen, newLen)，但 reflect 不会
    // 所以需要手动清零这部分
    if !et.Pointers() {
        memclrNoHeapPointers(add(new.array, oldcapmem), newlenmem-oldcapmem)
    }
    new.len = old.len  // 保持旧长度
    return new
}
```

### 1.7 `bytealg_MakeNoZero`（第 394-401 行）—— 零拷贝优化

```go
func bytealg_MakeNoZero(len int) []byte {
    cap := roundupsize(uintptr(len), true)
    return unsafe.Slice((*byte)(mallocgc(uintptr(cap), nil, false)), cap)[:len]
}
```

用于 `internal/bytealg`，分配但不清零——调用者保证会立即写入。

---

## 2. map.go — 哈希表的完整实现

### 2.1 文件头部的设计注释（第 8-54 行）

源码注释是理解 map 设计的最佳材料：

> A map is just a hash table. The data is arranged into an array of buckets. Each bucket contains up to 8 key/elem pairs. The low-order bits of the hash are used to select a bucket. Each bucket contains a few high-order bits of each hash to distinguish the entries within a single bucket.

装载因子的选择有一个完整的性能测试表（第 37-54 行）：

| loadFactor | %overflow | bytes/entry | hitprobe | missprobe |
|---|---|---|---|---|
| 4.00 | 2.13% | 20.77 | 3.00 | 4.00 |
| 4.50 | 4.05% | 17.30 | 3.25 | 4.50 |
| 5.00 | 6.85% | 14.77 | 3.50 | 5.00 |
| 5.50 | 10.55% | 12.94 | 3.75 | 5.50 |
| 6.00 | 15.27% | 11.67 | 4.00 | 6.00 |
| **6.50** | **20.90%** | **10.79** | **4.25** | **6.50** |
| 7.00 | 27.14% | 10.15 | 4.50 | 7.00 |
| 7.50 | 34.03% | 9.73 | 4.75 | 7.50 |
| 8.00 | 41.10% | 9.40 | 5.00 | 8.00 |

**Go 选择了 6.5**（实际是 `loadFactorNum/loadFactorDen = 13/2 = 6.5`），这是溢出桶比例（~21%）和内存开销（~10.8 字节/条目）的平衡点。

### 2.2 核心常量（第 64-101 行）

```go
const (
    bucketCntBits = abi.MapBucketCountBits  // 每桶 8 个元素（2^3）
    loadFactorDen = 2
    loadFactorNum = loadFactorDen * abi.MapBucketCount * 13 / 16  // 6.5

    // tophash 特殊值
    emptyRest      = 0  // 空，且后续也没有数据了
    emptyOne       = 1  // 空
    evacuatedX     = 2  // 已搬迁到新表的前半部分
    evacuatedY     = 3  // 已搬迁到新表的后半部分
    evacuatedEmpty = 4  // 空，但桶已搬迁
    minTopHash     = 5  // 正常填充的最小 tophash 值

    // flags
    iterator     = 1  // 有迭代器在使用 buckets
    oldIterator  = 2  // 有迭代器在使用 oldbuckets
    hashWriting  = 4  // 有 goroutine 正在写 map
    sameSizeGrow = 8  // 当前扩容是等量扩容
)
```

### 2.3 核心结构

**hmap（第 109-123 行）—— map 的"大脑"：**

```go
type hmap struct {
    count     int           // 元素个数（len() 读这个）
    flags     uint8         // 状态标志
    B         uint8         // 桶数 = 2^B
    noverflow uint16        // 溢出桶的近似计数
    hash0     uint32        // 哈希种子（随机初始化，防哈希碰撞攻击）

    buckets    unsafe.Pointer  // 桶数组
    oldbuckets unsafe.Pointer  // 扩容时的旧桶数组
    nevacuate  uintptr         // 扩容搬迁进度

    extra      *mapextra       // 可选字段
}
```

**mapextra（第 126-140 行）—— 额外字段：**

```go
type mapextra struct {
    overflow    *[]*bmap    // 溢出桶指针（key/elem 无指针时用）
    oldoverflow *[]*bmap    // 旧表的溢出桶指针
    nextOverflow *bmap      // 预分配的下一个空闲溢出桶
}
```

当 key 和 elem 都不包含指针时，桶的 `overflow` 字段不被 GC 扫描，所以需要在 `extra` 中保存引用，防止溢出桶被回收。

**bmap（第 143-153 行）—— 一个桶：**

```go
type bmap struct {
    tophash [8]uint8  // 每个槽的哈希高 8 位
    // 紧随其后（编译器生成，不在源码中）：
    //   8 个 key
    //   8 个 value
    //   1 个 overflow 指针
}
```

**精妙设计**：key 和 value 不是交替存放，而是**先 8 个 key 连续放，再 8 个 value 连续放**。源码注释（第 149-152 行）：

> packing all the keys together and then all the elems together makes the code a bit more complicated than alternating key/elem/key/elem/... but it allows us to eliminate padding which would be needed for, e.g., map[int64]int8.

**hiter（第 158-174 行）—— 迭代器状态：**

```go
type hiter struct {
    key         unsafe.Pointer  // 当前 key
    elem        unsafe.Pointer  // 当前 value
    t           *maptype
    h           *hmap
    buckets     unsafe.Pointer  // 迭代开始时的桶快照
    bptr        *bmap           // 当前桶
    overflow    *[]*bmap
    oldoverflow *[]*bmap
    startBucket uintptr         // 起始桶
    offset      uint8           // 桶内起始偏移
    wrapped     bool            // 是否已绕回
    B           uint8
    i           uint8
    bucket      uintptr
    checkBucket uintptr
}
```

### 2.4 辅助函数

**`tophash`（第 188-194 行）**：

```go
func tophash(hash uintptr) uint8 {
    top := uint8(hash >> (goarch.PtrSize*8 - 8))  // 取高 8 位
    if top < minTopHash {
        top += minTopHash  // 确保不与特殊值冲突
    }
    return top
}
```

**`evacuated`（第 196-199 行）**：

```go
func evacuated(b *bmap) bool {
    h := b.tophash[0]
    return h > emptyOne && h < minTopHash
}
```

**`incrnoverflow`（第 220-237 行）**—— 溢出桶计数：

```go
func (h *hmap) incrnoverflow() {
    if h.B < 16 {
        h.noverflow++
        return
    }
    // 桶很多时用概率采样，避免计数器本身成为瓶颈
    mask := uint32(1)<<(h.B-15) - 1
    if uint32(rand())&mask == 0 {
        h.noverflow++
    }
}
```

**`newoverflow`（第 239-265 行）**—— 分配溢出桶：

```go
func (h *hmap) newoverflow(t *maptype, b *bmap) *bmap {
    var ovf *bmap
    if h.extra != nil && h.extra.nextOverflow != nil {
        // 有预分配的溢出桶，直接用
        ovf = h.extra.nextOverflow
        if ovf.overflow(t) == nil {
            // 还有更多预分配的
            h.extra.nextOverflow = (*bmap)(add(unsafe.Pointer(ovf), uintptr(t.BucketSize)))
        } else {
            // 最后一个预分配的
            ovf.setoverflow(t, nil)
            h.extra.nextOverflow = nil
        }
    } else {
        // 没有预分配的，现场分配
        ovf = (*bmap)(newobject(t.Bucket))
    }
    h.incrnoverflow()
    b.setoverflow(t, ovf)
    return ovf
}
```

### 2.5 创建 map

**`makemap_small`（第 296-300 行）**—— 编译期确定 hint ≤ 8 时使用：

```go
func makemap_small() *hmap {
    h := new(hmap)
    h.hash0 = uint32(rand())  // 随机哈希种子
    return h
}
```

**`makemap`（第 318-351 行）**—— 通用版本：

```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 1. 溢出检查
    mem, overflow := math.MulUintptr(uintptr(hint), t.Bucket.Size_)
    if overflow || mem > maxAlloc { hint = 0 }

    // 2. 初始化 hmap
    if h == nil { h = new(hmap) }
    h.hash0 = uint32(rand())

    // 3. 计算 B（桶数的 log2）
    B := uint8(0)
    for overLoadFactor(hint, B) { B++ }
    h.B = B

    // 4. 分配桶数组
    if h.B != 0 {
        var nextOverflow *bmap
        h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
        if nextOverflow != nil {
            h.extra = new(mapextra)
            h.extra.nextOverflow = nextOverflow
        }
    }
    return h
}
```

**`makeBucketArray`（第 359-402 行）**—— 桶数组分配：

```go
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    base := bucketShift(b)           // 2^b 个桶
    nbuckets := base
    if b >= 4 {
        // 额外分配 2^(b-4) 个溢出桶
        nbuckets += bucketShift(b - 4)
        sz := t.Bucket.Size_ * nbuckets
        up := roundupsize(sz, !t.Bucket.Pointers())
        if up != sz {
            nbuckets = up / t.Bucket.Size_
        }
    }

    if dirtyalloc == nil {
        buckets = newarray(t.Bucket, int(nbuckets))
    } else {
        buckets = dirtyalloc
        // 清零并复用
    }

    if base != nbuckets {
        // 有预分配的溢出桶
        nextOverflow = (*bmap)(add(buckets, base*uintptr(t.BucketSize)))
        last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.BucketSize)))
        last.setoverflow(t, (*bmap)(buckets))  // 哨兵指针
    }
    return buckets, nextOverflow
}
```

### 2.6 查找路径 `mapaccess1`（第 409-468 行）

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 1. 空 map 或 nil map → 返回零值
    if h == nil || h.count == 0 {
        if err := mapKeyError(t, key); err != nil { panic(err) }
        return unsafe.Pointer(&zeroVal[0])
    }

    // 2. 并发写检测（fatal，不可 recover）
    if h.flags&hashWriting != 0 {
        fatal("concurrent map read and map write")
    }

    // 3. 计算哈希
    hash := t.Hasher(key, uintptr(h.hash0))

    // 4. 低 B 位定位桶
    m := bucketMask(h.B)
    b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.BucketSize)))

    // 5. 扩容期间：如果旧桶未搬迁，用旧桶
    if c := h.oldbuckets; c != nil {
        if !h.sameSizeGrow() {
            m >>= 1  // 旧桶数是新的一半
        }
        oldb := (*bmap)(add(c, (hash&m)*uintptr(t.BucketSize)))
        if !evacuated(oldb) {
            b = oldb  // 旧桶还没搬迁，用旧桶
        }
    }

    // 6. 高 8 位快速过滤 + 遍历桶链
    top := tophash(hash)
bucketloop:
    for ; b != nil; b = b.overflow(t) {
        for i := uintptr(0); i < abi.MapBucketCount; i++ {
            if b.tophash[i] != top {
                if b.tophash[i] == emptyRest {
                    break bucketloop  // 优化：后面不可能有数据了
                }
                continue
            }
            // 7. tophash 匹配 → 真正比较 key
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
            if t.IndirectKey() {
                k = *((*unsafe.Pointer)(k))
            }
            if t.Key.Equal(key, k) {
                // 8. 找到 → 返回 value 指针
                e := add(unsafe.Pointer(b), dataOffset+
                    abi.MapBucketCount*uintptr(t.KeySize)+
                    i*uintptr(t.ValueSize))
                if t.IndirectElem() {
                    e = *((*unsafe.Pointer)(e))
                }
                return e
            }
        }
    }
    return unsafe.Pointer(&zeroVal[0])
}
```

`mapaccess2`（第 479-538 行）是带 `ok` 返回值的版本，逻辑完全相同，只是返回 `(unsafe.Pointer, bool)`。

### 2.7 写入路径 `mapassign`（第 615-730 行）

```go
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // 1. nil map 检查
    if h == nil { panic(plainError("assignment to entry in nil map")) }

    // 2. 并发写检测
    if h.flags&hashWriting != 0 { fatal("concurrent map writes") }

    // 3. 计算哈希
    hash := t.Hasher(key, uintptr(h.hash0))

    // 4. 设置写标志（在 hasher 之后，因为 hasher 可能 panic）
    h.flags ^= hashWriting

    // 5. 懒初始化桶
    if h.buckets == nil {
        h.buckets = newobject(t.Bucket)
    }

again:
    // 6. 定位桶
    bucket := hash & bucketMask(h.B)

    // 7. 扩容中？先搬迁
    if h.growing() {
        growWork(t, h, bucket)
    }

    b := (*bmap)(add(h.buckets, bucket*uintptr(t.BucketSize)))
    top := tophash(hash)

    // 8. 查找已有 key 或空槽
    var inserti *uint8
    var insertk unsafe.Pointer
    var elem unsafe.Pointer
bucketloop:
    for {
        for i := uintptr(0); i < abi.MapBucketCount; i++ {
            if b.tophash[i] != top {
                if isEmpty(b.tophash[i]) && inserti == nil {
                    // 记录第一个空槽
                    inserti = &b.tophash[i]
                    insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
                    elem = add(unsafe.Pointer(b), dataOffset+
                        abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
                }
                if b.tophash[i] == emptyRest {
                    break bucketloop
                }
                continue
            }
            k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.KeySize))
            if t.IndirectKey() { k = *((*unsafe.Pointer)(k)) }
            if !t.Key.Equal(key, k) { continue }

            // 找到已有 key → 更新
            if t.NeedKeyUpdate() {
                typedmemmove(t.Key, k, key)
            }
            elem = add(unsafe.Pointer(b), dataOffset+
                abi.MapBucketCount*uintptr(t.KeySize)+i*uintptr(t.ValueSize))
            goto done
        }
        ovf := b.overflow(t)
        if ovf == nil { break }
        b = ovf
    }

    // 9. 没找到 → 需要插入

    // 触发扩容判断
    if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        hashGrow(t, h)
        goto again  // 扩容后重试
    }

    // 没有空槽 → 分配新溢出桶
    if inserti == nil {
        newb := h.newoverflow(t, b)
        inserti = &newb.tophash[0]
        insertk = add(unsafe.Pointer(newb), dataOffset)
        elem = add(insertk, abi.MapBucketCount*uintptr(t.KeySize))
    }

    // 10. 写入新 key/elem
    if t.IndirectKey() {
        kmem := newobject(t.Key)
        *(*unsafe.Pointer)(insertk) = kmem
        insertk = kmem
    }
    if t.IndirectElem() {
        vmem := newobject(t.Elem)
        *(*unsafe.Pointer)(elem) = vmem
    }
    typedmemmove(t.Key, insertk, key)
    *inserti = top
    h.count++

done:
    // 11. 清除写标志
    if h.flags&hashWriting == 0 { fatal("concurrent map writes") }
    h.flags &^= hashWriting
    if t.IndirectElem() {
        elem = *((*unsafe.Pointer)(elem))
    }
    return elem
}
```

### 2.8 删除路径 `mapdelete`（第 741-855 行）

删除后的 `emptyRest` 传播是亮点（第 810-839 行）：

```go
// 删除后，如果当前槽位后面全是 emptyOne，批量改为 emptyRest
b.tophash[i] = emptyOne
if i == abi.MapBucketCount-1 {
    if b.overflow(t) != nil && b.overflow(t).tophash[0] != emptyRest {
        goto notLast
    }
} else {
    if b.tophash[i+1] != emptyRest {
        goto notLast
    }
}
for {
    b.tophash[i] = emptyRest
    if i == 0 {
        if b == bOrig { break }
        // 找到前一个桶，从它的最后一个元素继续
        for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {}
        i = abi.MapBucketCount - 1
    } else {
        i--
    }
    if b.tophash[i] != emptyOne { break }
}
```

这是一个**回溯优化**：删除后向前传播 `emptyRest`，让后续查找可以提前终止。

另外，当 `count == 0` 时重置哈希种子（第 844-846 行）：

```go
if h.count == 0 {
    h.hash0 = uint32(rand())  // 防止攻击者反复触发哈希碰撞
}
```

### 2.9 渐进式扩容

**`hashGrow`（第 1139-1180 行）**—— 触发扩容：

```go
func hashGrow(t *maptype, h *hmap) {
    bigger := uint8(1)
    if !overLoadFactor(h.count+1, h.B) {
        bigger = 0           // 溢出桶太多但装载因子不高 → 等量扩容
        h.flags |= sameSizeGrow
    }
    oldbuckets := h.buckets
    newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

    // 原子切换
    h.B += bigger
    h.oldbuckets = oldbuckets
    h.buckets = newbuckets
    h.nevacuate = 0
    h.noverflow = 0

    // 溢出桶指针迁移
    if h.extra != nil && h.extra.overflow != nil {
        h.extra.oldoverflow = h.extra.overflow
        h.extra.overflow = nil
    }
}
```

**`growWork`（第 1231-1240 行）**—— 每次写操作顺手搬迁：

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
    evacuate(t, h, bucket&h.oldbucketmask())  // 搬迁当前操作涉及的旧桶
    if h.growing() {
        evacuate(t, h, h.nevacuate)            // 再搬迁一个待搬迁的桶
    }
}
```

**`evacuate`（第 1255-1367 行）**—— 搬迁一个旧桶：

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
    b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
    newbit := h.noldbuckets()
    if !evacuated(b) {
        var xy [2]evacDst  // X 和 Y 两个目标
        x := &xy[0]
        x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.BucketSize)))

        if !h.sameSizeGrow() {
            // 翻倍扩容：Y 桶 = 旧桶号 + 旧桶总数
            y := &xy[1]
            y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.BucketSize)))
        }

        for ; b != nil; b = b.overflow(t) {
            for i := 0; i < abi.MapBucketCount; i++ {
                // ...
                var useY uint8
                if !h.sameSizeGrow() {
                    hash := t.Hasher(k2, uintptr(h.hash0))
                    // NaN 特殊处理
                    if h.flags&iterator != 0 && !t.ReflexiveKey() && !t.Key.Equal(k2, k2) {
                        useY = top & 1  // 用 tophash 低位决定
                        top = tophash(hash)
                    } else {
                        if hash&newbit != 0 {
                            useY = 1  // 新桶号的第 B 位为 1 → 去 Y
                        }
                    }
                }
                b.tophash[i] = evacuatedX + useY
                // 拷贝到目标桶...
            }
        }
    }
    // 推进搬迁进度
    if oldbucket == h.nevacuate {
        advanceEvacuationMark(h, t, newbit)
    }
}
```

**`advanceEvacuationMark`（第 1369-1391 行）**：

```go
func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
    h.nevacuate++
    stop := h.nevacuate + 1024
    if stop > newbit { stop = newbit }
    // 跳过已搬迁的桶
    for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
        h.nevacuate++
    }
    if h.nevacuate == newbit {
        // 全部搬迁完成！
        h.oldbuckets = nil
        if h.extra != nil {
            h.extra.oldoverflow = nil
        }
        h.flags &^= sameSizeGrow
    }
}
```

### 2.10 迭代 `mapiterinit` + `mapiternext`

**`mapiterinit`（第 877-921 行）**：

```go
func mapiterinit(t *maptype, h *hmap, it *hiter) {
    it.t = t
    if h == nil || h.count == 0 { return }
    it.h = h
    it.B = h.B
    it.buckets = h.buckets

    // 随机起点
    r := uintptr(rand())
    it.startBucket = r & bucketMask(h.B)
    it.offset = uint8(r >> h.B & (abi.MapBucketCount - 1))

    // 标记有迭代器
    atomic.Or8(&h.flags, iterator|oldIterator)

    mapiternext(it)
}
```

**`mapiternext`（第 937-1061 行）**—— 迭代的核心逻辑：

关键处理——**扩容期间的迭代**（第 960-983 行）：

```go
if h.growing() && it.B == h.B {
    // 迭代开始时在扩容中，且扩容未完成
    oldbucket := bucket & it.h.oldbucketmask()
    b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.BucketSize)))
    if !evacuated(b) {
        checkBucket = bucket  // 需要检查这个元素是否属于当前新桶
    } else {
        b = (*bmap)(add(it.buckets, bucket*uintptr(t.BucketSize)))
        checkBucket = noCheck
    }
}
```

以及对 NaN 的特殊处理（第 1011-1022 行）：

```go
if t.ReflexiveKey() || t.Key.Equal(k, k) {
    hash := t.Hasher(k, uintptr(h.hash0))
    if hash&bucketMask(it.B) != checkBucket {
        continue  // 这个 key 不属于当前新桶，跳过
    }
} else {
    // NaN：用 tophash 低位决定
    if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
        continue
    }
}
```

### 2.11 `mapclear`（第 1075-1137 行）—— 清空 map

```go
func mapclear(t *maptype, h *hmap) {
    // 标记所有桶为 emptyRest
    markBucketsEmpty := func(bucket unsafe.Pointer, mask uintptr) {
        for i := uintptr(0); i <= mask; i++ {
            b := (*bmap)(add(bucket, i*uintptr(t.BucketSize)))
            for ; b != nil; b = b.overflow(t) {
                for i := uintptr(0); i < abi.MapBucketCount; i++ {
                    b.tophash[i] = emptyRest
                }
            }
        }
    }
    markBucketsEmpty(h.buckets, bucketMask(h.B))
    if oldBuckets := h.oldbuckets; oldBuckets != nil {
        markBucketsEmpty(oldBuckets, h.oldbucketmask())
    }

    h.oldbuckets = nil
    h.nevacuate = 0
    h.noverflow = 0
    h.count = 0
    h.hash0 = uint32(rand())  // 重置哈希种子

    // 复用桶数组
    _, nextOverflow := makeBucketArray(t, h.B, h.buckets)
    if nextOverflow != nil {
        h.extra.nextOverflow = nextOverflow
    }
}
```

### 2.12 `mapclone2`（第 1675-1782 行）—— `maps.Clone` 的实现

```go
func mapclone2(t *maptype, src *hmap) *hmap {
    dst := makemap(t, hint, nil)
    dst.hash0 = src.hash0

    // 快速路径：小 map 且 key 不可变
    if src.B == 0 && !(t.IndirectKey() && t.NeedKeyUpdate()) && !t.IndirectElem() {
        dst.buckets = newobject(t.Bucket)
        dst.count = src.count
        typedmemmove(t.Bucket, dst.buckets, src.buckets)
        return dst
    }

    // 逐桶拷贝
    for i := 0; i < dstArraySize; i++ {
        for j := 0; j < srcArraySize; j += dstArraySize {
            srcBmap := ...
            for srcBmap != nil {
                dstBmap, pos = moveToBmap(t, dst, dstBmap, pos, srcBmap)
                srcBmap = srcBmap.overflow(t)
            }
        }
    }

    // 处理未搬迁的旧桶
    if src.oldbuckets != nil {
        // ...
    }
    return dst
}
```

---

## 3. chan.go — channel 的完整实现

### 3.1 文件头部的不变量注释（第 9-18 行）

```
// Invariants:
//  At least one of c.sendq and c.recvq is empty,
//  except for the case of an unbuffered channel with a single goroutine
//  blocked on it for both sending and receiving using a select statement.
//
// For buffered channels, also:
//  c.qcount > 0 implies that c.recvq is empty.
//  c.qcount < c.dataqsiz implies that c.sendq is empty.
```

### 3.2 核心结构（第 33-53 行）

```go
type hchan struct {
    qcount   uint           // 缓冲区中元素数
    dataqsiz uint           // 缓冲区容量
    buf      unsafe.Pointer // 环形缓冲区
    elemsize uint16
    closed   uint32
    timer    *timer          // time.NewTimer 创建的 channel 有这个
    elemtype *_type
    sendx    uint           // 发送写入下标
    recvx    uint           // 接收读取下标
    recvq    waitq          // 阻塞的接收者队列
    sendq    waitq          // 阻塞的发送者队列
    lock     mutex           // 所有操作共用一把锁
}
```

**`waitq`（第 55-58 行）**：双向链表，连接所有阻塞在该 channel 上的 `sudog`。

### 3.3 `makechan`（第 73-120 行）—— 创建 channel

三种分配策略：

```go
switch {
case mem == 0:
    // 无缓冲或元素大小为 0：只分配 hchan
    c = (*hchan)(mallocgc(hchanSize, nil, true))
    c.buf = c.raceaddr()
case !elem.Pointers():
    // 元素无指针：hchan 和 buf 合并分配（减少碎片）
    c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    c.buf = add(unsafe.Pointer(c), hchanSize)
default:
    // 元素有指针：分别分配（GC 需要扫描 buf）
    c = new(hchan)
    c.buf = mallocgc(mem, elem, true)
}
```

### 3.4 `full` 和 `empty` —— 快速状态检查

**`full`（第 141-150 行）**：

```go
func full(c *hchan) bool {
    if c.dataqsiz == 0 {
        return c.recvq.first == nil  // 无缓冲：没有接收者就是满的
    }
    return c.qcount == c.dataqsiz    // 有缓冲：元素数等于容量
}
```

**`empty`（第 472-483 行）**：

```go
func empty(c *hchan) bool {
    if c.dataqsiz == 0 {
        return atomic.Loadp(unsafe.Pointer(&c.sendq.first)) == nil
    }
    if c.timer != nil {
        c.timer.maybeRunChan()  // timer channel 特殊处理
    }
    return atomic.Loaduint(&c.qcount) == 0
}
```

### 3.5 `chansend`（第 171-297 行）—— 发送的完整路径

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 1. nil channel → 永久阻塞
    if c == nil {
        if !block { return false }
        gopark(nil, nil, waitReasonChanSendNilChan, traceBlockForever, 2)
    }

    // 2. 快速路径（不加锁）：非阻塞 + 未关闭 + 满 → return false
    if !block && c.closed == 0 && full(c) {
        return false
    }

    // 3. 加锁
    lock(&c.lock)

    // 4. 关闭检查
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }

    // 5. 有等待的接收者？直接传递
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // 6. 缓冲区有空间？放入缓冲区
    if c.qcount < c.dataqsiz {
        qp := chanbuf(c, c.sendx)
        typedmemmove(c.elemtype, qp, ep)
        c.sendx++
        if c.sendx == c.dataqsiz { c.sendx = 0 }  // 环形
        c.qcount++
        unlock(&c.lock)
        return true
    }

    // 7. 都不行 → 阻塞
    if !block {
        unlock(&c.lock)
        return false
    }

    gp := getg()
    mysg := acquireSudog()
    mysg.elem = ep
    mysg.g = gp
    mysg.c = c
    c.sendq.enqueue(mysg)

    gp.parkingOnChan.Store(true)  // 告知栈收缩
    gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)

    // 被唤醒
    closed := !mysg.success
    releaseSudog(mysg)
    if closed {
        panic(plainError("send on closed channel"))
    }
    return true
}
```

### 3.6 `send`（第 305-334 行）—— 直接传递

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if sg.elem != nil {
        sendDirect(c.elemtype, sg, ep)  // 直接拷贝到接收者的栈上！
        sg.elem = nil
    }
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    if sg.releasetime != 0 {
        sg.releasetime = cputicks()
    }
    goready(gp, skip+1)  // 唤醒接收者
}
```

### 3.7 `sendDirect`（第 375-386 行）—— 跨栈拷贝

```go
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.Size_)
    memmove(dst, src, t.Size_)
}
```

源码注释（第 365-373 行）解释了为什么需要特殊处理：

> Sends and receives on unbuffered or empty-buffered channels are the only operations where one running goroutine writes to the stack of another running goroutine. The GC assumes that stack writes only happen when the goroutine is running and are only done by that goroutine.

**这是 Go 中唯一一个"一个 goroutine 写另一个 goroutine 栈"的场景。**

### 3.8 `chanrecv`（第 504-658 行）—— 接收的完整路径

与 `chansend` 对称，但有额外逻辑：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 1. nil channel
    if c == nil {
        if !block { return }
        gopark(nil, nil, waitReasonChanReceiveNilChan, traceBlockForever, 2)
    }

    // 2. timer channel 特殊处理
    if c.timer != nil {
        c.timer.maybeRunChan()
    }

    // 3. 快速路径（不加锁）
    if !block && empty(c) {
        if atomic.Load(&c.closed) == 0 {
            return  // 未关闭且空 → return false
        }
        if empty(c) {
            // 已关闭且空 → return (true, false)
            if ep != nil { typedmemclr(c.elemtype, ep) }
            return true, false
        }
    }

    // 4. 加锁
    lock(&c.lock)

    // 5. 已关闭？
    if c.closed != 0 {
        if c.qcount == 0 {
            unlock(&c.lock)
            if ep != nil { typedmemclr(c.elemtype, ep) }
            return true, false  // 已关闭且空
        }
        // 已关闭但缓冲区还有数据 → 继续从缓冲区取
    } else {
        // 6. 有等待的发送者？
        if sg := c.sendq.dequeue(); sg != nil {
            recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
            return true, true
        }
    }

    // 7. 从缓冲区取
    if c.qcount > 0 {
        qp := chanbuf(c, c.recvx)
        if ep != nil { typedmemmove(c.elemtype, ep, qp) }
        typedmemclr(c.elemtype, qp)
        c.recvx++
        if c.recvx == c.dataqsiz { c.recvx = 0 }
        c.qcount--
        unlock(&c.lock)
        return true, true
    }

    // 8. 阻塞
    // ... (类似 chansend 的阻塞逻辑)
}
```

### 3.9 `recv`（第 674-714 行）—— 从发送者直接接收

```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
    if c.dataqsiz == 0 {
        // 无缓冲：直接从发送者拷贝
        if ep != nil {
            recvDirect(c.elemtype, sg, ep)
        }
    } else {
        // 有缓冲（满的情况）：
        // 1. 从缓冲区头部取数据给接收者
        // 2. 把发送者的数据放入缓冲区尾部
        // 因为缓冲区满，头部和尾部是同一个位置！
        qp := chanbuf(c, c.recvx)
        if ep != nil { typedmemmove(c.elemtype, ep, qp) }
        typedmemmove(c.elemtype, qp, sg.elem)
        c.recvx++
        if c.recvx == c.dataqsiz { c.recvx = 0 }
        c.sendx = c.recvx
    }
    sg.elem = nil
    gp := sg.g
    unlockf()
    gp.param = unsafe.Pointer(sg)
    sg.success = true
    goready(gp, skip+1)
}
```

### 3.10 `closechan`（第 397-466 行）—— 关闭 channel

```go
func closechan(c *hchan) {
    if c == nil { panic(plainError("close of nil channel")) }

    lock(&c.lock)
    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("close of closed channel"))
    }

    c.closed = 1

    var glist gList

    // 唤醒所有接收者（success=false，收到零值）
    for {
        sg := c.recvq.dequeue()
        if sg == nil { break }
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }

    // 唤醒所有发送者（success=false，它们会 panic）
    for {
        sg := c.sendq.dequeue()
        if sg == nil { break }
        sg.elem = nil
        gp := sg.g
        gp.param = unsafe.Pointer(sg)
        sg.success = false
        glist.push(gp)
    }
    unlock(&c.lock)

    // 释放锁后统一唤醒
    for !glist.empty() {
        gp := glist.pop()
        goready(gp, 3)
    }
}
```

### 3.11 `chanparkcommit`（第 716-734 行）—— 阻塞时的栈保护

```go
func chanparkcommit(gp *g, chanLock unsafe.Pointer) bool {
    gp.activeStackChans = true
    gp.parkingOnChan.Store(false)
    unlock((*mutex)(chanLock))
    return true
}
```

源码注释解释了复杂的时序问题：设置 `activeStackChans` 必须在解锁之前，否则栈收缩可能在不安全的窗口期发生。

### 3.12 `dequeue` 的 select 竞争处理（第 854-884 行）

```go
func (q *waitq) dequeue() *sudog {
    for {
        sgp := q.first
        if sgp == nil { return nil }
        y := sgp.next
        if y == nil {
            q.first = nil
            q.last = nil
        } else {
            y.prev = nil
            q.first = y
            sgp.next = nil  // 标记为已移除
        }

        // 如果这个 sudog 属于 select，且已被其他 case 唤醒，跳过
        if sgp.isSelect && !sgp.g.selectDone.CompareAndSwap(0, 1) {
            continue
        }
        return sgp
    }
}
```

### 3.13 reflect 和编译器入口

```go
// 编译器将 select { case c <- v: ... default: ... } 编译为：
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
    return chansend(c, elem, false, getcallerpc())
}

// 编译器将 select { case v, ok = <-c: ... default: ... } 编译为：
func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected, received bool) {
    return chanrecv(c, elem, false)
}
```

### 3.14 timer channel（第 39 行 + 散布各处）

```go
timer    *timer // timer feeding this chan
```

`time.NewTimer`、`time.After` 等创建的 channel 带有 `timer` 字段。接收时会调用 `c.timer.maybeRunChan()`（第 520-522 行），确保 timer 到期后能正确投递。`timerchandrain`（第 341-363 行）清空 timer channel 的缓冲区。

---

## 4. proc.go — GMP 调度器

### 4.1 文件头部的设计注释（第 22-114 行）

这是理解 GMP 调度器的"设计文档"，核心概念：

> G - goroutine.
> M - worker thread, or machine.
> P - processor, a resource that is required to execute Go code.

以及关于**线程停车/唤醒策略**的详细讨论（第 34-114 行），解释了三种被拒绝的方案和最终采用的"spinning"方案：

> We unpark an additional thread when we submit work if:
> 1. There is an idle P, and
> 2. There are no "spinning" worker threads.

### 4.2 全局变量

```go
var (
    m0           m        // 主 M（OS 主线程）
    g0           g        // 主 G（g0，用于调度）
    mcache0      *mcache  // 初始的 mcache
)
```

### 4.3 `main()`（第 147-301 行）—— 程序入口

```go
func main() {
    // 1. 设置栈大小上限
    if goarch.PtrSize == 8 {
        maxstacksize = 1000000000  // 1GB
    } else {
        maxstacksize = 250000000   // 250MB
    }

    // 2. 启动 sysmon 守护线程
    if haveSysmon {
        systemstack(func() { newm(sysmon, nil, -1) })
    }

    // 3. 锁定主线程
    lockOSThread()

    // 4. 运行时初始化
    doInit(runtime_inittasks)

    // 5. 启用 GC
    gcenable()

    // 6. CGO 初始化
    if iscgo {
        startTemplateThread()
        cgocall(_cgo_notify_runtime_init_done, nil)
    }

    // 7. 运行所有包的 init 函数
    for m := &firstmoduledata; m != nil; m = m.next {
        doInit(m.inittasks)
    }

    // 8. 调用用户的 main.main()
    close(main_init_done)
    fn := main_main
    fn()

    // 9. 退出
    exit(0)
}
```

### 4.4 `schedinit`（第 782-880 行）—— 调度器初始化

```go
func schedinit() {
    // 初始化所有锁
    sched.maxmcount = 10000

    // 初始化顺序很重要
    stackinit()
    mallocinit()
    cpuinit(godebug)
    randinit()
    alginit()          // 哈希算法
    mcommoninit(gp.m, -1)  // 把 m0 加入全局链表
    modulesinit()
    typelinksinit()
    itabsinit()
    stkobjinit()

    // ...
    gcinit()

    // 设置 GOMAXPROCS
    procs := ncpu
    if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
        procs = n
    }
    procresize(procs)  // 创建 P 列表
}
```

### 4.5 `gopark`（第 407-425 行）—— goroutine 阻塞

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer,
    reason waitReason, traceReason traceBlockReason, traceskip int) {
    mp := acquirem()
    gp := mp.curg
    status := readgstatus(gp)
    if status != _Grunning && status != _Gscanrunning {
        throw("gopark: bad g status")
    }
    mp.waitlock = lock
    mp.waitunlockf = unlockf
    gp.waitreason = reason
    releasem(mp)
    mcall(park_m)  // 切换到 g0 栈，执行 park_m
}
```

`mcall(park_m)` 的流程：
1. 保存当前 G 的寄存器状态到 `gp.sched`
2. 切换到 g0 栈
3. 调用 `park_m(gp)`
4. `park_m` 调用 `unlockf`（释放锁），然后调用 `schedule()`

### 4.6 `ready`（第 1022-1042 行）—— 唤醒 goroutine

```go
func ready(gp *g, traceskip int, next bool) {
    mp := acquirem()
    casgstatus(gp, _Gwaiting, _Grunnable)
    runqput(mp.p.ptr(), gp, next)
    wakep()
    releasem(mp)
}
```

### 4.7 G 状态机

```go
// 读取 G 状态
func readgstatus(gp *g) uint32 {
    return gp.atomicstatus.Load()
}

// CAS 更新 G 状态（不能跨 Gscan 状态）
func casgstatus(gp *g, oldval, newval uint32) {
    // 循环 CAS，如果在 Gscan 状态则等待
    for i := 0; !gp.atomicstatus.CompareAndSwap(oldval, newval); i++ {
        if i == 0 { nextYield = nanotime() + yieldDelay }
        if nanotime() < nextYield {
            for x := 0; x < 10 && gp.atomicstatus.Load() != oldval; x++ {
                procyield(1)
            }
        } else {
            osyield()
        }
    }
}
```

### 4.8 `stopTheWorldWithSema`（第 1537-1651 行）—— STW 的实现

```go
func stopTheWorldWithSema(reason stwReason) worldStop {
    lock(&sched.lock)
    start := nanotime()
    sched.stopwait = gomaxprocs
    sched.gcwaiting.Store(true)

    // 1. 抢占所有运行中的 P
    preemptall()

    // 2. 停止当前 P
    gp.m.p.ptr().status = _Pgcstop
    sched.stopwait--

    // 3. 回收 Psyscall 的 P
    for _, pp := range allp {
        s := pp.status
        if s == _Psyscall && atomic.Cas(&pp.status, s, _Pgcstop) {
            pp.syscalltick++
            sched.stopwait--
        }
    }

    // 4. 回收空闲的 P
    for {
        pp, _ := pidleget(now)
        if pp == nil { break }
        pp.status = _Pgcstop
        sched.stopwait--
    }

    // 5. 等待剩余 P 自愿停止
    if sched.stopwait > 0 {
        for {
            if notetsleep(&sched.stopnote, 100*1000) { break }
            preemptall()  // 每 100μs 重试
        }
    }

    // 6. 验证所有 P 已停止
    for _, pp := range allp {
        if pp.status != _Pgcstop {
            throw("stopTheWorld: not stopped")
        }
    }

    worldStopped()
    return worldStop{...}
}
```

### 4.9 `startTheWorldWithSema`（第 1659-1726 行）—— 恢复世界

```go
func startTheWorldWithSema(now int64, w worldStop) int64 {
    mp := acquirem()

    // 1. 轮询网络
    if netpollinited() {
        list, delta := netpoll(0)
        injectglist(&list)
    }

    // 2. 调整 P 数量
    lock(&sched.lock)
    procs := gomaxprocs
    if newprocs != 0 {
        procs = newprocs
        newprocs = 0
    }
    p1 := procresize(procs)
    sched.gcwaiting.Store(false)
    unlock(&sched.lock)

    // 3. 唤醒有工作的 P
    for p1 != nil {
        p := p1
        p1 = p1.link.ptr()
        if p.m != 0 {
            mp := p.m.ptr()
            mp.nextp.set(p)
            notewakeup(&mp.park)  // 唤醒已有的 M
        } else {
            newm(nil, p, -1)      // 创建新 M
        }
    }

    // 4. 唤醒额外的 P
    wakep()

    releasem(mp)
    return now
}
```

### 4.10 `sysmon`（第 6043-6188 行）—— 守护线程

```go
func sysmon() {
    lock(&sched.lock)
    sched.nmsys++
    checkdead()
    unlock(&sched.lock)

    lasttrace := int64(0)
    idle := 0
    delay := uint32(0)

    for {
        // 自适应休眠：20μs ~ 10ms
        if idle == 0 {
            delay = 20
        } else if idle > 50 {
            delay *= 2
        }
        if delay > 10*1000 {
            delay = 10 * 1000
        }
        usleep(delay)

        now := nanotime()

        // 1. 深度休眠优化
        if debug.schedtrace <= 0 && (sched.gcwaiting.Load() || sched.npidle.Load() == gomaxprocs) {
            // 所有 P 都空闲时，可以深度休眠
            // ...
        }

        // 2. CGOS yield
        if *cgo_yield != nil {
            asmcgocall(*cgo_yield, nil)
        }

        // 3. 轮询网络（每 10ms）
        if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
            sched.lastpoll.CompareAndSwap(lastpoll, now)
            list, delta := netpoll(0)
            if !list.empty() {
                incidlelocked(-1)
                injectglist(&list)
                incidlelocked(1)
            }
        }

        // 4. 抢占长时间运行的 G 和回收 syscall 中的 P
        if retake(now) != 0 {
            idle = 0
        } else {
            idle++
        }

        // 5. 强制 GC（每 2 分钟）
        if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && forcegc.idle.Load() {
            lock(&forcegc.lock)
            forcegc.idle.Store(false)
            var list gList
            list.push(forcegc.g)
            injectglist(&list)
            unlock(&forcegc.lock)
        }

        // 6. 调度追踪
        if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
            lasttrace = now
            schedtrace(debug.scheddetail > 0)
        }
    }
}
```

### 4.11 `retake`（第 6201-6274 行）—— 抢占实现

```go
func retake(now int64) uint32 {
    for i := 0; i < len(allp); i++ {
        pp := allp[i]
        pd := &pp.sysmontick
        s := pp.status
        sysretake := false

        if s == _Prunning || s == _Psyscall {
            // 检查是否运行超过 10ms
            t := int64(pp.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
            } else if pd.schedwhen+forcePreemptNS <= now {
                preemptone(pp)
                sysretake = true
            }
        }

        if s == _Psyscall {
            // syscall 超过 10ms 且有其他工作 → 回收 P
            if pd.syscallwhen+10*1000*1000 > now {
                continue
            }
            if runqempty(pp) && sched.nmspinning.Load()+sched.npidle.Load() > 0 {
                continue
            }
            if atomic.Cas(&pp.status, s, _Pidle) {
                n++
                pp.syscalltick++
                handoffp(pp)
            }
        }
    }
    return uint32(n)
}
```

### 4.12 `preemptone`（第 6304-6329 行）—— 单 P 抢占

```go
func preemptone(pp *p) bool {
    mp := pp.m.ptr()
    gp := mp.curg

    gp.preempt = true

    // 设置 stackguard0 为 stackPreempt，让栈检查触发抢占
    gp.stackguard0 = stackPreempt

    // 异步抢占：发送 SIGURG 信号
    if preemptMSupported && debug.asyncpreemptoff == 0 {
        pp.preempt = true
        preemptM(mp)
    }

    return true
}
```

### 4.13 本地队列操作

**`runqput`（第 6696-6738 行）**—— 放入本地队列：

```go
func runqput(pp *p, gp *g, next bool) {
    if next {
        // 放入 runnext（最高优先级）
    retryNext:
        oldnext := pp.runnext
        if !pp.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
            goto retryNext
        }
        if oldnext == 0 { return }
        gp = oldnext.ptr()  // 旧的 runnext 被挤到队列
    }

    // 放入环形队列尾部
retry:
    h := atomic.LoadAcq(&pp.runqhead)
    t := pp.runqtail
    if t-h < uint32(len(pp.runq)) {
        pp.runq[t%uint32(len(pp.runq))].set(gp)
        atomic.StoreRel(&pp.runqtail, t+1)
        return
    }
    // 队列满了 → 放全局队列
    if runqputslow(pp, gp, h, t) { return }
    goto retry
}
```

**`runqget`（第 6819-6840 行）**—— 从本地队列取：

```go
func runqget(pp *p) (gp *g, inheritTime bool) {
    // 先检查 runnext
    next := pp.runnext
    if next != 0 && pp.runnext.cas(next, 0) {
        return next.ptr(), true  // 继承时间片
    }

    // 从队列头取
    for {
        h := atomic.LoadAcq(&pp.runqhead)
        t := pp.runqtail
        if t == h { return nil, false }
        gp := pp.runq[h%uint32(len(pp.runq))].ptr()
        if atomic.CasRel(&pp.runqhead, h, h+1) {
            return gp, false  // 新时间片
        }
    }
}
```

**`runqsteal`（第 6940-6957 行）**—— work stealing：

```go
func runqsteal(pp, p2 *p, stealRunNextG bool) *g {
    t := pp.runqtail
    n := runqgrab(p2, &pp.runq, t, stealRunNextG)  // 偷 p2 一半
    if n == 0 { return nil }
    n--
    gp := pp.runq[(t+n)%uint32(len(pp.runq))].ptr()
    atomic.StoreRel(&pp.runqtail, t+n)
    return gp
}
```

### 4.14 `mstart`（第 1753-1801 行）—— M 的入口

```go
func mstart0() {
    gp := getg()
    osStack := gp.stack.lo == 0
    if osStack {
        // 从系统栈初始化栈边界
        size := gp.stack.hi
        gp.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
        gp.stack.lo = gp.stack.hi - size + 1024
    }
    gp.stackguard0 = gp.stack.lo + stackGuard
    gp.stackguard1 = gp.stackguard0
    mstart1()
    mexit(osStack)
}

func mstart1() {
    gp := getg()
    // 设置 g0.sched 用于 mcall 返回
    gp.sched.g = guintptr(unsafe.Pointer(gp))
    gp.sched.pc = getcallerpc()
    gp.sched.sp = getcallersp()

    asminit()
    minit()

    if gp.m == &m0 {
        mstartm0()  // m0 特殊初始化
    }

    if fn := gp.m.mstartfn; fn != nil { fn() }

    if gp.m != &m0 {
        acquirep(gp.m.nextp.ptr())  // 获取 P
    }
    schedule()  // 进入调度循环
}
```

### 4.15 `acquireSudog` / `releaseSudog`（第 450-535 行）—— sudog 缓存

```go
func acquireSudog() *sudog {
    mp := acquirem()
    pp := mp.p.ptr()
    if len(pp.sudogcache) == 0 {
        lock(&sched.sudoglock)
        // 从全局缓存批量获取
        for len(pp.sudogcache) < cap(pp.sudogcache)/2 && sched.sudogcache != nil {
            s := sched.sudogcache
            sched.sudogcache = s.next
            s.next = nil
            pp.sudogcache = append(pp.sudogcache, s)
        }
        unlock(&sched.sudoglock)
        // 全局也没有 → 分配新的
        if len(pp.sudogcache) == 0 {
            pp.sudogcache = append(pp.sudogcache, new(sudog))
        }
    }
    // 从本地缓存取
    n := len(pp.sudogcache)
    s := pp.sudogcache[n-1]
    pp.sudogcache[n-1] = nil
    pp.sudogcache = pp.sudogcache[:n-1]
    releasem(mp)
    return s
}
```

### 4.16 随机化调度辅助（第 7180-7234 行）

```go
type randomOrder struct {
    count    uint32
    coprimes []uint32
}

type randomEnum struct {
    i     uint32
    count uint32
    pos   uint32
    inc   uint32
}

func (ord *randomOrder) start(i uint32) randomEnum {
    return randomEnum{
        count: ord.count,
        pos:   i % ord.count,
        inc:   ord.coprimes[i/ord.count%uint32(len(ord.coprimes))],
    }
}
```

基于**互质数**的伪随机遍历：如果 X 和 GOMAXPROCS 互质，则 `(i + X) % GOMAXPROCS` 遍历所有 P 且不重复。

### 4.17 `procPin` / `procUnpin`（第 7086-7109 行）

```go
func procPin() int {
    gp := getg()
    mp := gp.m
    mp.locks++
    return int(mp.p.ptr().id)
}

func procUnpin() {
    gp := getg()
    gp.m.locks--
}
```

`sync.Pool` 等使用这对函数"钉住"当前 P，防止在操作过程中被抢占。

---

## 5. mgc.go — GC 主控逻辑

### 5.1 文件头部的设计注释（第 1-128 行）

这是 Go GC 的"设计文档"，描述了完整的算法：

> The GC runs concurrently with mutator threads, is type accurate (aka precise), allows multiple GC thread to run in parallel. It is a concurrent mark and sweep that uses a write barrier. It is non-generational and non-compacting.

**完整的 5 个阶段**：

1. **Sweep Termination**
   - a. Stop the world
   - b. Sweep any unswept spans

2. **Mark Phase**
   - a. STW：设置 `_GCmark`，启用写屏障，入队根标记任务
   - b. Start world：并发标记（mark worker + mutator assist）
   - c. 扫描栈、全局变量、堆指针
   - d. 灰色对象出队 → 扫描 → 标黑
   - e. 分布式终止检测

3. **Mark Termination**
   - a. Stop the world
   - b. 设置 `_GCmarktermination`
   - c. 刷新 mcaches

4. **Sweep Phase**
   - a. 设置 `_GCoff`，禁用写屏障
   - b. Start world：并发清扫
   - c. 懒清扫 + 后台清扫

5. **回到第 1 步**

源码还解释了**并发清扫**的机制（第 84-110 行）：

> The heap is swept span-by-span both lazily (when a goroutine needs another span) and concurrently in a background goroutine.

以及 **GC 速率**（第 112-118 行）：

> Next GC is after we've allocated an extra amount of memory proportional to the amount already in use. The proportion is controlled by GOGC environment variable (100 by default).

还有 **Oblets** 的概念（第 120-128 行）—— 大对象扫描的优化：

> the garbage collector breaks up scan jobs for objects larger than maxObletBytes into "oblets" of at most maxObletBytes.

### 5.2 常量和变量

```go
const (
    concurrentSweep = true  // 并发清扫开关
    sweepMinHeapDistance = 1024 * 1024  // 并发清扫的最小堆距离
)

var gcphase uint32  // GC 阶段

var writeBarrier struct {
    enabled bool    // 编译器在写屏障前检查这个
    pad     [3]byte // 32 位对齐
    alignme uint64  // 保证 64 位对齐
}

var gcBlackenEnabled uint32  // 是否允许标黑

const (
    _GCoff             = iota // GC 未运行
    _GCmark                   // 标记中，写屏障启用
    _GCmarktermination        // 标记终止
)
```

### 5.3 `gcinit`（第 177-195 行）—— GC 初始化

```go
func gcinit() {
    sweep.active.state.Store(sweepDrainedMask)  // 第一轮不需要清扫
    gcController.init(readGOGC(), readGOMEMLIMIT())  // 初始化 pacer
    work.startSema = 1
    work.markDoneSema = 1
}
```

### 5.4 `gcenable`（第 201-209 行）—— 启用 GC

```go
func gcenable() {
    c := make(chan int, 2)
    go bgsweep(c)      // 后台清扫 goroutine
    go bgscavenge(c)   // 后台回收 goroutine
    <-c
    <-c
    memstats.enablegc = true
}
```

### 5.5 `GC()`（第 467-537 行）—— 用户主动触发 GC

```go
func GC() {
    // 1. 等待当前标记完成
    n := work.cycles.Load()
    gcWaitOnMark(n)

    // 2. 触发新一轮 GC
    gcStart(gcTrigger{kind: gcTriggerCycle, n: n + 1})

    // 3. 等待新标记完成
    gcWaitOnMark(n + 1)

    // 4. 完成清扫
    for work.cycles.Load() == n+1 && sweepone() != ^uintptr(0) {
        Gosched()
    }

    // 5. 等待清扫完成
    for work.cycles.Load() == n+1 && !isSweepDone() {
        Gosched()
    }

    // 6. 发布堆 profile
    mProf_PostSweep()
}
```

### 5.6 GC 触发条件（第 572-621 行）

```go
type gcTrigger struct {
    kind gcTriggerKind
    now  int64   // gcTriggerTime
    n    uint32  // gcTriggerCycle
}

const (
    gcTriggerHeap  gcTriggerKind = iota  // 堆大小达到阈值
    gcTriggerTime                         // 距上次 GC 超过 2 分钟
    gcTriggerCycle                        // 指定轮次
)

func (t gcTrigger) test() bool {
    if !memstats.enablegc || panicking.Load() != 0 || gcphase != _GCoff {
        return false
    }
    switch t.kind {
    case gcTriggerHeap:
        trigger, _ := gcController.trigger()
        return gcController.heapLive.Load() >= trigger
    case gcTriggerTime:
        if gcController.gcPercent.Load() < 0 { return false }
        lastgc := int64(atomic.Load64(&memstats.last_gc_nanotime))
        return lastgc != 0 && t.now-lastgc > forcegcperiod
    case gcTriggerCycle:
        return int32(t.n-work.cycles.Load()) > 0
    }
    return true
}
```

### 5.7 `gcStart`（第 629-814 行）—— GC 启动

```go
func gcStart(trigger gcTrigger) {
    // 安全检查
    mp := acquirem()
    if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
        releasem(mp)
        return
    }
    releasem(mp)

    // 完成未清扫的 span
    for trigger.test() && sweepone() != ^uintptr(0) {}

    // 获取信号量
    semacquire(&work.startSema)
    if !trigger.test() {
        semrelease(&work.startSema)
        return
    }

    semacquire(&gcsema)
    semacquire(&worldsema)

    // STW
    stw = stopTheWorldWithSema(stwGCSweepTerm)

    // 完成清扫
    finishsweep_m()
    clearpools()

    work.cycles.Add(1)

    // 启动标记 worker
    gcBgMarkStartWorkers()

    // 重置标记状态
    gcResetMarkState()

    // 准备根标记
    gcMarkRootPrepare()

    // 标记 tiny 分配
    gcMarkTinyAllocs()

    // 设置标记阶段，启用写屏障
    setGCPhase(_GCmark)

    // 启用 mutator assist
    atomic.Store(&gcBlackenEnabled, 1)

    // 恢复世界
    startTheWorldWithSema(0, stw)
    semrelease(&worldsema)
    semrelease(&work.startSema)
}
```

### 5.8 `gcMarkDone`（第 858-1004 行）—— 标记完成检测

```go
func gcMarkDone() {
    semacquire(&work.markDoneSema)

top:
    // 检查：还有工作吗？
    if !(gcphase == _GCmark && work.nwait == work.nproc && !gcMarkWorkAvailable(nil)) {
        semrelease(&work.markDoneSema)
        return
    }

    // "Ragged barrier"：刷新所有 P 的本地缓冲
    semacquire(&worldsema)
    work.strongFromWeak.block = true

    gcMarkDoneFlushed = 0
    forEachP(waitReasonGCMarkTermination, func(pp *p) {
        wbBufFlush1(pp)       // 刷新写屏障缓冲
        pp.gcw.dispose()      // 刷新 gcWork
        if pp.gcw.flushedWork {
            atomic.Xadd(&gcMarkDoneFlushed, 1)
            pp.gcw.flushedWork = false
        }
    })

    if gcMarkDoneFlushed != 0 {
        // 发现了新工作！重新检查
        semrelease(&worldsema)
        goto top
    }

    // 真的没有工作了 → STW
    stw = stopTheWorldWithSema(stwGCMarkTerm)

    // 再次检查是否有遗漏
    restart := false
    systemstack(func() {
        for _, p := range allp {
            wbBufFlush1(p)
            if !p.gcw.empty() {
                restart = true
                break
            }
        }
    })
    if restart {
        // 有遗漏 → 恢复世界，重新标记
        startTheWorldWithSema(0, stw)
        semrelease(&worldsema)
        goto top
    }

    // 确认标记完成
    atomic.Store(&gcBlackenEnabled, 0)
    gcWakeAllAssists()
    semrelease(&work.markDoneSema)

    // 进入 mark termination
    gcMarkTermination(stw)
}
```

### 5.9 `gcMarkTermination`（第 1008-1299 行）—— 标记终止

```go
func gcMarkTermination(stw worldStop) {
    setGCPhase(_GCmarktermination)

    work.heap1 = gcController.heapLive.Load()

    // 在 g0 栈上执行 gcMark（栈收缩）
    systemstack(func() {
        gcMark(startTime)
    })

    // 关闭写屏障，开始清扫
    systemstack(func() {
        setGCPhase(_GCoff)
        stwSwept = gcSweep(work.mode)
    })

    // 更新统计
    memstats.pause_ns[memstats.numgc%uint32(len(memstats.pause_ns))] = uint64(work.pauseNS)
    memstats.pause_total_ns += uint64(work.pauseNS)

    // 更新 pacer
    systemstack(gcControllerCommit)

    // 打印 gctrace
    if debug.gctrace > 0 {
        print("gc ", memstats.numgc, " @", ...)
    }

    // 唤醒等待清扫完成的 goroutine
    lock(&work.sweepWaiters.lock)
    memstats.numgc++
    injectglist(&work.sweepWaiters.list)
    unlock(&work.sweepWaiters.lock)

    // 恢复世界
    startTheWorldWithSema(now, stw)

    // 刷新堆 profile
    mProf_Flush()

    // 清理 mcaches
    forEachP(waitReasonFlushProcCaches, func(pp *p) {
        pp.mcache.prepareForSweep()
    })

    semrelease(&worldsema)
    semrelease(&gcsema)
}
```

### 5.10 三种标记 Worker 模式（第 260-287 行）

```go
type gcMarkWorkerMode int

const (
    gcMarkWorkerNotWorker      gcMarkWorkerMode = iota
    gcMarkWorkerDedicatedMode   // 专用模式：P 全力标记，不可抢占
    gcMarkWorkerFractionalMode  // 分数模式：补足 GC 目标 CPU 占用率
    gcMarkWorkerIdleMode        // 空闲模式：P 没事干时标记
)
```

**`pollFractionalWorkerExit`（第 301-314 行）**—— 分数 worker 的退出判断：

```go
func pollFractionalWorkerExit() bool {
    now := nanotime()
    delta := now - gcController.markStartTime
    if delta <= 0 { return true }
    p := getg().m.p.ptr()
    selfTime := p.gcFractionalMarkTime + (now - p.gcMarkWorkerStartTime)
    return float64(selfTime)/float64(delta) > 1.2*gcController.fractionalUtilizationGoal
}
```

### 5.11 `workType`（第 318-462 行）—— GC 工作状态

```go
type workType struct {
    full  lfstack  // 满的 workbuf 链表（无锁）
    empty lfstack  // 空的 workbuf 链表（无锁）

    wbufSpans struct {
        lock  mutex
        free  mSpanList  // 空闲的 workbuf span
        busy  mSpanList  // 使用中的 workbuf span
    }

    bytesMarked uint64  // 本轮标记的字节数

    markrootNext uint32  // 下一个 markroot 任务
    markrootJobs uint32  // markroot 任务总数

    nproc  uint32  // 参与标记的 P 数量
    nwait  uint32  // 等待中的 worker 数量

    // 根扫描信息
    nDataRoots, nBSSRoots, nSpanRoots, nStackRoots int
    baseData, baseBSS, baseSpans, baseStacks, baseEnd uint32
    stackRoots []*g

    // 状态转换信号量
    startSema    uint32  // 保护 _GCoff → _GCmark
    markDoneSema uint32  // 保护 _GCmark → _GCmarktermination

    mode        gcMode
    userForced  bool
    initialHeapLive uint64

    assistQueue struct {
        lock mutex
        q    gQueue
    }

    sweepWaiters struct {
        lock mutex
        list gList
    }

    cycles atomic.Uint32  // 已完成的 GC 轮次

    // 时间统计
    stwprocs, maxprocs int32
    tSweepTerm, tMark, tMarkTerm, tEnd int64
    pauseNS    int64
    heap0, heap1, heap2 uint64
    cpuStats
}
```

### 5.12 `gcBgMarkStartWorkers`（第 1301-1344 行）—— 启动标记 worker

```go
func gcBgMarkStartWorkers() {
    if gcBgMarkWorkerCount >= gomaxprocs { return }

    ready := make(chan struct{}, 1)
    for gcBgMarkWorkerCount < gomaxprocs {
        go gcBgMarkWorker(ready)
        <-ready  // 等待每个 worker 就绪
        gcBgMarkWorkerCount++
    }
}
```

源码注释（第 1335-1341 行）解释了为什么要逐个启动而不是批量：

> By running one goroutine at a time, we can take advantage of runnext to bounce back and forth between workers and this goroutine. In an overloaded application, this can reduce GC start latency.

### 5.13 `gcBgMarkWorker`（第 1377-1551 行）—— 后台标记 worker

```go
func gcBgMarkWorker(ready chan struct{}) {
    gp := getg()
    node := new(gcBgMarkWorkerNode)
    node.gp.set(gp)

    ready <- struct{}{}

    for {
        // 1. 休眠，等待被 findRunnableGCWorker 唤醒
        gopark(func(g *g, nodep unsafe.Pointer) bool {
            node := (*gcBgMarkWorkerNode)(nodep)
            if mp := node.m.ptr(); mp != nil {
                releasem(mp)
            }
            gcBgMarkWorkerPool.push(&node.node)  // 放回 worker 池
            return true
        }, unsafe.Pointer(node), waitReasonGCWorkerIdle, traceBlockSystemGoroutine, 0)

        // 2. 被唤醒，开始标记
        node.m.set(acquirem())
        pp := gp.m.p.ptr()

        startTime := nanotime()
        decnwait := atomic.Xadd(&work.nwait, -1)

        // 3. 根据模式执行标记
        systemstack(func() {
            casGToWaitingForGC(gp, _Grunning, waitReasonGCWorkerActive)
            switch pp.gcMarkWorkerMode {
            case gcMarkWorkerDedicatedMode:
                gcDrainMarkWorkerDedicated(&pp.gcw, true)
                if gp.preempt {
                    // 被抢占 → 清空本地队列到全局
                    if drainQ, n := runqdrain(pp); n > 0 {
                        lock(&sched.lock)
                        globrunqputbatch(&drainQ, int32(n))
                        unlock(&sched.lock)
                    }
                }
                gcDrainMarkWorkerDedicated(&pp.gcw, false)  // 继续标记
            case gcMarkWorkerFractionalMode:
                gcDrainMarkWorkerFractional(&pp.gcw)
            case gcMarkWorkerIdleMode:
                gcDrainMarkWorkerIdle(&pp.gcw)
            }
            casgstatus(gp, _Gwaiting, _Grunning)
        })

        // 4. 统计时间
        now := nanotime()
        duration := now - startTime
        gcController.markWorkerStop(pp.gcMarkWorkerMode, duration)

        // 5. 检查是否所有 worker 都完成
        incnwait := atomic.Xadd(&work.nwait, +1)
        pp.gcMarkWorkerMode = gcMarkWorkerNotWorker

        if incnwait == work.nproc && !gcMarkWorkAvailable(nil) {
            releasem(node.m.ptr())
            node.m.set(nil)
            gcMarkDone()  // 触发标记完成检测
        }
    }
}
```

### 5.14 `gcMark`（第 1572-1655 行）—— 标记终止时的清理

```go
func gcMark(startTime int64) {
    if gcphase != _GCmarktermination {
        throw("in gcMark expecting to see gcphase as _GCmarktermination")
    }

    // 检查没有剩余标记工作
    if work.full != 0 || work.markrootNext < work.markrootJobs {
        panic("non-empty mark queue after concurrent mark")
    }

    // 释放栈快照
    work.stackRoots = nil

    // 清理所有 P 的缓冲
    for _, p := range allp {
        p.wbBuf.reset()
        gcw := &p.gcw
        if !gcw.empty() {
            throw("P has cached GC work at end of mark termination")
        }
        gcw.dispose()
    }

    // 重置控制器
    gcController.resetLive(work.bytesMarked)
}
```

### 5.15 `gcSweep`（第 1665-1716 行）—— 清扫

```go
func gcSweep(mode gcMode) bool {
    lock(&mheap_.lock)
    mheap_.sweepgen += 2
    sweep.active.reset()
    mheap_.pagesSwept.Store(0)
    mheap_.sweepArenas = mheap_.allArenas
    mheap_.reclaimIndex.Store(0)
    mheap_.reclaimCredit.Store(0)
    unlock(&mheap_.lock)

    sweep.centralIndex.clear()

    if !concurrentSweep || mode == gcForceBlockMode {
        // 同步清扫：一次性扫完
        lock(&mheap_.lock)
        mheap_.sweepPagesPerByte = 0
        unlock(&mheap_.lock)
        for _, pp := range allp {
            pp.mcache.prepareForSweep()
        }
        for sweepone() != ^uintptr(0) {}
        prepareFreeWorkbufs()
        for freeSomeWbufs(false) {}
        mProf_NextCycle()
        mProf_Flush()
        return true
    }

    // 并发清扫：唤醒后台清扫 goroutine
    lock(&sweep.lock)
    if sweep.parked {
        sweep.parked = false
        ready(sweep.g, 0, true)
    }
    unlock(&sweep.lock)
    return false
}
```

### 5.16 `gcResetMarkState`（第 1728-1748 行）

```go
func gcResetMarkState() {
    forEachG(func(gp *g) {
        gp.gcscandone = false
        gp.gcAssistBytes = 0
    })

    // 清除页面标记
    lock(&mheap_.lock)
    arenas := mheap_.allArenas
    unlock(&mheap_.lock)
    for _, ai := range arenas {
        ha := mheap_.arenas[ai.l1()][ai.l2()]
        clear(ha.pageMarks[:])
    }

    work.bytesMarked = 0
    work.initialHeapLive = gcController.heapLive.Load()
}
```

### 5.17 `clearpools`（第 1787-1831 行）—— GC 时清理各种池

```go
func clearpools() {
    // 清理 sync.Pool
    if poolcleanup != nil { poolcleanup() }

    // 清理 boringcrypto 缓存
    for _, p := range boringCaches {
        atomicstorep(p, nil)
    }

    // 清理 unique maps
    if uniqueMapCleanup != nil {
        select {
        case uniqueMapCleanup <- struct{}{}:
        default:
        }
    }

    // 清理中心 sudog 缓存
    lock(&sched.sudoglock)
    var sg, sgnext *sudog
    for sg = sched.sudogcache; sg != nil; sg = sgnext {
        sgnext = sg.next
        sg.next = nil
    }
    sched.sudogcache = nil
    unlock(&sched.sudoglock)

    // 清理中心 defer 缓存
    lock(&sched.deferlock)
    var d, dlink *_defer
    for d = sched.deferpool; d != nil; d = dlink {
        dlink = d.link
        d.link = nil
    }
    sched.deferpool = nil
    unlock(&sched.deferlock)
}
```

---

## 🔗 五个文件的协作关系

```
程序启动
    │
    ▼
proc.go: schedinit()
    │   ├── mallocinit()     ← 内存分配器初始化
    │   ├── gcinit()         ← mgc.go: GC 初始化
    │   └── procresize()     ← 创建 P 列表
    │
    ▼
proc.go: main()
    │   ├── newm(sysmon)     ← 启动 sysmon 守护线程
    │   ├── gcenable()       ← mgc.go: 启动 bgsweep + bgscavenge
    │   └── main_main()      ← 用户代码
    │
    ▼
proc.go: schedule() 循环
    │
    ├── slice.go: makeslice / growslice → mallocgc
    ├── map.go:    makemap / mapassign  → mallocgc
    └── chan.go:   makechan             → mallocgc
    │
    ▼
mallocgc (内存分配)
    │
    ├── 分配时检查 GC 触发条件
    │   └── mgc.go: gcStart()
    │
    ▼
mgc.go: GC 流程
    │
    ├── STW → gcStart (sweep termination)
    ├── 并发标记 (mark phase)
    │   ├── gcBgMarkWorker (per-P worker)
    │   ├── mutator assist (分配时帮忙)
    │   └── gcMarkDone (分布式终止检测)
    ├── STW → gcMarkTermination
    └── 并发清扫 (sweep phase)
        └── bgsweep + 懒清扫
```

---

## 💡 核心设计洞察

### 1. 一切皆值拷贝
- `slice` 是 24 字节描述符的值拷贝
- `map` 是 `*hmap` 指针的值拷贝
- `channel` 是 `*hchan` 指针的值拷贝

### 2. 无锁设计的层次
- **完全无锁**：`runqget`/`runqput`（CAS 操作本地队列）
- **细粒度锁**：`hchan.lock`（per-channel）
- **全局锁**：`sched.lock`（调度器状态）

### 3. 渐进式策略
- **slice 扩容**：2x → 1.25x 平滑过渡
- **map 扩容**：每次写操作搬迁 1-2 个桶
- **GC 标记**：mutator assist + background worker + idle worker

### 4. 并发安全的实现
- **map**：`hashWriting` 标志检测并发写（fatal）
- **channel**：`lock mutex` 保护所有状态
- **GC**：写屏障 + STW + 分布式终止检测

### 5. 内存分配的优化路径
- **微对象**（<16B）：tiny 分配器
- **小对象**（≤32KB）：mcache → mcentral → mheap（per-P 无锁）
- **大对象**（>32KB）：直接 mheap

这 5 个文件共同构成了 Go 运行时的核心——数据结构（slice/map）的内存管理、通信机制（channel）、调度（proc）、内存回收（GC）。理解它们，就理解了 Go "简单语法 + 高性能" 的秘密。
