---
layout: post
title: std::sort与qsort的学习
categories: CPP
---

### 背景
---------------

#### STL之`std::sort`算法

STL中常用算法之一，其函数原型为：

```
default (1)    
template <class RandomAccessIterator>
  void sort (RandomAccessIterator first, RandomAccessIterator last);
custom (2)    
template <class RandomAccessIterator, class Compare>
  void sort (RandomAccessIterator first, RandomAccessIterator last, Compare comp);
```

使用例子：

```
int myints[] = {32,71,12,45,26,80,53,33};
  std::vector<int> myvector (myints, myints+8);               // 32 71 12 45 26 80 53 33
```

可以定制`compare`来实现自定义排序

#### C库之`qsort`

C标准库在stdlib.h中提供qsort算法，其原型为：

```
void qsort (void* base, size_t num, size_t size,
            int (*compar)(const void*,const void*));
```

demo:

```
#include <stdio.h>
#include <stdlib.h>
int values[] = { 40, 10, 100, 90, 20, 25 };
int compare (const void * a, const void * b)
{
  return ( *(int*)a - *(int*)b );
}
int main ()
{
  int n;
  qsort (values, 6, sizeof(int), compare);
  for (n=0; n<6; n++)
     printf ("%d ",values[n]);
  return 0;
}
```

### 比较分析
--------------

#### 效率

有同学总认为C的效率总比C++的程序效率高，其实不然，比如std::sort的效率就比qsort更高，下面从源码角度来分析下。

qsort内部采用标准的`快排算法`来实现，std::sort采用`Introspective Sort`算法实现。

快速排序虽然平均时间复杂度为O(NlogN)，但是由于某些情况下可能退化成O(N^2),且快排是递归调用，导致额外的开销。

而`Introspective Sort`是一种混合式排序方法，参考[wiki](https://en.wikipedia.org/wiki/Introsort)，大数据量时采用正常的快排，效率为O(logN)；数据量较小时，采用插入排序，O(N)；递归过程恶化时，使用堆排序，复杂度为O(N logN)。

具体的源码分析，参考侯杰《STL源码剖析》中`std::sort`剖析，或者点击[知无涯之std::sort源码剖析](http://feihu.me/blog/2014/sgi-std-sort/)

#### 可读性

使用qsort()你得自己实现比较函数`cmp`,比如上面的例子；而std::sort可以使用STL内置的`less<Type>`或者`greater<Type>`函数对象，也可以是自定义`函数指针`，`lamda表达式`，`function类型`等。

比如使用`less<Type>`:

```
sort(vector.begin(),vector.end(),less<string>());
```

### reference：

---------------

1. [http://feihu.me/blog/2014/sgi-std-sort/](http://feihu.me/blog/2014/sgi-std-sort/)
2. [https://www.felix021.com/blog/read.php?1951](https://www.felix021.com/blog/read.php?1951)
3. 《Effective STL》