## C++相关笔记

参考 [C++ primer plus](https://item.jd.com/11676692.html)

### C++概述

1980年Bjarne Stroustrup在Bell Labs创建了C++，C++可以说是一个包含了C的功能高级语言，C是一个面向过程的语言，C++则是既包含了C的面向过程，又加入了面向对象和泛型编程。Bjarne Stroustrup发明C++的哲学就是**making C++ useful than with enforcing particular programming philosophies or styles**，注重实用。之后为了能让C++在各个OS保持一致性，ANSI（American National Standards Institute）和ISO（The International Organization for Standardization）对C++的标准化进行了一系列的工作，形成了很多标准，比如C++98，C++03等等。

C++相较于机器语言，可以跨平台运行而不需要修改源码，原理就是在不同平台运行时，使用不同平台的编译器对C++源码进行编译，从源码到可执行代码的大致过程如下：

```
hello.cpp --> compiler --> hello.o --> linker --> hello.out
源代码--> 经过编译器编译 -->变成object code-->经过链接器-->最终变成可执行代码
```

### C++语法

##### 头文件/namespace/io

有些功能是C++的library已经定义好的，我们可以直接拿来用，但是需要include一个头文件。
同时，为了避免在使用功能函数时混淆的问题，则使用了 using namespace 的方式，类似于Java的包名。
io功能需要引入头文件`#include <iostream>`，同时需要`using namespace std`，如果不写`using namespace std`，则调用的时候，需要写全名。
输出使用 cout ,输入使用 cin。C++中输入和输出抽象为流，"<<"">>"可以表示流向，"<<"">>"为运算符重栽。代码如下：

```
#include <iostream>

int main() {
	// using namespace方式
    using namespace std;
    // 字符串流入 cout
    cout << "hello c++" << endl;
	// 使用全名方式
    std :: cout << "intput your name plz: " << endl;

    char name[] = {};
	// 输入的字符串流入name字段
    cin >> name;
    cout << "your name is " << name << endl;
}

输出如下：
hello c++
intput your name plz: 
lily 		// 键盘录入lily
your name is lily
```

##### 基本类型

char/short/int/long/long long/float/double，各种类型还有对应的unsigned类型，即无符号类型。默认的都是有符号类型。两者的区别就是表示范围不一样，比如short是 -32768~32767，而unsigned short是0~65535

```
    char c = 'a';
    short s = 1;
    int i = 2;
    long l = 2000000000;
    long long ll = 20000000009999;
    float f = 1.0f;
    double d = 1.0;
    unsigned short us1 = 1;
    unsigned short us = -1; // unsigned short如果赋值为-1，那么打印出来就是65535

    cout << us << endl;
```

##### Compound类型

Array类似于java中的Array,Array其实也可以理解为一种特殊的指针，比如 `int nums[3] = {1,2,3}`，我们可以使用nums[0]来访问第一个元素，也可以使用*nums来访问第一个元素，这是因为nums这个名字代表的就是nums[0]位置的内存地址

```
int main() 
{
    int nums[3] = {1,2,3};
    cout << *nums << " second is " << nums[1] << endl;
}
```

struct可以将不同的类型打包到一块儿，与java中的class有类似的地方，但是完全不是一种东西，struct中不能有方法等等。

```
int main() 
{
	// 两种不同的初始化方式
    struct people {
        char name[10];
        int age;
    } teacher = {
        "john",30
    };

    people a = {"lily",18};
    cout << a.name << " a age is " << a.age << endl;
    cout << teacher.name << " teacher age is " << teacher.age << endl;
}
```

union可以声明多种类型，但是只能有一种类型生效，主要是为了节省内存，比如有一个id，类型可以是int，也可以是long，但是如果直接声明成long，则可能造成浪费，所以可以使用union，见代码

```
int main() 
{
    union id {
        int id_int;
        double id_d;
    };

    id a;
    a.id_int = 15;
    a.id_d = 14.3;
    // 只能互斥的赋值，所以打印如下结果
    cout << a.id_int << "    " << a.id_d << endl;
}

输出如下：
-1717986918    14.3
```

enum跟java中的enum类似，值默认从0开始一次递增

```
int main() 
{
    enum colors {
        red, orange, yellow
    };

    colors c = orange;

    cout << c << endl;
}
输出 
```

pointer可以说是C++的灵魂，指针是用来保存内存地址的，普通的变量可以有变量名，使用变量名就可以访问变量值，但是如果是新申请的一段内存，是没有名字的，这个时候就可以使用pointer类型来对这片内存进行跟踪。在C++中可以使用new和delete来申请和释放内存。同时，指针也是有类型，什么样的指针就保存什么样的类型的内存地址。如果指针指向的是结构体类型，则可以使用 -> 来获取变量，具体使用见代码。

内存分为：Automatic Memory，Static Memory 和 Heap，Automatic Memory即在方法中声明的变量等等开辟的内存，随着function执行的结束而释放；Static Memory可以在方法外申明或者使用static关键字声明，生命周期最长，随着程序的结束而消失；Heap则是使用new 和 delete管理的内存，需要programer自己管理。

```
int main() 
{
    int i = 1;
    int* pi = &i;

    cout << *pi << endl;

    int* arr = new int[8];
    arr[0] = 1;
    *(arr+1) = 2;

    cout << *arr << "     " << arr[1] << endl;

    struct people {
        int age;
        int height;
    };

    people s1;
    people* ps = &s1;
    ps->age = 18;

    cout << ps->age << "      " << s1.age << endl;

    return 0;
}

输出如下：
1
1     2
18      18
```


