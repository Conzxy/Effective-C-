# 用consts、enums、inlines替代#define
#define是一个预处理指令，表示一个宏，它不被视为语言的一部分，这正是问题所在。

## #define的问题
```cpp
#define ASPECT_RATIO 1.653
```
我们要知道，记号ASPECT_RATIO可能未被编译器看到，或者在编译器处理源码之前就被预处理器移走，于是，该记号没有进入记号表（symbol table）。于是导致莫名奇妙的编译错误，错误信息可能会提示1.653，如果这个定义在别人的头文件中，那你是很难追踪的。  
当然，还可能出现在记号式调试器中，也是因为没有进入到记号表中。
## 用const代替#define
解决上述问题的常见方法：
```cpp
const double AspectRatio=1.653;
```
作为语言常量，AspectRatio是肯定被编译器看到的。
## const的特殊情况
### 1.常量指针
常量定义式通常放在头文件中①，因此有必要将指针声明为const。常量的char*-based字符串
```cpp
const char* const authorname="Scott Meyers";
```
这个如何解读？  
写成`char const*const`自然明白，从右往左读就是a const pointer to const char，前面的const修饰char，后面的const修饰*。  
当然，C++有string，这里定义成以下形式更好：
```cpp
const std::string authorname="Scott Meryers";
```
### 2.class 专属常量
static成员：为了将常量的作用域限制在类中，让它成为member，同时确保该常量具有一份实体让它成为一个static成员②。
```cpp
class GamePlayer{
private:
  static const int NumTurns=5;  //常量声明式
  int scores[NumTurens];
  ....
};
```
注意NumTurns是声明式而不是定义式！  
> 通常C++要求你对你所用的任何东西提供一个定义式，但如果它是个class专属变量又是static且为整数类型（integral type，如ints,chars,bools），则需特殊处理。只要不取它们的地址，你可以声明并使用它们而无须提供定义式。

但是如果你要取地址或者你的编译器坚持要看到定义时，那么提供定义式如下：
```cpp
const int GamePlayer::NumTurns;
```
这个定义式请放入实现文件而非头文件③。你可能会疑问定义式为什么不设初值？那是因为声明时已经获得初值了。

由2可以引出一个#define与const的不同：前者无法创建一个class专属常量。  
#define不重视作用域，在宏定义后的编译过程中都可以看见它（除非#undef），这意味着宏没有封装性（即无private）

## enum
我们首先要了解：“一个属于枚举类型的数值可权充ints被使用”，因此如果你的编译器错误地不允许“static整数型class常量”完成“in-class初值设定”，可以改用枚举类型补偿。
```cpp
class GamePlayer{
private:
  enum{NumTurns=5};

  int scores[NumTurns];
  ...
};
```
这里介绍枚举类型纯粹是实用主义，而且枚举类型有点类似#define，比如不能取一个enum的地址。

## 宏带来的另一个麻烦：文本替换
我们可能会写出如下宏：
```cpp
#define CALL_WITH_MAX(a,b) f((a)>(b)?(a):(b))
```
尽管这不会招致函数调用带来的额外开销，但这种函数形式的宏有着太多的缺点，首先你必须要注意小括号(因为文本替换)，即使这个处理好，文本替换带来的问题还没完，如下：
```cpp
int a=5,b=0;
CALL_WITH_MAX(++a,b);     //a累加2次
CALL_WITH_MAX(++a,b+10);  //a累加1次
```
因为文本替换，所以++a>b和++a都算一次，取决于“被拿来和谁比较”，这可能不是我们希望的。  
我们不需要为这种事烦恼，我们可以写出template inline函数来获得宏带来的效率。
```cpp
template<typename T>
inline void callwithMax(const T& a,const T& b)
{
  f(a>b?a:b);
}
```
这样我们就不用考虑为参数加上括号以及担心参数被核算多次等问题。
## 总结
有了const**s**、enum**s**和inline**s**，我们对预处理器（#define）的需求降低了，但#include、#ifdef/#ifndef仍然需要。

------
①：对于const namespace scoped object来说链接属性是interal linkage，所以常量定义式不会引发重复定义，再加上是常量，方便被不同的源代码包含。  
②：对于static数据成员，无论const都是external linkage，我们可以把static数据成员看作全局变量，只不过作用域不同，。  
③：防止重复定义，像全局变量一样定义是在实现文件中
