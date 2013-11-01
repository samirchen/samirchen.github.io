---
layout: post
title: C/C++函数返回局部变量相关问题
description: 总结了一下C/C++函数返回局部变量的相关问题。
category: blog
---

函数返回局部变量的时候会遇到各种各样的情况，涉及到内存相关的东西时一定要小心是否会出错。

##1、常见栈内变量

一般来说，在函数内对于存在栈上的局部变量的作用域只在函数内部，在函数返回后，局部变量的内存已经释放。因此，如果函数返回的是局部变量的值，不涉及地址，程序不会出错；但是如果返回的是局部变量的地址（指针）的话，就造成了野指针，程序运行会出错，因为函数只是把指针复制后返回了，但是指针指向的内容已经被释放了，这样指针指向的内容就是不可预料的内容，调用就会出错。

	#include <iostream>
	using namespace std;
	
	int fun1() {
	    int i = 1;
	    return i; // OK.
	}
	int* fun2() {
	    int i = 2;
	    int* ip = &i;
	    return ip; // Wrong!
	}
	int main() {
	    int r1 = fun1();
	    cout << r1 << endl; // 1
	    int* r2 = fun2();
	    cout << *r2 << endl; // 这里有可能可以打印出结果：2，看似正确的，但其实是有问题的。这是因为相应的内存还未被覆盖，但这块内存已经是自由的、不被保护的了。
	
	    return 0;
	}


##2、字符串

先看代码：

	#include <iostream>
	using namespace std;
	char* fun3() {
	    char* s = "Hello";
	    return s; // OK.
	}
	char* fun4() {
	    char s[] = "Hello";
	    return s; // Wrong!
	}
	int main() {
	    char* r3 = fun3();
	    cout << r3 << endl; // Hello
	    char* r4 = fun4();
	    cout << r4 << endl; // 内存已经无效的了。打印出乱码。
	
	    return 0;
	}




通过 `char* s = "Hello";` 的方式得到的是一个字符串常量 `Hello`，存放在只读数据段（.rodata section），把该字符串常量的只读数据段的首地址赋值给了指针 `s`，所以函数返回时，该字符串常量所在的内存不会被回收，所以能正确地通过指针访问。
通过 `char s[] = "Hello";` 的方式得到的是局部变量，字符串直接量作为基于栈的字符数组的初始值，这里得到的字符数组 `s` 实际的数据存储: `H e l l o \0`。增加了一个终结符 `\0`。所以函数返回时，栈被清空，局部变量内存清空，返回的是一个已被释放的内存的地址，打印出来的就会是乱码。

##3、静态变量

如果函数的返回值非要是一个局部变量地址，可以把局部变量声明为`static`静态变量。这样变量存储在静态存储区，程序运行过程中一直存在。

	#include <iostream>
	using namespace std;
	int* fun5() {
	    static int i = 5;
	    return &i; // OK.
	}
	char* fun6() {
	    static char s[] = "Hello";
	    return s; // OK.
	}
	int main() {
	    int* r5 = fun5();
	    cout << *r5 << endl; // 5
	    char* r6 = fun6();
	    cout << r6 << endl; // Hello
	
	    return 0;
	}



##4、数组

数组是不能作为函数的返回值的。因为编译器会把数组名认为是局部变量（数组）的地址。返回一个数组，实际上是返回指向这个数组首地址的指针。函数结束后，数组作为局部变量被释放，这个指针则变成了野指针。同1.1的`fun2()`及1.2的`fun4()（字符数组）`。

但是声明数组是静态的，然后返回是可以的，同1.3。

	#include <iostream>
	using namespace std;
	int* fun7() {
	    int a[3] = {1, 2, 3};
	    return a; // Wrong!
	}
	int* fun8() {
	    static int a[3] = {1, 2, 3};
	    return a; // OK.
	}
	int main() {
	    int* r7 = fun7();
	    cout << *r7 << endl; // 内存已经是无效的了。
	    int* r8 = fun8();
	    cout << *r8 <<endl; // 1
	
	    return 0;
	}



##5、堆内变量

函数返回指向存储在堆上的变量的指针是可以的。但是，程序员要自己负责在函数外释放（free/delete）分配（malloc/new）在堆上的内存。

	#include <iostream>
	using namespace std;
	char* fun9() {
	    char* s = (char*) malloc(sizeof(char) * 100);
	    return s; // OK. 但需要程序员自己释放内存。
	}
	int main() {
	    char* r9 = NULL;
	    r9 = fun9();
	    strcpy(r9, "Hello");
	    cout << r9 << endl; // Hello
	    free(r9); // 要记得自己释放内存。
	
	    return 0;
	}


##6、对象
函数内局部对象的引用是不能作为函数返回值的。函数内的局部对象在离开函数作用域后会被自动析构（自动调用其析构函数），则它的引用不再指向一个有效对象，结果无法预测。就类似1.1的`fun2()`返回局部变量地址后，这个地址指向的内存其实已经不再有效了。

函数内用new创建的堆上的对象的指针是可以作为函数返回值的，但是类似1.5要程序员自己负责在函数外释放对象内存。


`classa.h`

	#ifndef CLASSA_H
	#define CLASSA_H
	class ClassA
	{
	public:
	    int x;
	    int y;
	    int z;
	
	public:
	    ClassA(int xx, int yy, int zz);
	    ~ClassA();
	    void printXYZ();
	};
	#endif // CLASSA_H

`classa.cpp`

	#include "classa.h"
	#include <iostream>
	using namespace std;
	ClassA::ClassA(int xx, int yy, int zz) : x(xx), y(yy), z(zz) {
	    cout << "Construct a ClassA" << endl;
	}
	ClassA::~ClassA() {
	    cout << "Deconstruct a ClassA" << endl;
	}
	void ClassA::printXYZ() {
	    cout << x << " " << y << " " << z << endl;
	}

`main.cpp`

	#include <iostream>
	using namespace std;
	ClassA& fun10() {
	    ClassA ca(1, 2, 3);
	    return ca; // Wrong!
	}
	ClassA* fun11() {
	    ClassA* ca = new ClassA(1, 2. 3);
	    return ca; // OK. 但需要程序员自己释放内存。
	}
	int main() {
	    ClassA r10 = fun10();
	    r10.printXYZ(); // 对象在函数返回时已经被析构了，对应的内存已经无效。
	    ClassA* r11 = fun11();
	    r11->printXYZ(); // 1 2 3
	    delete r11; // 要记得自己释放内存。
	
	    return 0;
	}



##小结
总结上面的几个例子，我们只要搞清楚函数返回的东西是什么、其对应的内存存在哪一般就可以搞清楚它能不能被返回，其实有的例子印证的结果是重叠的。

一个完整的程序，内存分布情况：
<img src="/images/function-returns/program-memory-distribution.png" alt="program-memory-distribution">

上面的几个例子中，我们返回的东西包括：

- `栈上变量值`（基本类型）可以作为函数返回值；因为只涉及到传值，不会有什么问题；
- `栈上变量地址`（基本类型指针；数组；对象引用）不能在作为函数返回值；因为它们的作用域只在函数内，函数返回后它们的内存都不再有效；
- `堆上变量地址`（malloc或new得到的指针）可以作为函数返回值；变量存在堆上，其内存从分配后一直有效直到程序员自己释放（free或delete）；
- `静态变量地址`（static）可以作为函数返回值；因为它们存储在静态变量区，其内存整个程序运行期间一直有效；
- `只读数据段`(.rodata)地址（字符串常量）可以作为函数返回值；因为它们存储在只读数据段，其内存整个程序运行期间一直有效；

以上内容，难免有差错纰漏，欢迎大家指正！


[SamirChen]: http://samirchen.com "SamirChen"
[1]: {{ page.url }} ({{page.title}})

