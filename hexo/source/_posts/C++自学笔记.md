---
title: C++自学笔记
date: 2019-01-04 09:09:44
tags: C++
---

## C++自学笔记

### 一. 基础

1.变量类型：在C的基础上添加了wchar_t。

2.变量声明：变量和函数声明只在编译时有意义，在程序连接时编译器需要实际的变量声明也就是变量定义。并且声明可以多次，但是变量定义只能一次。

extern修饰变量：用于提供一个全局变量的引用，全局变量对所有的程序文件都是可见的。当您有多个文件且定义了一个可以在其他文件中使用的全局变量或函数时，可以在其他文件中使用 *extern* 来得到已定义的变量或函数的引用。可以这么理解，*extern* 是用来在另一个文件中声明一个全局变量或函数。extern 修饰符通常用于当有两个或多个文件共享相同的全局变量或函数的时候

```c++
// a.cpp
#include <iostream>
int count ;
extern void write_extern();
int main()
{
   count = 5;
   write_extern();
}

// b.cpp
#include <iostream>
extern int count;
void write_extern(void)
{
   std::cout << "Count is " << count << std::endl;
}
```



static：存储类指示编译器在程序的生命周期内保持局部变量的存在，而不需要在每次它进入和离开作用域时进行创建和销毁。因此，使用 static 修饰局部变量可以在函数调用之间保持局部变量的值。static 修饰符也可以应用于全局变量。当 static 修饰全局变量时，会使变量的作用域限制在声明它的文件内。

3.类型限定符：

| const    | **const** 类型的对象在程序执行期间不能被修改改变。           |
| -------- | ------------------------------------------------------------ |
| volatile | 修饰符 **volatile** 告诉编译器不需要优化volatile声明的变量，让程序可以直接从内存中读取变量。对于一般的变量编译器会对变量进行优化，将内存中的变量值放在寄存器中以加快读写效率。 |
| restrict | 由 **restrict** 修饰的指针是唯一一种访问它所指向的对象的方式。只有 C99 增加了新的类型限定符 restrict。 |



4.函数

默认函数：您可以为参数列表中后边的每一个参数指定默认值。当调用函数时，如果实际参数的值留空，则使用这个默认值。**注意：这里的默认参数必须放在普通参数之后**

```c++
int sum(int a, int b=20)
{
  int result;
  result = a + b;
  return (result);
}
```



函数的三种传值方式：

- 传值调用：把参数的实际值复制给函数的形式参数。在这种情况下，修改函数内的形式参数对实际参数没有影响。
- 指针调用把参数的地址复制给形式参数。在函数内，该地址用于访问调用中要用到的实际参数。这意味着，修改形式参数会影响实际参数。
- 引用调用：把参数的引用复制给形式参数。在函数内，该引用用于访问调用中要用到的实际参数。这意味着，修改形式参数会影响实际参数。

5.字符串

C风格: char a[10] = "hello";

C++风格: string a = "s d" + " ds";





### 二. 指针与数据结构

1.指针的声明，赋值和取值：

```cpp
int main ()
{
   int  var = 20;   // 实际变量的声明
   int  *ip;        // 指针变量的声明
   ip = &var;       // 在指针变量中存储 var 的地址 
   cout << "Value of var variable: ";
   cout << var << endl;
 
   // 输出在指针变量中存储的地址
   cout << "Address stored in ip variable: ";
   cout << ip << endl;

   // 访问指针中地址的值
   cout << "Value of *ip variable: ";
   cout << *ip << endl;
   return 0;
}
```

在变量声明的时候，如果没有确切的地址可以赋值，为指针变量赋一个 NULL 值是一个良好的编程习惯。

2.在函数参数列表中的两种表示方法：

```cpp
void swap(int &x, int &y)  // 引用传值，这里&x表示的是传入的参数是一个值，可以直接使用，或者用&取出这个参数的指针
    
void swap_2(int *x, int *y)  // 指针传值，这里*x代表传入的参数是一个指针，可以直接使用，或者用*取出这个参数的值
```

3.指针可以进行数学比较，毕竟都是指针。

4.指针数组：

```cpp
int *ptr[MAX]  // 指针数组
int (*ptr)[MAX] // 数组指针
```

5.函数中返回指针

```cpp
int * function()
```



### 引用

引用很容易与指针混淆，它们之间有三个主要的不同：

- 不存在空引用。引用必须连接到一块合法的内存。
- 一旦引用被初始化为一个对象，就不能被指向到另一个对象。指针可以在任何时候指向到另一个对象。
- 引用必须在创建时被初始化。指针可以在任何时间被初始化。



### 结构

结构变量用.来引出结构中的成员，结构变量指针用->来引出。

```cpp
using namespace std;
void printBook( struct Books *book );
 
struct Books
{
   char  title[50];
   char  author[50];
   char  subject[100];
   int   book_id;
};
 
int main( )
{
   Books Book1;        // 定义结构体类型 Books 的变量 Book1
   Books Book2;        // 定义结构体类型 Books 的变量 Book2
 
    // Book1 详述
   strcpy( Book1.title, "C++ 教程");
   strcpy( Book1.author, "Runoob"); 
   strcpy( Book1.subject, "编程语言");
   Book1.book_id = 12345;
 
   // Book2 详述
   strcpy( Book2.title, "CSS 教程");
   strcpy( Book2.author, "Runoob");
   strcpy( Book2.subject, "前端技术");
   Book2.book_id = 12346;
 
   // 通过传 Book1 的地址来输出 Book1 信息
   printBook( &Book1 );
 
   // 通过传 Book2 的地址来输出 Book2 信息
   printBook( &Book2 );
 
   return 0;
}
// 该函数以结构指针作为参数
void printBook( struct Books *book )
{
   cout << "书标题  : " << book->title <<endl;
   cout << "书作者 : " << book->author <<endl;
   cout << "书类目 : " << book->subject <<endl;
   cout << "书 ID : " << book->book_id <<endl;
}
```





### 三. 类和对象

1.继承：C++类定义及使用与Java十分相似，但是引入了继承方式的区别, 规则是：

> **public 继承：**基类 public 成员，protected 成员，private 成员的访问属性在派生类中分别变成：public, protected, private
>
> **protected 继承：**基类 public 成员，protected 成员，private 成员的访问属性在派生类中分别变成：protected, protected, private
>
> **private 继承：**基类 public 成员，protected 成员，private 成员的访问属性在派生类中分别变成：private, private, private

看个例子

```cpp
#include<iostream>
#include<assert.h>
using namespace std;
class A{
public:
  int a;
  A(){
    a1 = 1;
    a2 = 2;
    a3 = 3;
    a = 4;
  }
  void fun(){
    cout << a << endl;    //正确
    cout << a1 << endl;   //正确
    cout << a2 << endl;   //正确
    cout << a3 << endl;   //正确
  }
public:
  int a1;
protected:
  int a2;
private:
  int a3;
};
class B : protected A{
public:
  int a;
  B(int i){
    A();
    a = i;
  }
  void fun(){
    cout << a << endl;       //正确，public成员。
    cout << a1 << endl;       //正确，基类的public成员，在派生类中变成了protected，可以被派生类访问。
    cout << a2 << endl;       //正确，基类的protected成员，在派生类中还是protected，可以被派生类访问。
    cout << a3 << endl;       //错误，基类的private成员不能被派生类访问。
  }
};
int main(){
  B b(10);
  cout << b.a << endl;       //正确。public成员
  cout << b.a1 << endl;      //错误，protected成员不能在类外访问。
  cout << b.a2 << endl;      //错误，protected成员不能在类外访问。
  cout << b.a3 << endl;      //错误，private成员不能在类外访问。
  system("pause");
  return 0;
}
```



类中变量和函数声明也要使用这三个修饰符。并且默认是private。

2.C++同时引入了析构函数，用于在删除对象的时候运行。C++类会自动控制类的空间的申请和释放。所以析构函数就是在这个时候调用。

```cpp
Line::Line(double len);
~Line:Line();
```



3.拷贝构造函数

**拷贝构造函数**是一种特殊的构造函数，它在创建对象时，是使用同一类中之前创建的对象来初始化新创建的对象。拷贝构造函数通常用于：

- 通过使用另一个同类型的对象来初始化新创建的对象。
- 复制对象把它作为参数传递给函数。
- 复制对象，并从函数返回这个对象。

如果在类中没有定义拷贝构造函数，编译器会自行定义一个。如果类带有指针变量，并有动态内存分配，则它必须有一个拷贝构造函数。拷贝构造函数的最常见形式如下：

```cpp
classname (const classname &obj) {    // 构造函数的主体 
}
```

比如接下来的例子给出了三者组合的使用

```cpp
// 成员函数定义，包括构造函数
Line::Line(int len)
{
    cout << "调用构造函数" << endl;
    // 为指针分配内存
    ptr = new int;
    *ptr = len;
}
 
Line::Line(const Line &obj)
{
    cout << "调用拷贝构造函数并为指针 ptr 分配内存" << endl;
    ptr = new int;
    *ptr = *obj.ptr; // 拷贝值
}
 
Line::~Line(void)
{
    cout << "释放内存" << endl;
    delete ptr;
}
```



4.指向类的指针需要使用成员访问运算符->，就像指向结构的指针一样使用。

5.类的静态成员数据：

与Java不同的地方是不能把静态成员在类的所有对象中是共享的。如果不存在其他的初始化语句，在创建第一个对象时，所有的静态数据都会被初始化为零。我们不能把静态成员的初始化放置在类的定义中，但是可以在类的外部通过使用范围解析运算符 **::** 来重新声明静态变量从而对它进行初始化，函数也是如此：

```cpp
#include <iostream>
 
using namespace std;
 
class Box
{
   public:
      static int objectCount;
      // 构造函数定义
      Box(double l=2.0, double b=2.0, double h=2.0)
      {
         cout <<"Constructor called." << endl;
         length = l;
         breadth = b;
         height = h;
         // 每次创建对象时增加 1
         objectCount++;
      }
      double Volume()
      {
         return length * breadth * height;
      }
      static int getCount()
      {
         return objectCount;
      }
   private:
      double length;     // 长度
      double breadth;    // 宽度
      double height;     // 高度
};
 
// 初始化类 Box 的静态成员
int Box::objectCount = 0;
 
int main(void)
{
  
   // 在创建对象之前输出对象的总数
   cout << "Inital Stage Count: " << Box::getCount() << endl;
 
   Box Box1(3.3, 1.2, 1.5);    // 声明 box1
   Box Box2(8.5, 6.0, 2.0);    // 声明 box2
 
   // 在创建对象之后输出对象的总数
   cout << "Final Stage Count: " << Box::getCount() << endl;
 
   return 0;
}
```



6.**运算符重载**

重载的运算符是带有特殊名称的函数，函数名是由关键字 operator 和其后要重载的运算符符号构成的。与其他函数一样，重载运算符有一个返回类型和一个参数列表。

```cpp
Box operator+(const Box&);
```



7.**虚函数**

**虚函数** 是在基类中使用关键字 **virtual** 声明的函数。在派生类中重新定义基类中定义的虚函数时，会告诉编译器不要静态链接到该函数。也就是说，虚函数的调用不是在编译时刻确定的，而是在运行时刻确定的

我们想要的是在程序中任意点可以根据所调用的对象类型来选择调用的函数，这种操作被称为**动态链接**，或**后期绑定**。

虚函数有点类似于Java中的可以重写的函数。Java之中可以重写的函数都是运行时候确定的，

如果不想给一个虚函数给出实现，就定为纯需函数。纯虚函数有点像接口的函数

```cpp
virtual double getVolume() = 0;
```





8.接口和抽象类

如果类中至少有一个函数被声明为纯虚函数，则这个类就是抽象类。纯虚函数是通过在声明中使用 "= 0" 来指定的。



## 四.高级

### 1.文件操作

文件操作头文件

- ofstream：输出文件流
- ifstream：输入文件流
- fstream：文件流，拥有输入输出的功能

2.在从文件读取信息或者向文件写入信息之前，必须先打开文件。**ofstream** 和 **fstream** 对象都可以用来打开文件进行写操作，如果只需要打开文件进行读操作，则使用 **ifstream** 对象。

下面是 open() 函数的标准语法，open() 函数是 fstream、ifstream 和 ofstream 对象的一个成员。

```cpp
void open(const char *filename, ios::openmode mode);
```

这里的openmode指明了打开的模式

| 模式标志   | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| ios::app   | 追加模式。所有写入都追加到文件末尾。                         |
| ios::ate   | 文件打开后定位到文件末尾。                                   |
| ios::in    | 打开文件用于读取。                                           |
| ios::out   | 打开文件用于写入。                                           |
| ios::trunc | 如果该文件已经存在，其内容将在打开文件之前被截断，即把文件长度设为 0。 |

打开以后使用 >> 和 << 进行数据的写入与读出。与cout cin类似。



### 2. C++动态内存

**栈：**在函数内部声明的所有变量都将占用栈内存。

**堆：**这是程序中未使用的内存，在程序运行时可用于动态分配内存。

通常使用new和delete运算符来申请和删除内存。

如:

```cpp
#include <iostream>
using namespace std;
 
int main ()
{
   double* pvalue  = NULL; // 初始化为 null 的指针
   pvalue  = new double;   // 为变量请求内存
   *pvalue = 29494.99;     // 在分配的地址存储值
   cout << "Value of pvalue : " << *pvalue << endl;
   delete pvalue;         // 释放内存
   return 0;
}
```



### 3.模板

C++模板和Java模板语法很像。

```cpp
// 方法模板
template <typename T>
int compare(const T &left, const T &right)
{
   if (left < right)
   {
      return -1;
   }
   if (right < left)
   {
      return 1;
   }
   return 0;
}

// 类模板
template <class T>
class Stack
{
 private:
   vector<T> elems; // 元素

 public:
   void push(T const &); // 入栈
   void pop();           // 出栈
   T top() const;        // 返回栈顶元素
   bool empty() const
   { // 如果为空则返回真。
      return elems.empty();
   }
};
```

### 4.线程

```cpp
#include <iostream>
#include <cstdlib>
#include <pthread.h>
 
using namespace std;
 
#define NUM_THREADS     5
 
void *PrintHello(void *threadid)
{  
   // 对传入的参数进行强制类型转换，由无类型指针变为整形数指针，然后再读取
   int tid = *((int*)threadid);
   cout << "Hello Runoob! 线程 ID, " << tid << endl;
   pthread_exit(NULL);
}
 
int main ()
{
   pthread_t threads[NUM_THREADS];
   int indexes[NUM_THREADS];// 用数组来保存i的值
   int rc;
   int i;
   for( i=0; i < NUM_THREADS; i++ ){      
      cout << "main() : 创建线程, " << i << endl;
      indexes[i] = i; //先保存i的值
      // 传入的时候必须强制转换为void* 类型，即无类型指针        
      rc = pthread_create(&threads[i], NULL, 
                          PrintHello, (void *)&(indexes[i]));
      if (rc){
         cout << "Error:无法创建线程," << rc << endl;
      }
   }
   pthread_exit(NULL);
}
```
