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
bool类型true or false，在C++中，0表示false，非0表示true
char name[10] = "lily" 在char数组中的存储方式为 l,i,l,y,\0, _ , _ , _ , _ ,_ 会多存一个\0来表示字符结束

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

##### 循环/单个char读取/typedef来定义别名

for循环，while循环和do while循环与java类似。不同的一点就是，条件判断中是可以放入数字的，如果是非零就判断为true，如果为0则判断为false

```

int main() 
{
    for (int i=0;i<10;i++) 
    {
        cout << "count " << i << endl;
    }

    int i = 0;
    while (i<10) 
    {
        cout << "count " << i << endl;
        i++;
    }

    int j=0;
    do 
    {
        cout << "count " << j << endl;
        j++;
    } while (j<10);
    // 直接将数字放入到条件判断也是可行的，知道count变为0为止
    int count = 10;
    while (count) {
        cout << count-- << endl;
    }
    return 0;
}

```

双重for循环和二维数组与java一样

```
int main() 
{
	// 初始化二维数组
    int two_dimension_arr[2][3] = {
        {1,2,3},
        {4,5,6}
    };
    for (int i=0;i<2;i++) {
        for (int j=0;j<3;j++) {
            cout << two_dimension_arr[i][j]<< "\t";
        }
        cout << endl;
    }
    return 0;
}
```

可以使用cin.get()和cin.get(char)来读取单个字节，cin.get()使用方式为 ch = cin.get()，当读取到文件末尾时返回EOF标识符（值为-1）；cin.get(char)的方式为 cin.get(ch)，当读取到文件末尾时，返回false

```
int main() 
{
    char ch;
    cout << (ch=cin.get()) << endl;
    // cin.get(ch);
    // cout << ch << endl;
    return 0;
}

```

使用typedef可以来给类型定义别名，之后就可以使用这个别名来表示类型了

```
int main() 
{
    typedef int lily;
    lily a = 1;
    cout << a << endl;
    return 0;
}
```

##### 逻辑操作符和cctype

`if` `if else` `if else if else` `|| && !`都与java一致，特殊的是c++可以使用

| operator | Alternative Representation |
| --------| ---------------------------|
|   `||`  |            or              |
|   `&&`  |           and              |
|   `!`   |           not              |

可以使用字符来表示操作符

c中有一些便捷的判断字符的方法，需要`#include <cctype>`，方法如下：

| Function Name | Return Value |
| ------------- | ------------ |
|isalnum()      | 如果是letter或者 digital 则返回true|
|isalpha()      | 是否是字母|
|isblank()      | 是否是空格|
|iscntrl()      | 是否是ctrl|
|isdigit()      | 是否是数字 0～9|
|isgraph()      | 除了空格以外所有可打印的字符|
|islower()      | 是否小写字母|
|isprint()      | 包括空格的可打印字符|
|ispunct()      | 是否是符号|
|isspace()      | 是否是空白，包括a space, formfeed, newline, carriage return, horizontal tab, vertical tab|
|isupper()      | 是否大写字母|
|isxdigit()      | 是否是16进制中的字符|
|tolower()      | 转小写|
|toupper()      | 转大写|

三木运算符 expression1 ? exp2 : exp3 与java一致

switch语句，与java类似

```
int main() 
{
    int i;
    cin >> i;
    switch (i)
    {
    case 1:
        cout << "test" << endl;
        cout << "something" << endl;
        break;
    default:
        break;
    }
}
```

##### 文件I/O

文件I/O需要 fstream 头文件，同时需要声明一个ofstream类型的变量，并且需要open(char)来关联一个文件，使用完毕以后需要close(),其余操作与cout类似，如下：

```
#include <iostream>
#include <fstream>
using namespace std;
// 输入文件的代码
int main() 
{
    ofstream outFile;
    outFile.open("test");
    char name[50];
    cin.getline(name,50);
    outFile << name;
    outFile.close();
    return 0;
}

// 输出文件中内容的代码
#include <iostream>
#include <fstream>
#include <cstdlib>
using namespace std;

int main() 
{
    ifstream fin;
    fin.open("test");
    char test[50];
    fin.getline(test,40);
    cout << test << endl;
    fin.close();
    return 0;
}
```

