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
输出使用 cout ,输入使用 cin。C++中输入和输出抽象为流，"<<"">>"可以表示流向，"<<"">>"为运算符重载。代码如下：

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
char name[10] = "lily" 在char数组中的存储方式为 l,i,l,y,\0, _ , _ , _ , _ ,_ 会多存一个\0来表示字符结束。真正在内存中是ASCII编码之后，实际存入bit位

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

union可以声明多种类型，但是只能有一种类型生效，主要是为了节省内存，比如有一个id，类型可以是int，也可以是double，但是如果直接声明成double，则可能造成浪费，所以可以使用union，见代码

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

enum跟java中的enum类似，值默认从0开始依次递增

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
1 
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

##### Function

函数跟java中的方法基本一致，一点不同是C++中需要Function prototype，跟Function Definition，Function prototype是给编译器用的，用来编译的时候能找到这个函数。而Function Definition是具体实现，这两个要对应的上。还有一点不同是，C++的参数或返回值会做自动类型转换，比如
`void sayNum (int);`需要int类型的参数，但是在方法调用的时候，可以传入double类型的1.0作为参数，而java中不行。

```
void sayNum (int);

int main() 
{
	  // 初始化一个variable a，并赋值为1.0
    double a = 1.0;
    cout << "a address is " << &a << endl;
    sayNum(a);
    return 0;
}
// tips： a 和 num 分别是两块不同的内存
void sayNum(int num) { // 初始化一个variable num，并赋值为传入的1.0，并做类型转换
    cout << num << endl;
    cout << "num address is " << &num << endl;
}

输出如下：
a address is 0x7ffee20ae630
1
num address is 0x7ffee20ae5ec

```

Function中如果参数是Array的话，那么传入的值是Array第一个元素的地址，例如：

```
void sayNum (int[]);

int main() 
{
    int num_arr[] = {1,2,3};
    // 打印数组的大小，一个int是4个Byte，所以大小是 3x4=12
    cout << "num_arr size is " << sizeof(num_arr) << endl;
    sayNum(num_arr);
    return 0;
}

void sayNum(int num[]) {
	// 会打印num数组第一个元素的地址的大小，这里打印的unsigned long，占8个Byte
    cout << "function num size is " << sizeof(num) << endl;
}

输出如下：
num_arr size is 12
function num size is 8
```

const关键字，如果在指针之前，那么不可修改指针指向的value，如果在指针名之前，那么不可改变指针保存的地址。

```
int main() 
{
    int age = 18;
    int height = 165;
    const int* p_age = &age;	
    *p_age = 12;					// invalid
    p_age = &height;			// valid
    
    int* const p_int = &age;	
    *p_int = 20;					// valid
    p_int = &height;			// invalid
    return 0;
}
```

string字符串（c的字符串本质是一个char数组，即char[],会默认在最后一个数组元素存一个结束符"\0"，在Function中的处理，在function中可以使用char*来充当字符串参数类型，而且不需要传入数组大小，如下：

```
void sayStr (char*);

int main() 
{
    char a[] = "lily";
    char* b = "lucy";

    sayStr(a);
    sayStr(b);
    sayStr("john");
    return 0;
}

void sayStr(char* something) {
    while (*something) {
        cout << *something << "---";
        something++;
    }
    cout << endl;
}

输出如下：
l---i---l---y---
l---u---c---y---
j---o---h---n---
```

如果需要返回一个字符串，则可以使用new关键字，并将地址使用指针返回：

```
char* build_str(char, int);

int main() 
{
    char* str = build_str('A',18);
    cout << str << endl;
    return 0;
}

char* build_str(char a,int times) {
    char* res = new char[times+1];
    res[times] = '\0';
    while (times-- > 0) {
        res[times] = a;
    }
    return res;
}

输出如下：
AAAAAAAAAAAAAAAAAA
```

struct参数，如果function中有结构体参数的话，那么传入的结构体参数与普通的基本类型参数是一样的，都会copy出一份全新的变量，但是如果结构体很大的话，会浪费内存，并且会耗时，所以为了效率考虑的话，可以传入指针作为变量，代码如下：

```
struct people
{
    int age;
    int height;
};

people getStruct1(people);
void getStruct2(people*);

int main() 
{
    people p = {18,165};
    cout << " p age " << p.age << " , p height " << p.height << endl;
    people p2 = getStruct1(p);
    cout << " p age " << p2.age << " , p height " << p2.height << endl;
    getStruct2(&p);
    cout << " p age " << p.age << " , p height " << p.height << endl;
    return 0;
}

people getStruct1(people p) {
    p.age = 20;
    people p2;
    p2.age = 21;
    p2.height = 168;
    cout << "age changed" << endl;
    return p2;
}

void getStruct2(people* p) {
    p->height = 170;
    cout << "height changed" << endl;
}

输出如下：
 p age 18 , p height 165
age changed
 p age 21 , p height 168
height changed
 p age 18 , p height 170
```

Function指针也可以作为参数调用，有点像Java里的方法回调和kotlin中的Function类型的对象传入，使用方法就是要声明对应方法类型的指针，要明确返回值和传入的参数（即signature），具体使用方法如下：

```
int getHeight(int (*say)(char),char);

int say1(char c);
int say2(char c);

int main() 
{
    cout << "say1 " << getHeight(say1,'a') << endl;
    cout << "say2 " << getHeight(say2,'a') << endl;
    return 0;
}
// 接受一个传入 char 类型的参数并返回一个 int 类型的值的函数指针
int getHeight(int (*say)(char),char value) {
    return say(value);
}

int say1(char c) {
    return c * 1;
}

int say2(char c) {
    return c * 2;
}
```

inline函数，跟普通函数的区别就是，在编译器编译代码的时候，会将inline的函数体中的内容替换到调用函数的位置，而不是通过跳转到函数的地址执行代码后，再跳回来，这么做的好处就是可以减少代码执行时的函数地址切换的时间。但是inline函数的选用要慎重，因为如果inline的函数体代码很多，而且调用该函数的位置很多的话，就会使得代码的体积变大，因为凡是用到inline函数的地方都会被替换；其次就是，如果函数本身很耗时的话，那么相对于切换函数地址的耗时就相对较少了，不适合选用，而如果函数耗时极少，而且代码量小的话，那么就可以考虑使用inline函数做优化了，使用方式如下：

```
inline int sqrt(int a) 
{
    return a * a;
}

int main() 
{
    cout << sqrt(3) << endl;
    return 0;
}
```

Reference类型，使用Reference类型参数，可以避免struct或者class参数，在函数中copy出一份新的，如果想在改变调用者传入的参数的话，传统做法是使用指针，现在我们可以使用Reference完成同样的功能。使用如下：

```
void modify1(people&);
void modify2(people);

int main() 
{
    people a = {18,165};
    modify2(a);
    cout << a.age << endl;
    modify1(a);
    cout << a.age << endl;
    return 0;
}
// 传入的reference，不会copy
void modify1(people& source) {
    source.age = 20;
}
// 传入的是value，会在内部重新copy一份
void modify2(people source) {
    source.age = 20;
}

输出如下:
18
20
```

default argument，在函数声明的时候，可以使用默认参数，这样默认参数就可以不传入而使用默认值了。声明的时候，只需要在prototype中写默认值就可以了，如下：

```
void say(char* word,int times=3);

int main() 
{
    char a[5]; 
    cin >> a;
    say(a);
    cout << "-------------" << endl;
    say(a,5);
    return 0;
}

void say(char* word,int times) {
    int i=0;
    while (i < times) {
        cout << word << endl;
        i++;
    }
}
```

function overloading，方法参数不同（包括类型不同或者参数不同），方法参数即为signature，这样compiler就能知道使用哪个方法了

```
void say(int a, int b);
void say(char a);

int main() 
{
    say(1,3);
    say('a');
    return 0;
}

void say(int a, int b) {
    cout << "function1 " << a << "    " << b << endl;
}

void say(char a) {
    cout << "function2 " << a << endl;
}

输出如下：
function1 1    3
function2 a
```

function template，其实就是java中的泛型，使用方法如下：

```
template<typename T>
void Swap(T& a, T& b);

int main() 
{
    int a = 1;
    int b = 3;
    Swap(a,b);
    cout << " a is " << a << ", b is " << b << endl;
    char c = 'c';
    char d = 'd';
    Swap(c,d);
     cout << " c is " << c << ", d is " << d << endl;
    return 0;
}

template<typename T>
void Swap(T& a, T& b) 
{
    T temp = a;
    a = b;
    b = temp;
}

输出如下：
 a is 3, b is 1
 c is d, d is c
```

##### Memory Models and Namespaces

将Function和Structure放到不同的文件时，需要用到header file头文件，header file可以写入的类型如下：

* Function prototypes
* Symbolic constants defined using #define or const 
* Structure declarations
* Class declarations
* Template declarations 
* Inline functions

header file的使用：在头文件中声明了一个structure，同时声明了function prototype，这样就可以在两个文件中，使用同一个struture了。在header file中使用如下语法，可以避免多次引入同一个struture的问题。

```
#ifndef HEADER_TEST_
#define HEADER_TEST_ 
...
#endif
```

代码如下：

```
文件1 file1.cpp
#include <iostream>
#include "header_test.h"

using namespace std;

void run() {
    people p = {"lucy",21};
    cout << p.name << " age is " << p.age << " run run run "  << endl;
}

文件2 practise.cpp
#include <iostream>
#include "header_test.h"

int main() 
{
    using namespace std;
  
    people p = {"lily",20};
    run();
    cout << p.name << " main file age is " << p.age << endl;
    return 0;
}

文件3 header_test.h	
#ifndef HEADER_TEST_
#define HEADER_TEST_
struct people {
    char* name;
    int age;
}; 

void run();
#endif
```

static 关键字，首先被static关键字修饰的变量，生命周期会一直持续到project执行结束。其次，变量声明的位置是有访问范围的，如果一个variable声明在block内，比如函数或者if判断的语句，那么这个variable就只能在block内被访问，如果一个variable声明在函数外，即放在文件中，那么任何文件都能访问到他，如果再在这个variable前面加上static，那么他的访问范围就被限制到了当前文件

extern 关键字：如果一个值在Function外部声明的，那么他就是一个在所有文件中都能用的变量，使用方法如下（在file1.cpp中声明一个变量，在practise.cpp中使用，则需要在practise.cpp使用extern关键字）：

```
file1.cpp 

#include <iostream>
#include "header_test.h"

using namespace std;
int global = 100;
void run() {
    global = 2;
    people p = {"lucy",21};
    cout << p.name << " age is " << p.age << " run run run "  << endl;
}

practise.cpp

#include <iostream>
#include "header_test.h"
extern int global;		// 在这里使用extern来说明使用全局变量
int main() 
{
    using namespace std;
  
    cout << global << endl;
    return 0;
}
```

namespace：用来隔离文件与文件之间，或者library与library之间，命名相同名字的Function或者variable会出现冲突的问题，如果使用对应namespace的Function或者variable时，需要使用`::`来指定对应的namespace，代码如下：

```
#include <iostream>

namespace Lily {
    int a = 1;
}

namespace Lucy {
    int a = 20;
}

using namespace std;

int main() 
{
    cout << Lily::a << endl;
    cout << Lucy::a << endl;
    return 0;
}

```

using：使用using关键字就可以避免每次在使用某个namespace时，都需要使用`Lily::a`这种方式去指定来。方法就是，在要使用的位置之前，使用`using Lily::a`先声明一下，如果该语句放在全局的变量的位置，那么就是全局变量，如果该语句放在block内，就是局部变量。也可以使用`using namespace Lily`来使用对应namespace中的全部变量，使用如下：

```
#include <iostream>

namespace lily {
    int a = 1;
    int b = 2;
}

namespace lucy {
    int a = 20;
}
// 可以使用这种方式给namespace起别名
namespace girl = lily;
using namespace std;
using namespace lily;
int main() 
{
    cout << a << endl;
    cout << b << endl;
    cout << lucy::a << endl;
    return 0;
}
```

namespace在多文件的项目中的用法如下：

```
// header_test.h 

#ifndef HEADER_TEST_
#define HEADER_TEST_

namespace pers {
    struct person {
        char *name;
        int age;
    };

    void showPerson(person &p);
    void run(int distance);
}

namespace stu {
    using namespace pers;
    struct student {
        person p;
        int grade;
    };

    void showStudent(student * s);
}
#endif

// file1.cpp

#include <iostream>
#include "header_test.h"

using namespace std;

namespace pers {
    void showPerson(person &p) {
        cout << "name is " << p.name << ", age is " << p.age << endl;;
    }
    void run(int distance) {
        cout << "You have run " << distance << " km" << endl;
    }
}

namespace stu {
    void showStudent(student * s) {
        cout << "name is " << s->p.name << ", age is " << s->p.age << ", grade is " << s->grade << endl;
    }
}

// practise.cpp 

#include <iostream>
#include "header_test.h"

using namespace pers;
using stu::student;

int main() 
{
    using stu::showStudent;
    person p = {"lily",21};
    showPerson(p);
    person p1 = {"lucy",22};
    student s = {p1,15};
    showStudent(&s);
    return 0;
}
```

##### Objects and Classes

class and class implementation，class declaration可以放在header file中，而class implementation放在另一个source file中。在类的声明中，同样有权限修饰符，public，protected，private，默认的权限修饰符是private的，功能与java类似。权限修饰符可以将数据的访问隐藏，而将class的声明和实现分别放在不同的文件，也是对类进行了封装，只暴露出接口，而不暴露出实现细节。而且function定义好之后，使用者并不需要关心是如何实现的，如果class的定义者想要改某个function的实现方式了，直接修改实现就可以了，而class的使用者是无感知的，便于维护。在class implementation中（也就是在.cpp的文件中），需要使用`::`来指明实现的方法属于哪个类，因为同一个方法可能会在不同的class中重名。具体代码如下：

```
// 头文件 header.h

#ifndef HEADER_TEST_
#define HEADER_TEST_

#include<string>

class Person {
    // 默认是private
    int id;
    private:
    std::string name;
    int age;
    int height;
    int weight;
    int add(int extra_weight) {
        weight += extra_weight;
        return weight;
    }

    public:
    void run();
    int getHeight();
    int addWeight(int extra_weight);
};
#endif

// 源文件header.cpp

#include <iostream>
#include "header_test.h"

using namespace std;
// 需要使用Person::run()来指明run方法具体属于哪个类
void Person::run() {
    cout << " One person is running" << endl;
}

int Person::getHeight() {
    return height;
}

int Person::addWeight(int extra_weight) {
    return add(extra_weight);
}

// main.cpp

#include <iostream>
#include "header_test.h"

int main() 
{
    Person p;
    p.run();

    return 0;

}
```

constructor and destructor：constructor（构造方法）与java的构造方法类似，如果我们不提供的话，compiler会提供一个默认的无参数的constructor，如果要是我们提供了一个constructor，那么编译器就不会给我提供默认的构造方法了。constructor的声明方法与java类似。与java不同的一点是，destructor是当对象销毁的时候会默认调用的一个方法，一般会在以下场景下被调用：1. 如果创造的对象是一个static storage的对象（比如：static Person p )，那么destructor会在程序结束的时候被自动调用；2. 如果创建的是一个automatic storage的对象（比如 Person p），那么destructor会在声明该对象的方法block结束时被调用；3. 如果使用new来初始化的对象（比如 Person *p = new Person()），那么在调用delete的时候，destructor会被调用。destructor的声明方法为，以Person类为例：~Person()，在前面加一个"~"即可。具体代码如下：

```
// header_test.h

#ifndef HEADER_TEST_
#define HEADER_TEST_

#include<string>
using namespace std;
class Person {
    // 默认是private
    int id;
    private:
    std::string name;
    int age;
    int *height;
    int weight;
    int add(int extra_weight) {
        weight += extra_weight;
        return weight;
    }

    public:
    // 声明默认的构造方法
    Person();
    // 声明带参数的构造方法
    Person(int age,string &name);
    // 声明destructor
    ~Person();
    void run();
    int addWeight(int extra_weight);
};
#endif

// header.cpp

#include <iostream>
#include "header_test.h"

using namespace std;

void Person::run() {
    cout << " One person is running" << endl;
}

int Person::addWeight(int extra_weight) {
    return add(extra_weight);
}

// 构造方法的实现
Person::Person(int age,string &name) {
    this->age = age;
    this->name = name;
    height = new int;
    cout << "constructor is invoked" << endl;
}

// destructor的实现
Person::~Person() {
    cout << "before address is " << height << endl;
    delete height;
    cout << "end address is " << height << endl;
}

// practise.cpp

#include <iostream>
#include "header_test.h"

int main() 
{
    string name = "lily";
    string name2 = "lucy";
    string name3 = "john";
    // 初始化方式1
    Person p = Person(21,name);
    // 初始化方式2
    Person p2(22,name2);
    // 初始化方式3
    Person *p3 = new Person(24,name3);
    p.run();
    p2.run();
}
```

如果直接赋值的话，比如`p = p2`，这句代码会将p2中的内容全部复制到p1的对象里（这里要跟java有所区别，并不是复制地址）。
const修饰成员方法，如果声明的对象为const，则调用的方法也应该为const方法，如下：

```

#ifndef HEADER_TEST_
#define HEADER_TEST_

#include<string>
using namespace std;
class Person {
    ... 
    public:
    // 非const的run()方法
    void run();
   
};
#endif

#include <iostream>
#include "header_test.h"

int main() 
{
    string name = "lily";
    string name2 = "lucy";
    const Person p(21,name);
    // 调用非const的run方法会报错
    // p.run();
    return 0;

}

// 将run()方法修改为如下即可
void run() const;
```

this指针的使用，this指针指向了调用方法的对象，而this的值就是调用方法的对象的地址。比如Person类中有个方法是比较两个Person的年龄，并返回年龄大的那个Person的对象，那么就可以如下实现：

```
Person & Person::getMax(Person &p) {
    if (p.age > age) {
        return p;
    } else {
        return *this;
    }
}
```

Operator Overloading（运算符重载）：顾名思义，就是将运算符的作用重新定义。比如`*`操作符，如果放在某个指针变量前（如p为一个int类型的指针变量，`*p`就可以取出p的地址中的数据）；而如果是在两个操作数中间，则表示乘法（如`int a = b * c;`)。java中是没有运算符重载的。值得注意的一点是，运算符重载只可以重载c++支持的运算符，比如+、-、*、[]等。具体重载的方式就是使用特定的方法形式`
operatorop(argument-list)`，其中`op`就是要重载的操作符，argument-list是传入的参数，比如要重写`+`操作符，则就可以写为`operator+()`的形式。而内部具体的实现则与普通的方法无异，具体代码如下(直接贴出Time.cpp的文件）：

```
// Time.cpp

#include <iostream>
#include "time_header.h"

using namespace std;

void Time::show() {
    cout << "hours is " << hours << ", minutes is " << minutes << endl;
}

Time Time::add(Time &t) {
    int newM = minutes + t.minutes;
    int newH = hours + t.hours + newM/60;
    newM %= 60;
    return Time(newH,newM);
}

Time Time::operator+(Time &t) {
    int newM = minutes + t.minutes;
    int newH = hours + t.hours + newM/60;
    newM %= 60;
    return Time(newH,newM);
}

// practise.cpp

#include "time_header.h"
int main() 
{
    Time t1(1,30);
    Time t2(2,31);
    Time t3 = t1.add(t2);
    // 运算符重载方法可以这样调用
    Time t4 = t1.operator+(t2);
    // 当然最好的方式，就是这样调用
    Time t5 = t1 + t2;
    t3.show();
    t4.show();
    t5.show();
    return 0;

}

输出结果如下：
hours is 4, minutes is 1
hours is 4, minutes is 1
hours is 4, minutes is 1
```

可以重载的运算符如下：

```
+	-	*	/	%	^&	|	~	!	=	<> 	+= 	-= 	*= 	/=	 %=^=	 &= 	|=	 <<	 >>	 >>=<<= 	== 	!=	 <= 	>= 	&&||	 ++ 	-- 	, 	->* 	->()	 [] 	new 	delete 	new []  delete []
```

friend function：一种用来定义非成员方法，而还拥有访问成员变量的方法。一个使用场景就是，在运算符重载的时候，我们必须将有运算符重载的方法放到前面。例如：在Time类中，有一个运算符重载方法`operator+(int minutes)`，那么我们在使用时，方式为`Time t1,t2; t2 = t1+20;`，但是如果我们想写成 ` t2= 20+t1;`则会出问题，因为这句代码会被翻译为`20.operator+(Time t1);`，而很显然int类型的20并没有这个方法，所以这个时候，我就可以使用friend function了，具体如下：

```
// Time.h

#ifndef TIME_H_
#define TIME_H_
class Time
{
private:
    int hours;
    int minutes;
public:
    Time(int h, int m);
    ~Time();
    void show();
    friend Time operator+(int minutes,Time &t);
};
#endif

// Time.cpp
#include <iostream>
#include "time_header.h"

using namespace std;

void Time::show() {
    cout << "hours is " << hours << ", minutes is " << minutes << endl;
}

Time operator+(int minutes,Time &t) {
    int newM = t.minutes + minutes;
    return Time(t.hours,newM);
}

Time::Time(int h, int m)
{
    hours = h;
    minutes = m;
}

Time::~Time()
{
}

// practice.cpp

#include "Time.h"
int main() 
{
    Time t1(1,30);
    Time t2 = 3 + t1;
    t2.show();
    return 0;

}

输出如下：
hours is 1, minutes is 33
```

使用friend function来重载`<<`，当我们使用c++输出时，我们用到了`cout << "1" << endl;`，那么能不能把上面的`show()`方法也改成`cout << t1 << endl;`的形式呢？当然是可以的。同样，我们使用friend function就可以实现，值得注意的是，为了链式调用，我们需要返回一个`ostream`的一个引用，代码如下：

```
// Time.h

#ifndef TIME_H_
#define TIME_H_
#include <iostream>
class Time
{
private:
    int hours;
    int minutes;
public:
    Time(int h, int m);
    ~Time();
    friend Time operator+(int minutes,Time &t);
    friend std::ostream & operator<<(std::ostream & out,Time &t);
};
#endif

// Time.cpp
#include <iostream>
#include "Time.h"

using namespace std;

Time operator+(int minutes,Time &t) {
    int newM = t.minutes + minutes;
    return Time(t.hours,newM);
}

ostream & operator<<(ostream &out,Time &t) {
    out << "hours is " << t.hours << ", minutes is " << t.minutes ;
    return out;
}

// practice.cpp

#include "Time.h"
#include <iostream>

int main() 
{
    using namespace std;
    Time t1(1,30);
    Time t2 = 3 + t1;
    cout << "Time " << t2 << endl;
    return 0;

}
```

Automatic Conversion，自动类型转换，在C++中，是可以进行自动类型转换的，在这点上，C++比Java的自动类型转换更进一步，例如：`int a = 3.3;`，这个代码在C++中是允许的，最终a会被赋值为3，失去0.3的精度。而且C++中不只是基本数据类型可以这样自动转换，我们自己定义的Class也可以实现自动类型转换，规则就是，如果我们定义的构造方法中，**只有一个参数,或者有多个参数，但是只有一个参数没有默认值，其余参数均有默认值**（注意，只有一个参数或者带有默认参数的的方法才可以推断出自动类型转换），比如`Time(int h);`那么`Time t = 3;`这句代码就是合法的，而这句代码实质上是首先会根据数字3来调用`Time(3)`构造一个临时的Time类型的对象，然后将这个临时对象中的hours = 3赋值给t这个对象中的hours变量。同时我们也可以使用explicit来禁止这种隐式的自动类型转换，但是我们还是可以进行显式的自动类型转换。具体代码如下：

```
// Time.h 直接贴出 .h的代码
public:
    Time(int h);
    // Time(int h, int m = 2);

// practise.cpp

#include "Time.h"
#include <iostream>

int main() 
{
    using namespace std;
    Time t = 4;
    cout << "Time " << t << endl;
    return 0;

}

输出为：
Time hours is 4, minutes is 0

假如构造方法前加上explicit关键字
// Time.h
public:
	explicit Time(int h);
则Time t = 4;就是非法的了
但是 Time t = (Time)4；依然是没问题的
```

Conversion Function，如果我们想把我们定义的Class转换成其他类型的话，比如`Time t = Time(2,3); double d = t;`，那么我们就需要加入Conversion Function，其语法为 `operator typeName();`，比如我们要把Time类转换为double，那么定义的方法为`operator double()`，而使用的话，与C++的其他类型推断或者类型强制转换的语法无异。具体代码如下：

```
// Time.h 
    operator double();
    operator int();
    
// Time.cpp
Time::operator double() {
    return hours;
}

Time::operator int() {
    return minutes;
}

// practise.cpp
#include "Time.h"
#include <iostream>

int main() 
{
    using namespace std;
    Time t1(1,30);
    Time t2 = 3;
    // 这里是三种类型转换的形式，第一个是隐式的，后面两种是显式的
    double d = t2;
    // int i = int(t1);
    int i = (int)t1;
    cout << "double is  " << d << endl;
    cout << "int is " << i << endl;
    return 0;

}
输出结果为：
double is  3
int is 30
```

##### Classes and Dynamic Memory Allocation

C++ Primer Plus P628，展示了一个由于编译器自动生成了默认成员方法，导致的细节的问题，直接查看即可。总结一下，在Class中，我们可以使用动态申请内存的方式，去存储数据。比如，我们要存储一个名字，但是名字的长度可能是会变的，所以我们可以使用一个指针变量，在constructor中动态的使用new关键字根据name的长度去申请内存，而不是在编译期的时候，就定死这个name需要占用的内存，从而节省内存，但是切记在destructor中使用delete去释放内存。

通常，C++会动态的生成以下五种成员方法：

1. 如果没有定义constrctor，会生成一个默认的constructor
2. 如果没有定义destructor，会生一个destructor
3. 如果没有定义copy constructor，会生成一个copy constructor
4. 如果没有定义assingment operator，会生成一个assignment operator
5. 如果没有定义address operator，会生成一个address operator

Copy Constructors：当复制一个对象到新创建的对象的时候被使用。比如通过传递value的方式，传递方法参数。Copy Constructor的prototype为：`Class_name(const Class_name &);`，例如：`People(const People &);`，以下几种场景会调用Copy Constructor：
// p是一个People类型的对象
1. People p2(p);
2. People p2 = p;
3. People p2 = People(p);
4. People * ppointer = new People(p)

Copy Constructor本质上做的事，就是把一个对象的成员copy到新的对象的成员上，但是只是非static的成员会copy。但是默认的Copy Constructor是shallow copy（浅拷贝），如变量中使用了string的指针，则只会拷贝地址。例如：

```
People p2 = p
// 就相当于做了如下操作
People p2;
p2.age = p.age;
p2.name = p.name;
```

而如果浅拷贝会造成指向同一个地址的string，被销毁了两次，那么我们可以进行deep copy（深拷贝）

Assignment Operators：如果我们将一个对象赋值到已经存在的对象，那么Assignment operator就会被使用，同样的，如果我们没有显式的提供Assignment Operator，编译器就会帮我们默认生成一个，如下代码：

```
People p(21,"lucy");
People p2(22,"lily");
p2 = p;
```

Assignment operator形式为`Class_name & Class_name::operator=(const Class_name &);
`， 具体的例子比如：`people & operator=(const people & P);`。我们也需要显示的给出Assignment Operator，避免不必要的错误。

```
// people.h

#ifndef PEOPLE_H_
#define PEOPLE_H_

class people
{
private:
    int age;
    char * name;
public:
    people(int age, char * name);
    ~people();
    people(const people & p); // copy constructor
    people & operator=(const people & P); // Assignment Constructor
    people operator+(people & p);
    char * getName();
};
#endif

// people.cpp

#include "people.h"
#include <cstring>
#include <iostream>

using namespace std;

people::people(int age, char* name) {
    this->age = age;
    int len = strlen(name);
    this->name = new char[len+1];
    strcpy(this->name,name);
    cout << "constructor invoked" << endl;
}

people::~people() {
    cout << "deconstructor invoked" << endl;
    delete name;
}

// 如果不进行深拷贝，则会造成已经释放的空间被再次释放
people::people(const people & p) {
    cout << "copy constructor invoked" << endl;
    int len = strlen(p.name);
    this->name = new char[len+1];
    strcpy(this->name,p.name);
}

// assignment constructor 使用深拷贝
people & people::operator=(const people & p) {
    if (this == &p) {
        return *this;
    }
    delete name;
    int len = strlen(p.name);
    name = new char[len+1];
    strcpy(name,p.name);
    return *this;
}

char * people::getName() {
    return this->name;
}

// practise.cpp

#include "people.h"
#include <iostream>

int main() 
{
    using namespace std;
    people p1(21,"lily");
    cout << "p2 name " << p1.getName() << endl;
    people p2 = p1;
    cout << "p2 name " << p2.getName() << endl;
    return 0;

}
```

Static Class Member Function：我们可以在一个Class中声明static的成员方法，有两个结果：低一点，static member function不能被一个对象调用，需要使用类名调用，语法是：`people::getTotal();`。第二点，静态方法中只能使用静态变量。

在Constructor中使用new关键字时，通常要记住以下几点：
1. 如果在Constructor中使用了new，那么就需要在destructor中使用delete
2. new和delete需要相匹配，new和delete 或者 new [] 和 delete [] 各为一对
3. 如果在多个Constructor中使用new时，应该保持一致，使用new 或者使用 new []
4. 在写一个类时，也要注意定义一个copy constructor，并进行深拷贝
5. 通常要需要定义assignment operator，并进行深拷贝，通常需要检查是否是自己在赋值给自己`if (this == &st) return *this;`，而且需要释放这个对象中的指针指向的内存。

> tips：通常空指针可以使用`0`或者`NULL`来表示，在C++11中，可以使用`nullptr`

返回object还是reference，见P662

Pointer的总结：
1. 声明一个指针： `String *strptr`
2. 初始化一个指针到已经存在的对象：`String *strptr = &test[0]`
3. 使用默认构造初始化一个指针：`String *strptr = new String;`
4. 匹配String(const char*)初始化指针：`String *strptr = new String("test");`
5. 匹配String(const String &)初始化指针：`String *strptr = new String(test[0]);`
6. 使用 -> operator访问方法：`strptr->length()`
7. 使用 * deferencing operator获取指向的对象：`*str`

placement new初始化pointer，直接上代码：

```
    char * block= new char[256];
    string * p1 = new string("lily");
    // 将要创建的对象创建到申请的block的内存上
    string * p2 = new (block) string("lucy");
    cout << "p1 -- " << p1 << ", p2 " << p2 << endl;

    string * p3 = new string("lily");
    // 将要创建的对象创建到申请的block的内存上
    string * p4 = new (block) string("lucy");
    cout << "p3 -- " << p3 << ", p4 " << p4 << endl;
```

打印如下：
p1 -- 0x7fa5af402a70, p2 0x7fa5af402970
p3 -- 0x7fa5af402a90, p4 0x7fa5af402970
p2和p4的内存是一样的，因为都将对象新建到了同一个位置，而且在释放内存的时候，如果只是释放了block的内存，string的destructor是不会调用的，这种情况需要手动释放，如下：

```
    char * block= new char[256];
    string * p1 = new string("lily");
    string * p2 = new (block) string("lucy");
    cout << "p1 -- " << p1 << ", p2 " << p2 << endl;

    string * p3 = new string("lily");
    // 使用sizeof，对对象创建的位置偏移
    string * p4 = new (block + sizeof(p2)) string("lucy");
    cout << "p3 -- " << p3 << ", p4 " << p4 << endl;

    delete p1;
    delete p3;
    // 手动调用destructor
    p2->~string();
    p4->~string();
    delete [] block;

```

initializer list syntax: 可以使用初始化列表，在构造方法的内部代码调用前，初始化成员变量。代码如下：

```
// age(a)，变量名加括号，括号内就是要初始化的值，可以跟多个，比如age(a),name(name)
people::people(int a, char* n) : age(a){
    int len = strlen(name);
    this->name = new char[len+1];
    strcpy(this->name,name);
    cout << "constructor invoked" << endl;
}
```

##### Class Inheritance



