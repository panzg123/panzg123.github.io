---
layout: post
title: Python中的函数参数传递问题
categories: Python
---

参考自：[stackoverflow](http://stackoverflow.com/questions/986006/how-do-i-pass-a-variable-by-reference)


首先，贴Python中的金科玉律，**“变量无类型，对象有类型”**


对象有两种，“可更改”`（mutable）`与“不可更改”`（immutable）`对象.


`strings`, `tuples`, 和`numbers`是不可更改的对象，而`list`,`dict`等则是可以修改的对象。


变量可以看成一种引用，贴在了对象上，当变量传递给函数时，函数会复制一份引用；  

* 如果你往函数里传递可更改对象，那么函数体里的引用和函数体外的引用是指向同一块内存，你可以在函数中更改，但是如果你重绑定引用，则外部毫不知情，外部引用仍指向原对象。  

* 如果你往函数里传递不可变更对象，那么函数里的引用和函数体外的引用是完全不同的。  

让我们来看一些例子：
  
### List--可变更对象  
  
#### example1

这里我们传递List对象，并在函数中对其进行修改  

```
def try_to_change_list_contents(the_list):
    print 'got', the_list
    the_list.append('four')
    print 'changed to', the_list

outer_list = ['one', 'two', 'three']

print 'before, outer_list =', outer_list
try_to_change_list_contents(outer_list)
print 'after, outer_list =', outer_list
```


输出：  

```
before, outer_list = ['one', 'two', 'three']
got ['one', 'two', 'three']
changed to ['one', 'two', 'three', 'four']
after, outer_list = ['one', 'two', 'three', 'four']
```  
  
#### example2  

上面的例子是我们的常见做法，但是假如我们在函数体中对List进行重绑定，则外部引用对象值不会改变，如下：

```
def try_to_change_list_reference(the_list):
    print 'got', the_list
    the_list = ['and', 'we', 'can', 'not', 'lie']
    print 'set to', the_list

outer_list = ['we', 'like', 'proper', 'English']

print 'before, outer_list =', outer_list
try_to_change_list_reference(outer_list)
print 'after, outer_list =', outer_list
```  

输出：

```
before, outer_list = ['we', 'like', 'proper', 'English']
got ['we', 'like', 'proper', 'English']
set to ['and', 'we', 'can', 'not', 'lie']
after, outer_list = ['we', 'like', 'proper', 'English']
```
  
### String--不可变更对象  
  
#### example3  

由于string类型是不可变更的，尽管我们尝试改变该引用，但是是无效的

```
def try_to_change_string_reference(the_string):
    print 'got', the_string
    the_string = 'In a kingdom by the sea'
    print 'set to', the_string

outer_string = 'It was many and many a year ago'

print 'before, outer_string =', outer_string
try_to_change_string_reference(outer_string)
print 'after, outer_string =', outer_string
```

输出  

```
before, outer_string = It was many and many a year ago
got It was many and many a year ago
set to In a kingdom by the sea
after, outer_string = It was many and many a year ago
```

### 如何解决String等对象的传参问题呢？
  
#### example4

建议在函数结束时，return返回新值，具体方法如下：

```
def return_a_whole_new_string(the_string):
    new_string = something_to_do_with_the_old_string(the_string)
    return new_string

# then you could call it like
my_string = return_a_whole_new_string(my_string)
```
  
#### example5  

如果你真的想避免使用返回值，那么你可以用类来包裹你的参数，来传递到函数中，比如使用`List`如下：

```
def use_a_wrapper_to_simulate_pass_by_reference(stuff_to_change):
    new_string = something_to_do_with_the_old_string(stuff_to_change[0])
    stuff_to_change[0] = new_string

# then you could call it like
wrapper = [my_string]
use_a_wrapper_to_simulate_pass_by_reference(wrapper)

do_something_with(wrapper[0])
```