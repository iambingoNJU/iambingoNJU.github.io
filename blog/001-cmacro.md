# 关于c宏的经典使用样例

从定义上面看，**宏**(macro)是个很简单的概念，就是**替换**嘛！但是，在实际中，宏有着很强大的用处。笔者最近查了一些资料，总结了一下c中宏的经典使用场景，下面一一作出介绍：

## case 1 -- 常量定义
这个应该是我们当初在c语言入门的时候接触宏时最先使用的样例吧。比如：
```c
#define PI 3.1416
```
这样写的好处就是在“一处定义，多处使用”的场景下，如果将来要修改PI的真值的话，只需要修改这个宏定义，其它使用PI的值的地方都是使用符号PI，可以自动扩展为宏定义的真值。

另外一个好处就是可读性要好，想一想，读代码的人在看到一个有意义的名字的时候比看到一个不懂是什么含义的数字要更亲切吧，不过前提是命名要命好！！！

## case 2 -- 内联函数定义
利用带参数的宏定义，可以实现类似函数的功能，比如：
```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
```
但是与普通函数调用不同的是，普通的函数调用的时候会产生**栈帧**（stack frame），而带参数的宏则不会，只是简单的替换，相当于内联函数，不会导致函数调用产生的额外开销，因此在实现比较小的功能的时候使用宏比较合适。

但是使用宏代替函数的时候也会导致一些问题。首先，要注意适当添加括号；比如上例中，如果这样定义：
```c
#define MAX(a, b) a > b ? a : b
```
则在实际使用的时候很可能出问题。如果某个使用者写了这样一句：
```c
 int x = MAX(3, 2) + 4;
```
展开后变成了这样:
```c
int x = 3 > 2 ? 3 : 2 + 4;
```
这样，使用者期望x的值是7，但实际x的值却为3。

其次，这种形式的替换会造成同一表达式的多次计算。还是上面那个例子:
```c
int x = MAX(3 + 2, 4);
```
这时3 + 2会被计算两次，尽管编译器在一定程度上会帮我们进行优化，但是写出这样的代码还是不如人意的。

更为严重的是，如果传参的时候传一个带副作用的表达式，还会造成逻辑上的错误。比如这样调用：
```c
int a = 7, b = 5;
int x = MAX(++ a, b);
```
如果是普通函数定义的话，x的值应该为8。但是在这里，经过宏展开后，变成这样的：
```c
int a = 7, b = 5;
int x = ((++ a) > (b) ? (++ a) : (b);
```
最后x的值变为了9。

但是用宏实现的函数也有一些好处。
第一，上述的MAX宏实际上可以认为是一个重装函数，它可以针对int, double, float, char各种类型进行操作，而且利用c的隐式类型转换规则，MAX宏的两个参数还可以是不同类型的变量。这实际上实现了c++中的泛型函数的概念。

第二，c标准中预定义了一些宏：\__DATE__, \__TIME__, \__LINE__, \__FILE__, \__func__, \__VA_ARGS__。其中前四个变量是与编译时刻相关的，\__DATE__是编译的日期，\__TIME__是编译的时间，\__LINE__是当前被编译在文件中的行数，\__FILE__是当前被编译的文件名。\__func__是当前所在的函数，\__VA_ARGS__是宏中的可变参数(即“...”)。
利用这些变量，我们可以实现一些诊断函数，用来输出一些调试信息。比如，我们可以实现自己的Log函数：
```c
#define Log(format, ...) \
		printf("\33[1;34m[%s,%d,%s] " format "\33[0m\n", \
				__FILE__, __LINE__, __func__, ## __VA_ARGS__)
```
> 注意\__VA_ARGS__前面的##。当format中没有带格式化的参数的时候，那么后面的可变参数为空，这样经过宏扩展后，后面会多出一个逗号，这里##便可去掉这个逗号，保证这个Log宏在各种情况下都可以工作。

## case 3 -- 字符串操作#和##
在宏定义中有两个很有意思的操作符#和##，利用它们，我们可以实现一些很强大的功能。
先来看看#的简单应用：
```c
#define EVAL(expr) printf(#expr " = %d\n", expr)
```
当我们这样调用时：
```c
EVAL(3 + 5);
```
则会输出：
```c
3 + 5 = 8
```
实际上，#的功能就是把宏参数字符串化，也就是说，把宏参数替换为双引号引用的宏参数。在这里，3 + 5被替换为了\"3 + 5"。

再看一个##的应用：
```c
#define concat_temp(x, y) x ## y
#define concat(x, y) concat_temp(x, y)
#define concat3(x, y, z) concat(concat(x, y), z)
#define concat4(x, y, z, w) concat3(concat(x, y), z, w)
#define concat5(x, y, z, v, w) concat4(concat(x, y), z, v, w)
```
这里concat函数的作用就是将多个标识符拼接起来，一般用于定义函数的时候拼接多个标识符形成函数名。比如：
```c
#define INSTR add
int concat3(INSTR, _, r2rm)(int arg) {...}
```
经过宏替换之后，变成了：
```c
#define INSTR add
int add_r2rm(int arg) {...}
```
因此，##的作用时将两个标识符拼接在一起。
> 你们应该会注意到上面的concat定义中出现了一个concat_temp,这个宏有什么作用呢？为什么不直接这样定义呢？
> ```c
> #define concat(x, y) x ## y
> ```
> 因为上述这种简单的定义在有些情况下会出问题。比如：
> ```c
> #define INSTR ADD
> int concat(func, INSTR)(int arg);
> ```
> 经过宏替换之后会变成这样：
> ```c
> #define INSTR ADD
> int funcINSTR(int arg);
> ```
> 声明的函数名并不是我们期待的funcADD，这是因为我们的concat的实参中还含有一个宏，这个宏在遇到##的时候并不会再次展开，而多数情况下，我们却期待通过调用这个函数会自动展开参数中的宏。
> 因此，这种两层定义的宏就可以实现我们期望的功能。还是上面的concat(func, INSTR)这个例子，第一次宏展开后，变成了concat_temp(func, INSTR),此时并没有##或#的出现，因此会继续将INSTR展开，变为了concat_temp(func, ADD), 最后一次扩展后变为了funcADD，也就是我们期待的结果。

> 不仅##会有这个问题,#也会有这个问题,因此在我们定义一个STR宏的时候，这样定义：
> ```c
> #define STR_TEMP(x) #x
> #define STR(x) STR_TEMP(x)
> ```
> 道理和##的完全一样。

## case 4 -- do {...} while(0)
这种用法比较常见。比如我们自己定义一个Assert函数：
```c
#ifndef NDEBUG
#define Assert(cond, ...) \
	do { \
		if(!(cond)) { \
			fflush(stdout); \
			fprintf(stderr, "\33[1;31m"); \
			fprintf(stderr, __VA_ARGS__); \
			fprintf(stderr, "\33[0m\n"); \
			assert(cond); \
		} \
	} while(0)
#else
#define Assert(cond, ...) 
#endif
```
循环体中的语句只会执行一次，那么这里为什么要用do {} while(0)呢？
一般这么用的情况都是在一个宏定义中出现多条语句，那我们来分析一下为什么要这么用。
首先，我们这样定义：
```c
#define SS \
	stmt1; \
	stmt2;
```
假如这样使用呢？
```c
if(cond)
	SS;
stmt3;
```
宏展开后，变成了：
```c
if(cond)
	stmt1; stmt2;;
stmt3;
```
这样，不管cond是真是假，stmt2语句都会执行。而我们自己的意图肯定是，只有cond为真的时候，stmt1和stmt2才会执行。
我们看到，造成上述结果的原因是调用者希望宏中的多条语句是绑定在一起执行的，但是这样定义的话，其它语句的存在会将他们分隔开来。
既然如此，这样定义可以吗？
```c
#define SS { \
	stmt1; \
	stmt2; \
}
```
直接想似乎也想不出什么问题，但是在下面这种情况下，会出问题的：
```c
if(cond)
	SS;
else
	stmt3;
```
这样会编译出错的。因为宏展开的结果为：
```c
if(cond)
	{ stmt1; stmt2; };
else
	stmt3;
```
出错的原因是else前面多一个分号。有人说，我在使用SS的地方后面不加分号不就行了嘛。确实可以，但是在c中通常我们习惯性的会在语句后面加一个分号，如果将来别人利用你的代码的时候，他发现了这个问题，肯定会抱怨你的。

鉴于上面的这些原因，就有人想出了do{}while(0)式的用法：
```c
#define SS \
	do { \
		stmt1; \
		stmt2; \
	} while(0)
```
这种定义中while(0)后面是需要一个分号的,我们习惯性会加上的分号就会补充在这里。

## case 5 -- X macro
现在介绍高级一点的用法。这中用法的名字不是我起的，别人家发明的，这里只是拿来介绍一下。
举个例子吧。假设给定你枚举类型color:
```c
typedef enum { Red, Green, Blue, Cyan, Magenta, Yellow } Color;
```
现在要你写一个函数，函数的功能是将枚举类型的颜色转换成对应的字符串表示，比如，Color2Str(Red)应该返回"Red"，最简单的，想都不用想的写法：
```c
char* Color2Str(Color c) {
	switch(c) {
		case Red:	return "Red";
		case Green:	return "Green";
		case Blue:	return "Blue";
		case Cyan:	return "Cyan";
		case Magenta:	return "Magenta";
		case Yellow:	return "Yellow";
		default:	return "<unkown>";
	}
}
```
假设你还要实现另外一个函数，功能是给定一种颜色，你要返回与之相对的颜色。于是你可以这样写：
```c
Color GetOppositeColor(Color c) {
	switch(c) {
		case Red:	return Cyan;
		case Green:	return Magenta;
		case Blue:	return Yellow;
		case Cyan:	return Red;
		case Magenta:	return Green;
		case Yellow:	return Blue;
		default:	return c;
	}
}
```
这样写也没什么问题。但是每次添加一个函数的时候，你都要遍历所有的颜色，对每个颜色都要编写差不多的东西，你不觉得很烦嘛。有没有什么简单一点的办法呢？当然有！
首先，我们先把颜色相关的东西抽象出来，单独放在一个文件中：
file: color.h
```c
X(Red, Cyan)
X(Green, Magenta)
X(Blue, Yellow)
X(Cyan, Red)
X(Magenta, Green)
X(Yellow, Blue)
```
有了这个文件，我们可以这样定义上面两个函数：
```c
char* Color2Str(Color c) {
	switch(c) {
		#define X(color, opposite) case color: return #color;
		#include "color.h"
		#undef X
		default:	return "<unkown>";
	}
}

Color GetOppositeColor(Color c) {
	switch(c) {
		#define X(color, opposite) case color: return opposite;
		#include "color.h"
		#undef X
		default:	return c;
	}
}
```
这样写，不仅逻辑更加清晰，可扩展性也更好，以后想要另外再实现新功能时，直接重新定义X宏即可。
甚至我们还可以直接用这个来定义这个枚举类型：
```c
typedef enum {
	#define X(color, opposite) color,
	#include "color.h"
	#undef X
	NUMBER_COLOR
} Color;
```
这样，以后我们想要添加新的颜色的时候，只要更改color.h文件中的内容即可。
结合上面的例子，可以归纳出X macro的通用框架：
```c
#define macroname(arguments)
#include "filename"
#undefine macroname
```
学会了吧，赶紧想想以前写的东西能不能用起来吧。

## case 未完待续 ...