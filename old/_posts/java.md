---
title: java
date: 2019-06-28 21:01:52
categories: "java"
tags:

---

java参数传递 vs c++引用 vs go指针传递
===
1. [**c++引用**](https://blog.csdn.net/dujiangyan101/article/details/2844138)
---
>**引用**  
>
>- 引用不可为空，对象必须存在不可为null  
>- 地址概念：某块内存的别名
>- 引用是别名，地址不可更改
>- 引用使用无需解引*，对引用的赋值为对地址变量的赋值
>- sizeof引用为引用指向的对象的大小  
>>
>>  ```int i = 5; 
int j = 6; 
int &k = i; 
k = j; // k 和i 的值都变成了6;     
```


  
2. [**java引用**](https://blog.csdn.net/jiangnan2014/article/details/22944075)
---  

>**引用**
>- 值传递
>- 所有的参数传递都是 传值，从来没有 传引用 这个事实；
>- 所有的参数传递都会在 程序运行栈上 新分配一个 值 的复制品；
>- java只有按值传递，所谓的按地址(引用)传递，也属于按值传递，只不过这个“值”是个地址;
>- 对于引用类型的传参也是传值的，传的是引用类型的值，其实就是对象的地址；
>- java所有对像变量都是对像的引用；
>- 函数的形式参数，是传入参数的拷贝；引用变量之间拷贝的是【地址】，基本变量之间拷贝的是 内存中的值 （被称为直接量）；
>>  ```public class Test1
{
        String a = "123";
        public static void test(Test1 test)
        {
                test.a = "abc";//更改test指向的内存空间的值
        }
        public static void main(String[] args)
        {
                Test1 test1 = new Test1();
                test1.a = "567";
                System.out.println(test1.a); //567
                test(test1);//复制test1的地址至test
                System.out.println(test1.a); //abc
        }
}
```
>- java引用可以改变引用的地址，java引用.为对引用地址所指向对象的操作，=表示对引用本身的修改（地址)
>>  
>> ```public class Test
{
        public static void test(StringBuffer str)
        {
                str.append("world");//改变引用对象的内存空间的值
                str = new StringBuffer("world");//更新地址，并对地址对象赋值，不会改变原值
        }
        public static void main(String[] args)
        {
                StringBuffer str = new StringBuffer("hello");
                System.out.println(str);  //hello
                test(str);//复制地址
                System.out.println(str);  //注释掉str.append为hello，否则为hello world
        }
}
```

>[参考1: java参数传递](https://juejin.im/post/5bce68226fb9a05ce46a0476)  
>[参考2: java堆栈](https://pengjiaheng.iteye.com/blog/518623)

3. **[go(c语言)指针传递](https://www.flysnow.org/2018/02/24/golang-function-parameters-passed-by-value.html)**
---

>+ 类似C语言，函数传值一切皆copy
>+ 参数为指针则为指针所指向的地址的值copy
>

```package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, playground")
	var a int = 1
	b := &a
	fmt.Println("the addr of b: ",&b, ", a: ", &a, " value of b: ", b, "the value of *b: ", *b)
	fun1(b)
	fmt.Println("2st the addr of b: ",&b, ", a: ", &a, " value of b: ", b, "the value of *b: ", *b)
	
}

func fun1(c *int){

fmt.Println("the add of c: ", &c, "the value of c: ", c, "the value of *c: ", *c)
var d int = 2
c = &d
fmt.Println("2 st the add of c: ", &c, "the value of c: ", c, "the value of *c: ", *c)
}
```
Hello, playground
the addr of b:  0x40c130 , a:  0x414020  value of b:  0x414020 the value of *b:  1  

the add of c:  0x40c138 the value of c:  0x414020 the value of *c:  1  

2 st the add of c:  0x40c138 the value of c:  0x41402c the value of *c:  2  

2st the addr of b:  0x40c130 , a:  0x414020  value of b:  0x414020 the value of *b:  1

>传map

```package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello, playground")
	a := make(map[int]int)
	a[1]=3
	fmt.Printf("1st a[1] is: %d addr of map:%p \n", a[1],&a)
	fun1(a)
	fmt.Printf("1st a[1] is: %d addr of map:%p \n", a[1],&a)

	
}

func fun1(c map[int]int){
fmt.Printf("1st c[1] is: %d addr of map:%p \n", c[1],&c)
c=make(map[int]int)
c[1]=4
fmt.Printf("1st c[1] is: %d addr of map:%p \n", c[1],&c)


}
```
>Hello, playground
  
>1st a[1] is: 3 addr of map:0x40c130  
 
>1st c[1] is: 3 addr of map:0x40c138  

>1st c[1] is: 4 addr of map:0x40c138   

>1st a[1] is: 3 addr of map:0x40c130 

4. **[小结：以c++为准](https://blog.csdn.net/luoshenfu001/article/details/8601494)**
---
>+ 参数列表，初始化之后不可更改，名称：值
>+ 指针为独立的参数，其参数列表中值为本身的地址，初始化之后不可变更
>+ 引用作为别名，其参数列表值为引用变量的地址，初始化之后不可变更
>+ 指针指向的地址可以可变，引用引用的地址不可改变（参数列表的初始化)
>+ 对指针的操作及运算为指针本身的运算eg:++指针指向的地址++
>+ 对引用的运算为引用所引用的对象本身的运算
>+ const int *p， p指向的变量为常量，p可以指向不同的常量
>+ int const * p, p不可改变其指向，其指向的对象为变量
>+ 与语言内存模型相关
>+ 一般分为堆、栈，堆为数据区（malloc calloc 数组 map chan 对象包括对象成员参数），栈为运行时变量区（局部变量）

