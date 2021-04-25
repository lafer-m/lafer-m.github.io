---
layout: post
title: golang/map内存布局详解
tags: [golang]
---

// A map is just a hash table. The data is arranged
// into an array of buckets. Each bucket contains up to
// 8 key/value pairs. The low-order bits of the hash are
// used to select a bucket. Each bucket contains a few
// high-order bits of each hash to distinguish the entries
// within a single bucket.
//
// If more than 8 keys hash to a bucket, we chain on
// extra buckets.
//
// When the hashtable grows, we allocate a new array
// of buckets twice as big. Buckets are incrementally
// copied from the old bucket array to the new bucket array.
//
// Map iterators walk through the array of buckets and
// return the keys in walk order (bucket #, then overflow
// chain order, then bucket index).  To maintain iteration
// semantics, we never move keys within their bucket (if
// we did, keys might be returned 0 or 2 times).  When
// growing the table, iterators remain iterating through the
// old table and must check the new table if the bucket
// they are iterating through has been moved ("evacuated")
// to the new table.

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
    // 随机的hash key.
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
	bmap *bmap // overflow 溢出的bucket, 由于bucket固定只能存储8对key/value，所以golang处理hash冲突的时候使用链表来解决。
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

### 参考资料
[map详解](https://segmentfault.com/a/1190000023879178)