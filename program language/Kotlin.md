## Kotlin语法相关

参考 [Kotlin上手教程](https://rengwuxian.com/tag/kotlin/)

##### 项目中用到的

1. apply plugin: 'kotlin-android-extensions'引入插件不需要findViewById
2. Kotlin与Java互相调用，使用@Nullable使得Kotlin的`?.`生效
3. 使用`=`号可以给一行语句的方法赋值，比如 `override fun getLayoutResId() = R.layout.fragment_subject_classification`
4. when if可以有返回值，Kotlin中没有三木运算符，用if else 代替
5. List是不可添加元素的，MutableList是可以添加元素的
6. 关于internal修饰符，是Module级别的，比如在写SDK的时候，一些方法不想暴露出来，像ActivityThread中，Google会使用@hide来实现，而在Kotlin中，直接使用internal修饰符就可以做到

##### 字符串模版

使用`${}`的方式，同时字符串中的转义字符也会被转移，代码如下：

```
val name = "world"
println("Hi ${name.length}") 

val name = "world!\n"
println("Hi $name") // 这里会多一个空行
```

如果不想过多转义，则可以使用`""" """`的方式，代码如下：

```
val name = "world"
val myName = "kotlin"
    
val text = """
      Hi $name!
    My name is $myName.\n
"""        // 使用"""的方式，\n不会被转义
println(text)

```

输出如下：
      Hi world!
    My name is kotlin.\n

几个注意点：
1. \n 并不会被转义
2. 最后输出的内容与写的内容完全一致，包括实际的换行
3. $ 符号引用变量仍然生效

如果想要对齐，则可以使用trimMargin()，去掉前面的空行

```
val text = """
      |Hi world!
    |My name is kotlin.
""".trimMargin()
println(text)
```

输出如下：

Hi world!
My name is kotlin.

注意的点：
1. | 符号为默认的边界前缀，前面只能有空格，否则不会生效
2. 输出时 | 符号以及它前面的空格都会被删除
3. 边界前缀还可以使用其他字符，比如 trimMargin("/")，只不过上方的代码使用的是参数默认值的调用方式

##### 集合中的几个操作符

`forEach`: 遍历每个元素，代码如下：

```
 val intArrayOf = listOf(1, 2, 3, 4, 5, 6, 7)
 intArrayOf.forEach {
     println("int is $it")
 }
```

filter:对每个元素进行过滤操作，如果 lambda 表达式中的条件成立则留下该元素，否则剔除，最终生成新的集合（不会改变原集合）

```
    val intArrayOf = listOf(1, 2, 3, 4, 5, 6, 7)
    val filter = intArrayOf.filter {
        it > 3
    }

    filter.forEach {
        println("filter $it")
    }
```

map： 遍历每个元素并执行给定表达式，最终形成新的集合（不会改变原集合）

```
val intArrayOf = listOf(1, 2, 3, 4, 5, 6, 7)
    val mapList = intArrayOf.map {
        it + 2
    }

    mapList.forEach { println("int is $it") }
    println("-----------------")
    intArrayOf.forEach {
        println("int is $it")
    }
```

flatMap: 历每个元素，并为每个元素创建新的集合，最后合并到一个集合中

```
    var intArrayList = listOf(1, 2, 3, 4, 5, 6, 7)


    val flatMap = intArrayList.flatMap { i: Int ->
        listOf("${i + i}", "a")
    }

    flatMap.forEach {
        println("it is $it")
    }
```

#####  Sequence

具体参考[Sequence](https://rengwuxian.com/kotlin-basic-3/)

##### ==和===

== ：可以对基本数据类型以及 String 等类型进行内容比较，相当于 Java 中的 equals
=== ：对引用的内存地址进行比较，相当于 Java 中的 ==

```
val str1 = "123"
val str2 = "123"
println(str1 == str2) // 内容相等，输出：true

val str1= "字符串"
val str2 = str1
val str3 = str1
print(str2 === str3) // 引用地址相等，输出：true
```

##### kotlin中的泛型 (待定）

in out 与java中相类似的是 ? super Apple 、? extends Fruit

[Kotlin泛型](https://rengwuxian.com/kotlin-generics/)

##### Lambda和高阶函数

从高阶函数说起。
如果我们在一个方法中调用另一个方法，可以考虑如下做法：

```
int a() {
  return b(1);
}
a();
```

如果我们想要动态的设置b中的参数，可以如下调用：

```
int a(int param) {
  return b(param);
}
a(1); // 内部调用 b(1)
a(2); // 内部调用 b(2)
```

但是如果我们想动态的设置的是方法本身，比如在a中要调用的方法不确定，可能是method1，也可能是method2，比如下面这种：

```
int a(??? method) {
  return method(1);
}
a(method1);
a(method2);
```

在java中是不被允许的。首先java中想要传入一个方法是不可行的，只能以接口的方式传入。代码如下：
```
private static OnclickListener listener1 = new OnclickListener() {
        @Override
        public void onClick(int position) {
            System.out.println("listener1...");
        }
    };

    private static OnclickListener listener2 = new OnclickListener() {
        @Override
        public void onClick(int position) {
            System.out.println("listener2...");
        }
    };

    public static void main(String[] args) {

        View view = new View();
        view.setListener(listener1);
//        view.setListener(listener2);
    }
```

其实本质上只是传过来了一个稍后用来被调用的`onClick(position)`方法，只不过Java中不能传递方法，所以我们只用用一个`interface`包起来。

而Kotlin中，就可以直接传入函数类型，作为一个参数，或者返回值。所谓函数类型，其实是一类类型，一个方法本质上其实就是你给他传入一些值，经过处理之后，它再给你返回一个值。所以，函数类型的定义就可以是`(Int,String) -> Int`，第一个括号中是要传入的参数的类型和数量，`->`的Int是返回值，也可以没有返回值。**这种「参数或者返回值为函数类型的函数」，在 Kotlin 中就被称为「高阶函数」——Higher-Order Functions。**

另外，除了作为函数的参数和返回值类型，你把它赋值给一个变量也是可以的。

不过对于一个声明好的函数，不管是你要把它作为参数传递给函数，还是要把它赋值给变量，都得在函数名的左边加上双冒号才行：

```
    fun method1(position: Int) {
        println("position")
    }

    val a = ::method1
    View().setOnclickLisener(::method1)
```

::的意思，这个符号的意思就是一个函数加了两个冒号，就可以把这个函数才变成了一个对象。

Kotlin 里「函数可以作为参数」这件事的本质，是函数在 Kotlin 里可以作为对象存在——因为只有对象才能被作为参数传递啊。赋值也是一样道理，只有对象才能被赋值给变量啊。但 Kotlin 的函数本身的性质又决定了它没办法被当做一个对象。那怎么办呢？Kotlin 的选择是，那就创建一个和函数具有相同功能的对象。怎么创建？使用双冒号。

在 Kotlin 里，一个函数名的左边加上双冒号，它就不表示这个函数本身了，而表示一个对象，或者说一个指向对象的引用，但，这个对象可不是函数本身，而是一个和这个函数具有相同功能的对象。

比如下面这种用法：

```
fun method1(position: Int) {
    println("position")
}

    method1(1)
    a(1)
    (::method1)(1)
    // 上面这三种调用方法是等价的
```

对象是不能加个括号来调用的，对吧？但是函数类型的对象可以。为什么？因为这其实是个假的调用，它是 Kotlin 的语法糖，实际上你对一个函数类型的对象加括号、加参数，它真正调用的是这个对象的 invoke() 函数：

```
(::method1)(1) // 实际上会调用(::method1).invoke(1)
```

包括双冒号加上函数名的这个写法，它是一个指向对象的引用，但并不是指向函数本身，而是指向一个我们在代码里看不见的对象。这个对象复制了原函数的功能，但它并不是原函数。

lambda基本语法： `{i:Int -> print(i)}`
在传入函数类型的值的时候，可以简化成lambda形式，所以setOnclickListener()可以简化为如下：

```
    View().setOnclickLisener(fun(position: Int) {
        println("position $position")
    })

    // lambda形式
    View().setOnclickLisener({ i:Int ->
        println("position is $i")
    })
```

其实上述lambda还可以继续简化，如下：

```
    // 如果 Lambda 是函数的最后一个参数，你可以把 Lambda 写在括号的外面：
    View().setOnclickLisener() { i:Int ->
        println("position is $i")
    }
    
    // 而如果 Lambda 是函数唯一的参数，你还可以直接把括号去了：
    View().setOnclickLisener { i:Int ->
        println("position is $i")
    }
    
    // 另外，如果这个 Lambda 是单参数的，它的这个参数也省略掉不写：
    // 因为 Kotlin 的 Lambda 对于省略的唯一参数有默认的名字：it：
    View().setOnclickLisener { 
        println("position is $it")
    }
```

之所以可以省略唯一的参数，是因为它的类型是可以推断出来的。调用的函数在声明的地方有明确的参数信息，比如：

```
// 这个参数的参数类型和返回值写得清清楚楚
    fun setOnclickLisener(listener: (Int) -> Unit) {
        this.listener = listener
    }
```

但是，当你要把一个匿名函数赋值给变量而不是作为函数参数传递的时候，就不能省略参数类型了，如下：

```
val b = fun(i:Int) {
        println(i)
}
// 可以省略成lambda形式
   val b = { i:Int ->
        println(i)
    }
// 但是如果省略类型会报错，因为这里是推断不出参数类型的！
    val b = {
        println(it) // 报错
    }
```

另外 Lambda 的返回值不是用 return 来返回，而是直接取最后一行代码的值：

```
    val b = { i: Int ->
        println(i)
        "lily"   // 这里返回的是一个String类型
    }
```

另外因为 Lambda 是个代码块，它总能根据最后一行代码来推断出返回值类型，所以它的返回值类型确实可以不写。

##### Kotlin 里匿名函数和 Lambda 表达式的本质

匿名函数是一个对象，而不是函数，因为只有对象才能传入到参数中或者赋值给变量，他和::加方法名是一种东西。同理，Lambda 其实也是一个函数类型的对象而已。你能怎么使用双冒号加函数名，就能怎么使用匿名函数，以及怎么使用 Lambda 表达式。Kotlin 的 Lambda 跟 Java 8 的 Lambda 是不一样的，Java 8 的 Lambda 只是一种便捷写法，本质上并没有功能上的突破，而 Kotlin 的 Lambda 是实实在在的对象。

总结：
好，这就是 Kotlin 的高阶函数、匿名函数和 Lambda。简单总结一下：
在 Kotlin 里，有一类 Java 中不存在的类型，叫做「函数类型」，这一类类型的对象在可以当函数来用的同时，还能作为函数的参数、函数的返回值以及赋值给变量；
创建一个函数类型的对象有三种方式：双冒号加函数名、匿名函数和 Lambda；
一定要记住：双冒号加函数名、匿名函数和 Lambda 本质上都是函数类型的对象。在 Kotlin 里，匿名函数不是函数，Lambda 也不是什么玄学的所谓「它只是个代码块，没法归类」，Kotlin 的 Lambda 可以归类，它属于函数类型的对象。

##### Kotlin 的扩展函数和扩展属性（Extension Functions Properties）



##### Scope functions 作用域函数

没有带来新的技术的能力，但是会让我们的代码变得更简洁，可读性更高

作用域函数主要有五个：let, run, with, apply 和 also.他们所做的事儿都是在一个对象上执行一段代码（使用lambda形式传入的），区别在于在block内部这个对象是以什么形式使用的，以及最后的返回值是啥。

各个作用域函数区别

|Function|Object reference|Return value|	Is extension function|
|---|---|---|---|
|let|it|Lambda result|yes|
|run|this|Lambda result|yes|
|run|-|Lambda result|	No: called without the context object|
|with|this|Lambda result|No: takes the context object as an argument.j|
|apply|this|Context object|yes|
|also|it|Context object|yes|

官方给出的选择建议：
1. 如果在一个非空的对象上执行lambda，可以使用 let，比如项目中的 RandomTestPagerActivity
2. 对象的配置，使用 apply，比如项目中的 CourseClassDetailActivity中
3. 对象配置，并返回一些计算的结果，使用run 
4. 需要一些额外的效果，使用also，暂时没用到
5. 在一个对象上成组调用，使用with

上述各个函数的使用，其实还是有重叠的，具体的还是看自己怎么选。

当然官方建议在何时的时候使用即可，避免过度使用，因为会加大代码的可读性，而且可能导致一些问题。要避免嵌套使用，因为极其容易混淆 this 和 it

使用with的场景：with can be read as “ with this object, do the following.当调用某个对象的一些方法，而不需要返回值的时候，建议使用with


