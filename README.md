
## C++ 语法解说

我们先来看一行代码（[`document.h`][StartArray]）：

~~~c++
bool StartArray() {
    new (stack_.template Push<ValueType>()) ValueType(kArrayType); // <--
    return true;
}
~~~


## Placement `new`

简单来说，placement `new` 就是不分配内存，由使用者给予内存空间来构建对象。其形式是：

~~~cpp
new (T*) T(...);
~~~


## `template` disambiguator

(1)其实只是调用 `Stack` 类的模板成员函数 `Push()`。如果删去这个 `template`，代码就显得正常一点：

~~~cpp
    ValueType* v = stack_.Push<ValueType>(); // (1)
~~~

这里 `Push<ValueType>` 是一个 dependent name，它依赖于 `ValueType` 的实际类型。这里编译器不能确认 `<` 为小于运算符，还是模板的 `<`。为了避免歧义，需要加入`template` 关键字。这是C++标准的规定，缺少这个 `template` 关键字 gcc 和 clang 都会报错，而 vc 则会通过（C++标准也容许实现这样的编译器）。和这个语法相近的还有 `typename` disambiguator。

理解这些语法之后，我们进入核心问题。

## 混合任意类型的堆栈

处理树状的数据结构时，我们经常需要用到堆栈（stack）这种数据结构。C++ 标准库也提供了 [`std::stack`][stdstack] 这个容器。然而，这个模板类容器的实例，只能存放一种类型的对象。在 RapidJSON 的解析过程中，我们希望它能同时存放已解析的 `Value` 对象，以及 `Member` 对象（key-value对）。或者我们从另一个角度去想，程序堆栈（program stack）本身就是可储存各种类型数据的堆栈。在 RapidJSON 中的其它地方也有这种需求。

在 [`internal/stack.h`][stack.h] 中的 `Stack` 类实现了这个构思，其声明是这样的：

~~~cpp
class Stack {
    Stack(Allocator* allocator, size_t stackCapacity);
    ~Stack();

    void Clear();
    void ShrinkToFit();
    
    template<typename T> T* Push(size_t count = 1);
    template<typename T> T* Pop(size_t count);
    template<typename T> T* Top();
    template<typename T> T* Bottom();

    Allocator& GetAllocator();
    bool Empty() const;
    size_t GetSize();
    size_t GetCapacity();
};
~~~

这个类比较特殊的地方，就是堆栈操作使用模板成员函数，可以压入或弹出不同类型的对象。另外，为了完全防止拷贝构造函数调用的可能性，这些函数都是返回指针。虽然引用也可以，但使用指针在一些应用情况下会更自然。

例如，要压入4个 `int`，再每次弹出两个：

~~~cpp
Stack s;
*s.Push<int>() = 1;
*s.Push<int>() = 2;
*s.Push<int>() = 3;
*s.Push<int>() = 4;
for (int i = 0; i < 2; i++) {
    int* a = s.Pop<int>(2);
    std::cout << a[0] << " " << a[1] << std::endl;
}
// 输出：
// 3 4
// 1 2
~~~

注意到，`Pop()` 返回弹出的最底端元素的指针，我们仍然可以通过这指针合法地访问这些弹出的元素。

## 重要事项（坑出没注意）

在 `StartArray()` 的例子里，我们看到使用 placement `new` 来构建对象。在普通的情况下，`new` 和 `delete` 应该是成双成对的，但使用了 placement `new`，就通常不能使用 `delete`，因为 `delete` 会调用析构函数**并**释放内存。在这个例子里，`stack_` 对象提供了内存空间，所以我们只需要调用 `ValueType` 的析构函数。例如，如果解析在中途终止了，我们要手动弹出已入栈的 `ValueType` 并调用其析构函数：

~~~cpp
while (!stack_.Empty())
    (stack_.template Pop<ValueType>(1))->~ValueType();
~~~

