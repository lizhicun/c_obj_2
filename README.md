# c_obj_2
第二章 构造函数语义学（The Semantics of Constructors）
本章目标：挖掘编译器对“对象构造过程的干涉”，以及对程序形式和效率的冲击
默认构造函数
拷贝构造函数
程序转换语意学
成员们的初始化

2.1默认构造函数
以下四种情况，编译器会认为需要合成一个默认构造函数。C++ Standard 把这些合成物称为 implicit nontrivial default constructor。
“带有 Default Constructor”的 Member Class Object
```
class Foo {
 public:
  Foo(), Foo(int)...
};

class Bar {
 public:
  Foo   foo;
  char *str;
};

void foo_bar() {
  Bar bar;  // Bar::foo must be initialized here

  if (str) {
  }
  ...
}
```
合成出来的构造函数可能如下
```
// possible synthesis of Bar default constructor
// invoke Foo default constructor for member foo
inline Bar::Bar() {
  // Pseudo C++ Code
  foo.Foo::Foo();
}
```
“带有 Default Constructor”的 Base Class
没啥说的
“带有一个 Virtual Function”的 Class
```
class Widget {
 public:
  virtual void flip() = 0;
  // ...
};
void flip(const Widget& widget) {
  widget.flip();
}
// presuming Bell and Whistle are derived from Widget
void foo() {
  Bell    b;
  Whistle w;
  flip(b);
  flip(w);
}
```
编译器会为每个 Widget object 的 vptr 设定初值。对于没有任何 constructor 的 class，编译器则合成一个 default constructor 来做此事

“带有一个 Virtual Base Class”的 Class
略

至于没有存在这四种情况下且没有声明 constructor 的 class，它们拥有的是 implicit trivial default constructor，且实际上并不会被合成出来。

在合成的 default constructor 中，只有 base class subobject 和 member class object 会被初始化，其他的 nonstatic data member 都不会被初始化，因为编译器不需要。

C++ 新手（我）一般有两个误解：

1.任何 class 如果没有定义 default constructor，就会被合成出一个来。
2.编译器合成出来的 default constructor 会明确设定 “class 内每一个 data member 的默认值”。
以上两点都是错的！

2.2 拷贝构造函数
若没有定义 copy constructor，那么当 class object 以 相同 class 的另一个 object 作为初值时，其内部是以 default memberwise initialization 手法完成的
Default Memberwise Initialization
```
class String {
 public:
  // ... no explicit copy constructor
 private:
  char *str;
  int   len;
};

String noun("book");
String verb = noun;
```
则 verb 的初始化就会这样进行：
```
// semantic equivalent of memberwise initialization
verb.str = noun.str;
verb.len = noun.len;
```

class 不展现出“bitwise copy semantics”时，编译器会合成拷贝构造函数。有以下四种情况
当class 内含一个 member object，而这个 member object 有一个 copy constructor（包括程序员定义的和编译器合成的）。
当class 继承自一个 base class，而这个 base class 有一个 copy constructor（同样，包括程序员定义的和编译器合成的）。
当class 声明了 virtual function 时。
当class 派生自一个继承串链，其中有 virtual base class 时。

前两条略

在 class 声明了 virtual function 后，编译期间会有两个程序扩张操作：
增加一个 virtual function table（vtbl。
在每一个 class object 内安插 vptr。
很显然，在 copy 的时候需要为 vptr 正确的设定初值才行，而不是简单的拷贝。这时候，class 就不再展现 bitwise semantics 了。
看代码
```
class ZooAnimal {
 public:
  ZooAnimal();
  virtual ~ZooAnimal();
  virtual void animate();
  virtual void draw();
  // ...
 private:
  // data necessary for ZooAnimal's
  // version of animate() and draw()
};
class Bear : public ZooAnimal {
 public:
  Bear();
  void         animate();
  void         draw();
  virtual void dance();
  // ...
 private:
  // data necessary for Bear's version
  // of animate(), draw(), and dance()
};

Bear yogi;
Bear winnie = yogi;
```
[图]
当一个 base class object 用一个 derived class 的 object 初始化时，其 vptr 的复制也必须保证安全：
```
ZooAnimal franny = yogi;	// 译注：这会发生切割（sliced）行为
```
[图]
合成出的 ZooAnimal copy constructor 会明确设定 object 的 vptr 指向 ZooAnimal class 的 virtual table，而非单纯的拷贝

2.3程序转化语义学（Program Transformation Semantics）
这节大概是想说，我们对代码的假设，可能与实际并不相符。
比如
```
#include "X.h"

X foo() {
  X xx;
  // ...
  return xx;
}
```
我们可能会做出如下假设：
每次 foo() 被调用，就传回 xx 的值。
如果 class X 定义了一个 copy constructor，那么当 foo() 被调用时，保证该 copy constructor 也会被调用。
这两个假设都得视编译器所提供的进取性优化程度（degree of aggressive optimization）而定，可能它们都不正确。

有三种情况，会以一个obj的初值内容作为另一个obj的初值
1.明确的初始化
```
X x;
X xx = x;
```
2.将obj作为参数
```
X x;
foo(x);
```
3.函数返回一个obj
```
X foo() {
    X x;
    return x;
}
```
下面具体讲
明确的初始化操作（Explicit Initialization
代码张这样
```
void foo_bar() {
  X x1(x0);
  X x2 = x0;
  X x3 = X(x0);
  // ...
}
```
转换后可能长这样
```
// Possible program transformation
// Pseudo C++ Code
void foo_bar() {
  X x1;
  X x2;
  X x3;
  // compiler inserted invocations
  // of copy constructor for X
  x1.X::X(x0);
  x2.X::X(x0);
  x3.X::X(x0);
  // ...
}
```
会有如下两个转化阶段：
重写每一个定义(即占用内存)。
class 的 copy constructor 调用操作会被安插进去

参数的初始化（Argument Initialization）
```
void foo(X x0);

X xx;
// ...
foo(xx);
```
可能转换成这样
```
// Pseudo C++ code
// compiler generated temporary
X __temp0;
// compiler invocation of copy constructor
__temp0.X::X(xx);
// rewrite function call to take temporary
foo(__temp0);
```

返回值的初始化（Return Value Initialization）
```
X bar() {
  X xx;
  // 处理 xx ...
  return xx;
}
```
```
// function transformation to reflect
// application of copy constructor
// Pseudo C++ Code
void bar(X& __result) {	// 这里多了一个参数哦
  X xx;
  // compiler generated invocation
  // of default constructor
  xx.X::X();
  // ... process xx
  // compiler generated invocation
  // of copy constructor
  __result.X::X(xx);
  return;
}
```
现在编译器则会将如下调用操作：
```
X xx = bar();
```
转换为
```
// note: no default constructor applied
X xx;
bar( xx );
```

在编译器层面做优化（Optimization at the Compiler Level）
例如
```
X bar() {
  X xx;
  // ... process xx
  return xx;
}
```
所有的 return 指令传回相同的具名数值（named value），因此编译器可能会做优化，以 result 参数代替 named return value
```
void bar(X &__result) {
  // default constructor invocation
  // Pseudo C++ Code
  __result.X::X();
  // ... process in __result directly
  return;
}
```
这种优化被称为 Named Retrun Value（NRV）优化
只有copy constructor才会触发NRV 优化。
[测试]
NRV受争议的点
1.由编译器完成，不知道完成情况究竟什么样
2.无法handle复杂情况

程序员是否要写拷贝构造函数
自己写的好处是:如果一个 class 没有任何 member带有 copy constructor，也没有virtual function，那么这个 class 会以“bitwise copy”，这样效率高，且安全，不会有 memory leak，也不会产生 address aliasing。
坏处：如果这个 class 需要 大量的 memberwise 初始化操作，例如上面的测试，以传值的方式传回 object，那么就可以提供一个 copy constructor 来让编译器进行 NRV 优化。

2.4
成员们的初始化队伍（Member Initialization List）
如下情况中，如果在函数体内初始化，会影响效率：
```
class Word {
  String _name;
  int    _cnt;

 public:
  // not wrong, just naive ...
  Word() {
    _name = 0;
    _cnt = 0;
  }
};
```
这时候，编译器会做出如下扩张：
```
// Pseudo C++ Code
Word::Word(/* this pointer goes here */) {
  // invoke default String constructor
  _name.String::String();
  // generate temporary
  String temp = String(0);
  // memberwise copy _name
  _name.String::operator=(temp);
  // destroy temporary
  temp.String::~String();
  _cnt = 0;
}
```
可以看到，Word constructor 会先产生一个暂时的 String object，然后将它初始化，最后用赋值运算符将其指定给 _name。
如果这样写则效率更佳：
```
// preferred implementation
Word::Word : _name(0) {
  _cnt = 0;
}
```
它会被扩张为：
```
// Pseudo C++ Code
Word::Word(/* this pointer goes here */) {
  // invoke String( int ) constructor
  _name.String::String(0);
  _cnt = 0;
}
```
