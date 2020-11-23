# 考虑写出一个不抛异常的swap函数
swap作为STL的一部分，也是异常安全性能编程的脊柱以及用来处理自我赋值的常见机制。
## 缺省swap（std::swap）
我们先来看看缺省版本的swap吧：
```cpp
namespace std{
  template<typename T>
  void swap(T& a,T& b)
  {
    T temp(a);
    a=b;
    b=temp;
  }
}
```
类型T需要支持copying（因为Meyers写书的时候还没有移动语义，所以书上没提，其实支持moving也是可以的，这里不作区分）。  
这个缺省版本有三次复制，因此在某些情况下相当鸡肋。
其中最主要的是“以指针指向一个对象，内含真正数据”，即所谓的pimpl手法。
```cpp
class WidgetImpl{
public:
  ...
private:
  int a,b,c;
  std::vector<double> v;
};

class Widget{
public:
  Widget(const Widget&);
  Widget& operator=(const Widget& rhs)
  {
    ...
    *pImpl=*(rhs.pImpl);
    ...
  }
private:
  WidgetImpl *pImpl;
};
```
如果我要交换两个Widget对象，那么将会复制3次Widget对象和3次WidgetImpl对象，这不是一件划算的事，因为我想要的只是交换Widget内的指针罢了。  
因此我们要定义自己的swap函数。
## 特化swap
我们可以特化std内的swap来实现我们想要的操作：
```cpp
namespace std{
  template<>
  void swap<Widget>(Widget& a,Widget& b)
  {
    swap(a.pImpl,b.pImpl);  //还通不过编译
  }
}
```
这里稍微介绍下全特化和偏特化：  

特化类型|属性|使用模板
---|---|---
全特化|具体类型限死|类模板、函数模板皆可
偏特化|有多个类型，限定一部分|仅类模板（函数模板通过重载实现）

所以上面这个swap是std::swap的全特化版本，我们要注意一点：**尽管我们不能改变std命名空间的任何东西，但是可以为标准template制造特化版本以适应自己的classes。**
当然，上面这个代码是通不过的，因为pimpl手法运用的就是private指针，那么我们要不声明为friend？不，这里我们选择member：
```cpp
class Widget{
public:
  ...
  void swap(Widget& rhs)
  {
    using std::swap;  //这个是必要的
    swap(pImpl,rhs.pImpl);
  }
  ...
};
namespace std{
  template<>
  void swap<Widget>(Widget& a,Widget& b)
  {
    a.swap(b);
  }
}
```
为什么这样写主要是为了和STL容器保持一致性，因为所有STL容器也都是提供public swap成员和std::swap特化版本。
## 参数化
正如前面所说，我们无法提供一个偏特化的函数模板，因此只能重载，因此：
```cpp
namespace std{
  tempalte<typename T>
  void swap(Widget<T>& a,Widget<T>& b)
  {
    a.swap(b);
  }
}
```
但是标准委员会规定不准膨胀那些已经声明好的东西，因此我们最好不要添加新的东西到std中。那么我们就用non-member去调用member即可：
```cpp
namespace Widgetstuff{
  ...
  template<typename T>
  class Widget{...}; //内含swap成员函数
  ...
  template<typename T>
  void swap(Widget<T>& a,Widget<T>& b)
  {
    a.swap(b);
  }
}
```
namespace不是必要的，但是最好加上，没必要让挤进global namespace。
## using std::swap的必要性
现在swap有几个版本了？
* 缺省版本
* std特化版本
* 模板重载（T专属）

我们这里注意一种情况：将swap用于赋值运算符的重载，不用using显式声明std::swap的话，那么第一时间找到的就是member版本的swap，同时此时不会进行ADL名字查找（因为成员函数进入查找集了），也就是说我们想要的non-member swap会被忽视。  
我们想用的到底是哪个版本？通过using声明将std::swap曝光，可以强迫进行ADL名字查找（这是一份很好的保证），若参数T命名空间中有T专属的swap，则调用它，若没有则使用std内的swap。（这里要注意下：编译器相比一般化template更喜欢T专属std::swap特化版）

## 流程

* 1.提供一个public swap成员函数，让它高效地置换你的类型的两个对象。这个函数绝不该抛出异常。  
* 2.在你的class或template所在的命名空间提供一个non-member swap，并令它调用上述swap成员函数
* 3.如果正在编写的class并非class template，为你的class特化std::swap，并令它调用swap成员函数。

## 成员swap绝不可抛出异常的说明
我们承诺：成员swap绝不可抛出异常。  
因为swap一个最好的应用就是帮助classes提供强烈的异常安全性保障。非成员版本不受此约束，因为缺省swap是以copy构造以及copy赋值为基础，而这两个都可能抛出异常。  
因此我们写自定义swap不仅仅是提供了高效置换对象的方法，同时不会抛出异常。这两个性质是相辅而成的，因为高效率的swap**s**几乎都是基于对内置类型的操作（例如pimpl手法的底层指针），而**内置类型上的操作绝不会抛出异常**。
## 总结
* 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不会抛出异常。
* 如果你提供一个**member** swap，也该提供一个**non-member** swap来调用前者，对于classes（而非templates），也请特化std::swap。
* 调用swap时应针对std::swap使用using声明，然后调用swap并且不限定namespace
* 为“用户定义类型”进行std template 全特化是好的，但是不要往std中添加新东西
