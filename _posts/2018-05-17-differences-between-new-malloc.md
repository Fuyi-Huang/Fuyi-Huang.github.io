---
title: 在c++里用new和malloc的几点区别
tags: 
    - programming 
    - c++
article_header:
  type: cover
  image:
    src: assets/images/header_images/zelda.jpg
---

# 前言

昨天面试c++的岗位的时候，面试官问我[new](https://en.cppreference.com/w/cpp/memory/new/operator_new)和[malloc](https://en.cppreference.com/w/cpp/memory/c/malloc)的区别。因为平时写c++都是用的new，没怎么用过malloc，所以不知道它们的区别，没回答上来。现在来整理一下。

# new和malloc的几点区别

1. 返回类型安全性

    用new申请内存空间时，返回的数据类型是声明类型的指针，而malloc返回的数据类型是void*，需要之后类型转换成我们想要的类型。

    ```c++
    MyClass* mc = new MyClass();

    MyClass* mc1 = (MyClass*) malloc(sizeof(MyClass));
    ```

2. 是否需要指定内存大小

    new表达式声明时不需要指定要申请的内存空间的大小，而用malloc申请空间时需要指定空间大小。
    但是，new表达式不需要指定申请空间大小是因为new表达式调用的函数会计算所需的空间大小，如MyClass* mc = new MyClass()编译后的伪代码可能如下：

    ```c++
    void* _tmp = ::operator new(sizeof(MyClass));   // 这里计算所需的空间大小
    MyClass* mc = reinterpret_cast<MyClass*>(_tmp);
    ```

3. 是否调用构造函数/析构函数

    new和malloc都是申请一段动态存储空间，它不受声明时的scope限制，会一直存在于内存中直到被显式地释放或者程序终止。但是，不同的是new是申请并初始化一段内存空间，而malloc则是申请一段未初始化的空间。所以当用new创建对象时，它会调用对象的构造函数，对象被delete时也会调用对象的析构函数；而malloc则不会。例如

    ```c++
    class MyClass {
        public: 
            MyClass() {
                myInt = 99;
                std::cout << myInt << std::endl;
            }
            ~MyClass() {
                std::cout << myInt + 99 << std::endl;
            }
        private:
            int myInt;
    };
    int main(int argc, char* argv[])
    {
        MyClass* my = new MyClass();
        delete my;

        std::cout << "---------------------" << std::endl;

        MyClass* my1 = (MyClass*) malloc(sizeof(MyClass));
        free(my1);

        return 0;
    }
    ```

    上述代码的执行结果是：

    ```
    $ ./a.exe
    99
    198
    ---------------------
    ```

    用new申请空间时如果是数组的话，会对数组中的每一个存储单元都调用构造函数、析构函数。要对每个单元都调用构造函数和析构函数的话需要知道数组中的元素的个数，所以申请的空间中会有多申请一段用于存储数组元素个数的空间

    ```c++
    #include<iostream>
    using namespace std;

    class Myclass{
        int x, y;
    public: 
        MyClass() {}
        ~MyClass() {}
    };

    int main() {
        MyClass* mc = new MyClass[10];
        delete[] mc;

        return 0;
    }
    ```

    上述的代码会被编译器编译成类似如下的代码：

    ```c++
    int main() {
        void* _tmp = ::operator new[](sizeof(MyClass) * 10 + sizeof(size_t));
        MyClass* _tmpA = reinterpret_cast<MyClass*>(reinterpret_cast<size_t*>(_tmp) + 1);
        {
            // array length record at offset: -sizeof(size_t)
            // 数组的元素个数存在减去size_t大小的空间
            *(reinterpret_cast<size_t*>(_tmpA) - 1) = 10;
            MyClass* _cur = _tmpA;
            for(int _i = 9; _i != -1; _cur++, _i--)
                MyClass::MyClass(_cur); // call constructor;    调用构造函数
        }
        MyClass* mc = _tmpA;

        if(mc != nullptr)
        {
            size_t len = *(reinterpret_cast<size_t*>(mc) - 1);
            MyClass* _cur = mc;
            // call the deconstructor from last to first element in the array
            // 对数组中的最后一个到第一个元素调用析构函数
            while(_cur != mc)
            {
                _cur = _cur - 1;
                auto _vptr = reinterpret_cast<void (***)(MyClass*)>(_cur)[0];
                _vptr[0](_cur);
            }
            // call the operator delete[](size_t sz) at the original location
            ::operator delete[](reinterpret_cast<size_t*>(mc) - 1);
        }

        return 0;
    }
    ```

    如上述代码分析的那样，会有一个size_t的额外空间用来存储数组长度。

4. 申请空间失败后的操作

    mallco分配内存空间失败之后返回null指针，而new则是抛出exception。

    ```c++
    #include<iostream>
    #include<cstdlib>

    int main() {
        int* ptr = (int*) std::malloc(sizeof(int[10]));
        if(ptr)
        {
            for(int i = 0; i < 10; i++)
                ptr[i] = i * i;
            for(int i = 0; i < 10; i++)
                std::cout << "ptr[" << i << "] = " << ptr[i] << std::endl;
        }
        std::free(ptr);

        return 0;
    }
    ```

    new申请内存空间失败抛出的异常是[std::bad_alloc](https://en.cppreference.com/w/cpp/memory/new/bad_alloc)

    ```c++
    #include <iostream>
    #include <new>
    
    int main()
    {
        try {
            while (true) {
                new int[100000000ul];
            }
        } catch (const std::bad_alloc& e) { // 上述new申请空间失败抛出bad_alloc异常
            std::cout << "Allocation failed: " << e.what() << '\n';
        }
    }
    ```

    输出：

    > Allocation failed: std::bad_alloc

    c++里有相应分配空间失败的处理函数new_handler

    ```c++
    #include <iostream>
    #include <new>
    
    void handler()
    {
        std::cout << "Memory allocation failed, terminating\n";
        std::set_new_handler(nullptr);
    }
    
    int main()
    {
        std::set_new_handler(handler);
        try {
            while (true) {
                new int[100000000ul];
            }
        } catch (const std::bad_alloc& e) {
            std::cout << e.what() << '\n';
        }
    }
    ```

    输出：

    ```
    Memory allocation failed, terminating
    std::bad_alloc
    ```

5. 重新分配内存

    当程序运行时发现内存空间不足时，用malloc（或calloc）申请的空间可以用realloc函数来扩容，如下：

    ```c++
    void* realloc(void* ptr, std::size_t new_size);
    ```

    具体可以看[cppreference: realloc](https://en.cppreference.com/w/cpp/memory/c/realloc)在重新分配空间的过程中会先检查当前ptr所指向的空间的后续可用空间是否足够大，是则原本的空间不变，拓展ptr可用的空间；否则分配一个new_size大小的空间，然后将原本的内容拷贝到新的空间，接着释放原本的空间。

    用new申请的空间没有相应的扩容函数，但是c++有vector，vector是一个当容器容量不足时自动扩容的容器。

# 后言

在cppreference的malloc页面有这么一段话：

> This function does not call constructors or initialize memory in any way. There are no ready-to-use smart pointers that could guarantee that the matching deallocation function is called. The preferred method of memory allocation in C++ is using RAII-ready functions std::make_unique, std::make_shared, container constructors, etc, and, in low-level library code, new-expression.

就是说在c++里面，一般用RAII的方法申请空间，所以有必要研究一下RAII
