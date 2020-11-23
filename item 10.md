# 令operator=返回一个reference to *this
赋值运算符重载一般都是返回一个对*this的引用，这与资源管理优化及内置类型的要求是分不开的。
## 连锁赋值
```cpp
int x,y,z;
x=y=z=15; //连锁赋值
```
连锁赋值是采用右结合律的，即
```
x=(y=(z=15));
```
为了实现连锁赋值实际上并不要求必须返回一个引用，我们返回值也是可行的。但是不是一个很好的主意。我们看下面这个例子：
```cpp
#include <iostream>

using namespace std;
class A{
private:
    int val;
public:
    A(int a=0):val(a){}
    A(const A &a):val(a.val)
    {
        cout<<"constructor"<<endl;
    }
    A operator=(const A &a)
    {
        cout<<"assignment operation"<<endl;
        val=a.val;
        return *this;
    }
    ~A(){cout<<"destructor"<<endl;}
    friend ostream& operator<<(ostream&,const A&);
};

ostream& operator<<(ostream& os,const A &a)
{
    os<<a.val;   
    return os;
}

int main()
{
    A a,b,c,d{9};
    //A &r=(a=b);
    a=b=c=d;
    cout<<a<<" "<<b<<" "<<c<<endl;
}
```
运行结果：
```
assignment operation
constructor            //unnecessary
assignment operation
constructor            //unnecessary
assignment operation
constructor            //unnecessary
destructor             //unnecessary
destructor             //unnecessary
destructor             //unnecessary
9 9 9
destructor
destructor
destructor
destructor
```
如果operator=返回类型加上&
则运行结果：
```
assignment operation
assignment operation
assignment operation
9 9 9
destructor
destructor
destructor
destructor
```
高下立判，明显返回值的operator=返回时创建临时副本（调用拷贝构造）以及销毁临时对象（调用析构），这些都是赘余的。  
而且注意

```
int &r=(a=b);
```
如果返回临时对象，是不能进行左值引用的，这也会造成一定的麻烦（user可能会因为与内置赋值运算符不一样而感到诧异）。  
总之临时对象不是一件好事，理应舍弃这种做法。  

## 协议一致性
上面讲了为什么用&比值返回更合适，其实，这个协议被所有内置类型和标准程序库提供的类型如string，vector等共同遵守，当然，这不是必须遵守的，但是，除非你能提供更好的版本，否则还是随众为好。
