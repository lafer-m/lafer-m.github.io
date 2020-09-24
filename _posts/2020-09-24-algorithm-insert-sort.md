---
layout: post
title: 插入排序(C/golang)
tags: [algorithm]
---


### 插入排序算法实现
随机生成长度为n的整数数组，算法的时间消耗是 x*n^2 x是常数因子，随着数据越多，时间消耗约大，数据量小的时候是一种比较快速的排序方法。
```
#include <stdio.h>
#include <stdlib.h>
#include <time.h>


int * randomArr()
{
	int* a;
	a = (int*)malloc(sizeof(int));
	srand((unsigned)time(NULL));
	for (int i=0;i<10;i++) {
		a[i] = rand()  ;
	}
	return a;
}


int main()
{
  int *a = randomArr();
  for(int i=0; i<10; i++) {
	  printf("before sort *(a + %d): %d\n",i, *(a + i) );
  }
  for(int j=1; j<10;j++){
	  int value = a[j];
	  int k = j -1;
	  while((k>=0) && (a[k]> value)) {
		  a[k+1] = a[k];
		  k--;
	  }
	  a[k+1] = value;
  }
  for(int i=0; i<10; i++) {
	  printf("after sort *(a + %d): %d\n",i, *(a + i) );
  }
  free(a);
  return 0;
}
```

golang版本
```
package main

import (
	"math/rand"
	"time"
)

func randomArr() []uint32 {

	result := []uint32{}
	rand.Seed(time.Now().UnixNano())
	for i := 0; i < 10; i++ {
		result = append(result, rand.Uint32())
	}
	return result
}

func main() {
	arr := randomArr()

	for k,v := range arr {
		println("before sort ", k,v)
	}

	for i := 1; i < len(arr); i++ {
		currentValue := arr[i]
		j := i - 1

		for j >= 0 && arr[j] > currentValue {
			arr[j+1] = arr[j]
			j--
		}
		arr[j+1] = currentValue
	}

	for k,v := range arr {
		println("after sort ", k,v)
	}
}
```