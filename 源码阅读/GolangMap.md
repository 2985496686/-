





# 数据结构

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
	count     int // map中key-value的数量
	flags     uint8
    B         uint8  // 桶数组长度的指数，桶长度: 2^B
	noverflow uint16 // 溢出桶的数量
	hash0     uint32 // hash种子，在计算hash值的时候会被用到

	buckets    unsafe.Pointer // 指向哈希桶数组的指针
	oldbuckets unsafe.Pointer // 扩容过程中指向旧桶数组的指针，用于解决扩容过程中的读操作
	nevacuate  uintptr        // 扩容时的进度标识

	extra *mapextra // 预申请的溢出桶，防止在用户调用的时候才去申请空间
}
```



# 创建map



```go
func makemap(t *maptype, hint int, h *hmap) *hmap {
    // 判断预分配的容量是否溢出或者超过了最大容量
    //MulUintptr函数会计算两个参数的乘积，并判断是否溢出，未溢出会将乘积返回
    //t.bucket.size 表示map内部数据结构的大小
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}
	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
    // 生成随机的hash因子
	h.hash0 = fastrand()
    
    // 寻找不大于负载因子情况下最小的bucket数组大小
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// 初始化桶数组和备用溢出数组
    // B == 0 时采取lazy init的方法
	if h.B != 0 {
		var nextOverflow *bmap
        // 根据内部数据结构t和buctet数组大小 2^B 来创建bucket数组和预申请的溢出数组
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}
```

