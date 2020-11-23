# 确定对象被使用前已被初始化

## 基本保证

**永远在使用对象之前先将它初始化**：

* 对于无任何成员的内置类型，你必须手工完成此事

* 对于内置类型以外的自定义类型，则确保每一个构造函数都能将所有成员初始化

## 初值列表

构造函数优先选择初值列表（`member initialization list`）

有这样几个优点：

* 省去了拷贝赋值的开销以及没有多余的默认构造

* 如果成员类型为const或引用，不得不初始化

承诺：

* 总是在初值列表中初始化所有成员

* 按照成员声明次序进行初始化

## non-local static对象的处理方式

这里指的non-local是相对于函数而言的，函数以内的static对象是local

（但实际local只要是个block scope都是，不限于函数）

注：这里static是指static storage duration不是声明为static，也不一定得是internal linkage

我们的问题是：

> 如果某翻译单元内的某个non-local static对象的初始化用到了另一个翻译单元的某个non-local static对象，可能是还没有初始化的

```cpp
class FileSystem{
public:
    ...
    std::size_t numDisks()const;
    ...
};
extern FileSystem tfs;
```

```cpp
class Directory{
public:
    Directory(paras);
    ...    
};

Directory::Directory(paras){
    ...
    std::size_t disks=tfs.numDisks();
    ...
}

Directory tempDir(paras){...}//须在Directory构造之前初始化tfs
```

注意，C++并没有明确定义“在不同翻译单元内的non-local static对象的初始化次序”

解决方法就是利用已明确定义的规则：函数内的local static对象会在“该函数被调用期间”“首次遇到对象定义式”时被初始化，即**用local static对象代替non-local static对象**

```cpp
class FileSystem{...};
FileSystem& tfs(){
    static FileSystem fs;
    return fs;
}

class Directory{...};
Directory::Directory(paras){
    std::size_t disks=tfs().numDisks;   
}
Directory& tempDir(){
    static Directory td;
    return td;
}
```

## Things to Remember

* 为内置类型对象进行手工初始化，即使在file scope中也是如此

* 构造函数应使用初值列表来进行初始化并且严格按照成员声明顺序

* 为避免“跨翻译单元的初始化次序”问题，用local static对象替代non-local static对象
