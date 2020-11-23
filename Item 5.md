# 了解C++默默编写并调用了哪些函数
## 自动生成的函数类型

当你自己没有声明任何函数时，C++会自动给你声明：

* default constructor

* copy constructor

* copy assignment operator

* destructor

这些函数唯有在被调用时才会被编译器创建出来

注意，编译器生成的函数都不是virtual（这方面请看`Item 7`）

## 无法自动生成的情形

* 在“内含引用成员或const成员”的类中请提供自己的拷贝赋值和拷贝构造

* 基类如果拷贝赋值是`delete`或是`private`的则拒绝为派生类生成拷贝赋值

## 三/五法则

* 如果这个类需要一个析构函数，我们几乎可以肯定它也需要它需要一个拷贝构造函数和拷贝赋值运算符重载

* 如果需要一个拷贝构造函数，几乎肯定也需要一个拷贝赋值运算符重载，反之亦然

* 但是无论需要拷贝构造函数还是拷贝赋值运算符重载都不意味必须要析构函数