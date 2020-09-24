---
layout: post
title: 插入排序(C语言)
tags: [algorithm]
---


### 插入排序算法实现c语言版本
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