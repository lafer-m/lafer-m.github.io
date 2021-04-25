---
layout: post
title: golang/map内存布局详解
tags: [golang]
---

golang map的实现是一个hash table，数据存储在一个bucket的数组里面，每个bucket最多保存8对key-value。

### 简介
map中的数据存储在了一个bucket的数组里面，每个bucket可以存储8对key/value值。低位的hash值用于判断hash值落在哪个bucket中，简单来说的话就是hash % 2^B取余，获取该key落到的bucket. 高位的hash值用于在一个bucket里面确定key/value的位置。如果一个bucket里面的需要存储超过8个key，这个时候会创建溢出bucket,将超出的key,存储到溢出bucket里面去。  
当hash map增长的时候，将hmap.buckets新申请为原来的两倍，原来的buckets指向hmap.oldbuckets，并且进行渐进式的迁移，也就是说在下次插入或者删除的时候触发迁移，将oldbuckets中的key/value迁移到新的buckets中。遍历的时候有可能会需要遍历两个数组，如果oldbuckets存在。  

### 先看map的内存布局
如下的例子中，我直接打印了map[string]int64包含两个buckets不包含溢出桶的内存布局，示例代码中hmap/bmap的结构都是拷贝的runtime.map的数据结构,为了能够简单打印出对应的字段，并在示例代码中通过详细的注释解释数据段的含义，英文注释为runtime map.go中原始的注释。   
hmap:  make(map[string]int64,12)的时候会返回一个*hmap指针   
bmap:  桶的数据结构，示例中初始化创建的map中包含了两个桶
```
package main

import (
	"fmt"
	"reflect"
	"unsafe"
)

const (
	// Maximum number of key/value pairs a bucket can hold.
    // 一个桶中最多能存储8对key/value.
	bucketCntBits = 3
	bucketCnt     = 1 << bucketCntBits
)

// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
    // map 长度，即包含的key/value对数。
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
    // B ，桶的对数， 即map所拥有的桶数量是2^B个。
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    // 溢出桶的个数，map很小的时候是准的，大的map可能是不准的。
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
    // 随机的hash seed.
	hash0     uint32 // hash seed

    // 桶数组指针，[]bmap
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    // 如果不为nil的时候，说明发生了扩容， 大小为buckets的一半，同样是桶数组指针, []bmap
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    // 迁移进度，扩容时候已经从oldbuckets迁移到buckets中的数量。
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

    // 当key/value均不包含指针的时候，为避免扫描该bucket，将溢出桶存储在extra里面，并标记该bucket不包含指针。
	extra *mapextra // optional fields
}

// mapextra holds fields that are not present on all maps.
type mapextra struct {
	// If both key and value do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and value do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// nextOverflow holds a pointer to a free overflow bucket.
	nextOverflow *bmap
}

// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
    // hash值的高位定位这个key在这个bucket中的位置。
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt values.
	// NOTE: packing all the keys together and then all the values together makes the
	// code a bit more complicated than alternating key/value/key/value/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.


    // 在runtime map.go中以下部分并未定义， 本次测试为了打印数据加上了特定的定义，key/value的类型可能是任意类型。
	// 这里补全定制的数据结构，以获得完整的数据，获取map的内存布局。
	keys   [bucketCnt]string // string为1字节
	values [bucketCnt]int64  // int64为8字节
	// pading [n]byte  //假如需要padding，对齐字节就需要，我现在测试的情况看起来并不需要。
	bmap *bmap // overflow 溢出的bucket, 由于bucket固定只能存储8对key/value，golang处理hash冲突的时候使用链表来解决。
}

func main() {
	// 初始化的时候，make会为map生成两个
	aMap := make(map[string]int64, 12)
	aMap["test"] = 12
	aMap["abcd"] = 11
	p1 := unsafe.Pointer(&aMap)
	p2 := unsafe.Pointer(reflect.ValueOf(aMap).Pointer())

	// 这里两个指针并不是同一个，p1是指针的指针, 而p2才是指向hmap的指针。
	fmt.Printf("%v != %v\n", p1, p2)
	// 这里我想验证下map的指针底层内容就是hmap的开始地址么
	// 所以我把runtime.map里面的数据结构拷贝放到我自己的包了，然后做指针转换
	p3 := (*hmap)(unsafe.Pointer(reflect.ValueOf(aMap).Pointer()))
	PrintHmapStruct(p3)
}

func PrintHmapStruct(p *hmap) {
	fmt.Printf("map 长度： %#x\n", p.count)
	fmt.Printf("map flags: %#x\n", p.flags)
	fmt.Printf("map B 桶对数log_2: %#x\n", p.B)
	fmt.Printf("map 溢出桶的大概个数: %#x\n", p.noverflow)
	fmt.Printf("map hash key: %#x\n", p.hash0)
	//通过B=1,知道桶的个数是2^1，即是2个桶，而且没有溢出桶
	buts := (*[2]bmap)(p.buckets)
	for n, b := range buts {
		for k, topHash := range b.tophash {
			fmt.Printf("map 桶%d->topHash%d: %#x\n", n, k, topHash)
		}

		for k, key := range b.keys {
			fmt.Printf("map 桶%d->key%d: %s\n", n, k, key)
		}

		for k, value := range b.values {
			fmt.Printf("map 桶%d->value%d: %#x\n", n, k, value)
		}

		fmt.Printf("map 桶%d->溢出桶指针: %v\n", n, b.bmap)
	}

	// 打印剩下的hmap字段
	// oldbuckets，map扩容的时候才不为nil, 同样是指向桶数组的指针, 本次测试为nil。
	fmt.Printf("map oldbuckets(扩容的时候不为nil): %v\n", p.oldbuckets)
	// nevacuate 已经迁移的buckets数量。扩容的时候，p.buckets数组扩容为原来的两倍，并将老的值指向p.oldbuckets, 然后在insert或delete操作时进行迁移操作，迁移到新的buckets数组里面。
	fmt.Printf("map nevacuate: %#x\n", p.nevacuate)
	// 当key/value均不包含指针的时候，为避免扫描该bucket，将溢出桶存储在extra里面，并标记该bucket不包含指针。
	fmt.Printf("map extra: %v\n", p.extra)
}

```

运行结果, 分析结果，buckets数组包含两个桶（初始化的时候创建），而两个key：test/abcd可能落在任何一个桶中，在下面的结果中，落在了第二个桶里面，如下的bucket可能位于内存任意位置。
```
map 长度： 0x2
map flags: 0x0
map B 桶对数log_2: 0x1
map 溢出桶的大概个数: 0x0
map hash key: 0x200dc23e
map buckets *[]bmap指针: (unsafe.Pointer)(0xc000096000)
	map 桶0->topHash0: 0x0
	map 桶0->topHash1: 0x0
	map 桶0->topHash2: 0x0
	map 桶0->topHash3: 0x0
	map 桶0->topHash4: 0x0
	map 桶0->topHash5: 0x0
	map 桶0->topHash6: 0x0
	map 桶0->topHash7: 0x0
	map 桶0->key0: 
	map 桶0->key1: 
	map 桶0->key2: 
	map 桶0->key3: 
	map 桶0->key4: 
	map 桶0->key5: 
	map 桶0->key6: 
	map 桶0->key7: 
	map 桶0->value0: 0x0
	map 桶0->value1: 0x0
	map 桶0->value2: 0x0
	map 桶0->value3: 0x0
	map 桶0->value4: 0x0
	map 桶0->value5: 0x0
	map 桶0->value6: 0x0
	map 桶0->value7: 0x0
	map 桶0->溢出桶指针: <nil>
	map 桶1->topHash0: 0x23
	map 桶1->topHash1: 0xb8
	map 桶1->topHash2: 0x0
	map 桶1->topHash3: 0x0
	map 桶1->topHash4: 0x0
	map 桶1->topHash5: 0x0
	map 桶1->topHash6: 0x0
	map 桶1->topHash7: 0x0
	map 桶1->key0: test
	map 桶1->key1: abcd
	map 桶1->key2: 
	map 桶1->key3: 
	map 桶1->key4: 
	map 桶1->key5: 
	map 桶1->key6: 
	map 桶1->key7: 
	map 桶1->value0: 0xc
	map 桶1->value1: 0xb
	map 桶1->value2: 0x0
	map 桶1->value3: 0x0
	map 桶1->value4: 0x0
	map 桶1->value5: 0x0
	map 桶1->value6: 0x0
	map 桶1->value7: 0x0
	map 桶1->溢出桶指针: <nil>
map oldbuckets(扩容的时候不为nil): <nil>
map nevacuate: 0x0
map extra: <nil>
```

分析完上述结果后，我们再看如下的内存布局，就很容易理解了
![map-memory-layout](http://www.mrzzjiy.cn/assets/golang-map-mem.png)

### map创建

源码分析基于make(map[string]int64,12):  makemap --> makeBucketArray --> newarray
```
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
	if h == nil {
		h = new(hmap)
	}
    // 初始化哈希key，随机值
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
	B := uint8(0)
    // 计算B的大小，当hint=12,B=1的时候，12>8 && 12> 13*(2/2)返回false,这个时候算的bucket数量就是为了满足loadFactor要小于6.5.
	for overLoadFactor(hint, B) {
		B++
	}
    // 计算出B=1
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
    // 这个时候B=1，所以需要初始化创建2^1个bucket,即2个.
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
makeBucketArray
```
// makeBucketArray initializes a backing array for map buckets.
// 1<<b is the minimum number of buckets to allocate.
// dirtyalloc should either be nil or a bucket array previously
// allocated by makeBucketArray with the same t and b parameters.
// If dirtyalloc is nil a new backing array will be alloced and
// otherwise dirtyalloc will be cleared and reused as backing array.
func makeBucketArray(t *maptype, b uint8, dirtyalloc unsafe.Pointer) (buckets unsafe.Pointer, nextOverflow *bmap) {
    // base = 1 << 1 = 2, 需要创建的bucket数量为2个
	base := bucketShift(b)
	nbuckets := base
	// For small b, overflow buckets are unlikely.
	// Avoid the overhead of the calculation.
    // 这里如果b很小，那么溢出buckets的计算一般可以跳过, 不会预申请溢出桶。
	if b >= 4 {
		// Add on the estimated number of overflow buckets
		// required to insert the median number of elements
		// used with this value of b.
		nbuckets += bucketShift(b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}

	if dirtyalloc == nil {
        // 需要新申请内存，这里的作用就是申请一个[2]bmap类型的数组，返回了一个指向数组的指针。
		buckets = newarray(t.bucket, int(nbuckets))
	} else {
		// dirtyalloc was previously generated by
		// the above newarray(t.bucket, int(nbuckets))
		// but may not be empty.
		buckets = dirtyalloc
		size := t.bucket.size * nbuckets
        // 清空底层数组。
		if t.bucket.kind&kindNoPointers == 0 {
			memclrHasPointers(buckets, size)
		} else {
			memclrNoHeapPointers(buckets, size)
		}
	}

    // 预申请一些溢出桶，放到extra中。
	if base != nbuckets {
		// We preallocated some overflow buckets.
		// To keep the overhead of tracking these overflow buckets to a minimum,
		// we use the convention that if a preallocated overflow bucket's overflow
		// pointer is nil, then there are more available by bumping the pointer.
		// We need a safe non-nil pointer for the last overflow bucket; just use buckets.
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}
```
newarray在malloc.go中，这里就不深入展开了。
```
// newarray allocates an array of n elements of type typ.
func newarray(typ *_type, n int) unsafe.Pointer {
	if n == 1 {
		return mallocgc(typ.size, typ, true)
	}
	mem, overflow := math.MulUintptr(typ.size, uintptr(n))
	if overflow || mem > maxAlloc || n < 0 {
		panic(plainError("runtime: allocation size out of range"))
	}
	return mallocgc(mem, typ, true)
}

```


### assign key/value
同样以aMap["test"] = 12进行分析
```
// Like mapaccess, but allocates a slot for the key if it is not present in the map.
func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
    // check部分代码，跳过分析。
	if h == nil {
		panic(plainError("assignment to entry in nil map"))
	}
	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(mapassign)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled {
		msanread(key, t.key.size)
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}
    // 计算key的hash值。
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write.
    // 设置map的flags状态
	h.flags ^= hashWriting

	if h.buckets == nil {
        // 申请一个底层bucket数组。
		h.buckets = newobject(t.bucket) // newarray(t.bucket, 1)
	}

again:
    // 取 hash & (1<< 1 - 1),这里就是取hash值最后一个bit的值作为bucket。取余操作，等价于 hash % (2 ^ B)
	bucket := hash & bucketMask(h.B)
    // oldBuckets不为nil的时候
	if h.growing() {
        // 迁移老的buckets
		growWork(t, h, bucket)
	}
    //通过bucket值取到bmap
	b := (*bmap)(unsafe.Pointer(uintptr(h.buckets) + bucket*uintptr(t.bucketsize)))
    // hash值的高8位
	top := tophash(hash)

	var inserti *uint8
	var insertk unsafe.Pointer
	var val unsafe.Pointer
bucketloop:
	for {
		for i := uintptr(0); i < bucketCnt; i++ {
            // 不存在该key
			if b.tophash[i] != top {
                // slot是空闲的，且key的位置还没被分配，就可以将该slot分配给这个key
				if isEmpty(b.tophash[i]) && inserti == nil {
					inserti = &b.tophash[i]
					insertk = add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
					val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				}
                // 这个slot是空闲的，但是没有其他空闲位置了,这个时候会触发扩容hashGrow.
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
            // 存在该key,更新value值。
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if !alg.equal(key, k) {
				continue
			}
			// already have a mapping for key. Update it.
			if t.needkeyupdate() {
				typedmemmove(t.key, k, key)
			}
			val = add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			goto done
		}
        // slot都满了，是否有溢出桶，有就就行下次循环，没有就走hashGrow或者newoverflow.
		ovf := b.overflow(t)
		if ovf == nil {
			break
		}
		b = ovf
	}

	// Did not find mapping for key. Allocate new cell & add entry.

	// If we hit the max load factor or we have too many overflow buckets,
	// and we're not already in the middle of growing, start growing.
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
        // 超过loaderFactor或者有太多的溢出桶，触发扩容，然后从新分配slot给key
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}

	if inserti == nil {
		// all current buckets are full, allocate a new one.
        // 新申请溢出桶
		newb := h.newoverflow(t, b)
		inserti = &newb.tophash[0]
		insertk = add(unsafe.Pointer(newb), dataOffset)
		val = add(insertk, bucketCnt*uintptr(t.keysize))
	}

	// store new key/value at insert position
	if t.indirectkey() {
		kmem := newobject(t.key)
		*(*unsafe.Pointer)(insertk) = kmem
		insertk = kmem
	}
	if t.indirectvalue() {
		vmem := newobject(t.elem)
		*(*unsafe.Pointer)(val) = vmem
	}
	typedmemmove(t.key, insertk, key)
	*inserti = top
	h.count++

done:
	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
	if t.indirectvalue() {
		val = *((*unsafe.Pointer)(val))
	}
	return val
}

```

### delete key
delete逻辑会设置slot/cell的状态。清空key/value的内存。
```
func mapdelete(t *maptype, h *hmap, key unsafe.Pointer) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapdelete)
		racewritepc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map writes")
	}

	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))

	// Set hashWriting after calling alg.hash, since alg.hash may panic,
	// in which case we have not actually done a write (delete).
	h.flags ^= hashWriting

	bucket := hash & bucketMask(h.B)
    // 触发迁移
	if h.growing() {
		growWork(t, h, bucket)
	}
	b := (*bmap)(add(h.buckets, bucket*uintptr(t.bucketsize)))
	bOrig := b
	top := tophash(hash)
search:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break search
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			k2 := k
			if t.indirectkey() {
				k2 = *((*unsafe.Pointer)(k2))
			}
			if !alg.equal(key, k2) {
				continue
			}
			// Only clear key if there are pointers in it.
            // 清除key的值
			if t.indirectkey() {
				*(*unsafe.Pointer)(k) = nil
			} else if t.key.kind&kindNoPointers == 0 {
				memclrHasPointers(k, t.key.size)
			}
            // 清除value的值
			v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
			if t.indirectvalue() {
				*(*unsafe.Pointer)(v) = nil
			} else if t.elem.kind&kindNoPointers == 0 {
				memclrHasPointers(v, t.elem.size)
			} else {
				memclrNoHeapPointers(v, t.elem.size)
			}
            // 设置slot的状态为空闲。
			b.tophash[i] = emptyOne
			// If the bucket now ends in a bunch of emptyOne states,
			// change those to emptyRest states.
			// It would be nice to make this a separate function, but
			// for loops are not currently inlineable.
            // 这里是检查和设置是否是emptyRest状态的逻辑
			if i == bucketCnt-1 {
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
					if b == bOrig {
						break // beginning of initial bucket, we're done.
					}
					// Find previous bucket, continue at its last entry.
					c := b
					for b = bOrig; b.overflow(t) != c; b = b.overflow(t) {
					}
					i = bucketCnt - 1
				} else {
					i--
				}
				if b.tophash[i] != emptyOne {
					break
				}
			}
		notLast:
			h.count--
			break search
		}
	}

	if h.flags&hashWriting == 0 {
		throw("concurrent map writes")
	}
	h.flags &^= hashWriting
}

```






### get key
map获取key,有两种形式，如 v := aMap["test"],或者 v, ok := aMap["test"], 如果没有该key ,v=0, ok =false.   
find bucket --> 循环链表查找key对应的value 

```
// mapaccess1 returns a pointer to h[key].  Never returns nil, instead
// it will return a reference to the zero object for the value type if
// the key is not in the map.
// NOTE: The returned pointer may keep the whole map live, so don't
// hold onto it for very long.
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
    //计算hash值，获取bucket,如果有oldBuckets，也要再old里面找。
	alg := t.key.alg
	hash := alg.hash(key, uintptr(h.hash0))
	m := bucketMask(h.B)
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
        // 是否已经被迁移
		if !evacuated(oldb) {
			b = oldb
		}
	}
	top := tophash(hash)
bucketloop:
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
            // 查找key相等的返回value.
			if alg.equal(key, k) {
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				if t.indirectvalue() {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}

```




### 参考资料
[map详解](https://segmentfault.com/a/1190000023879178)