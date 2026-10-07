# 1 lambda

## 1.1 lambad简单应用

lambda表达式又称为匿名表达式，是C11提出的新语法。

**[]** 存储lambda表达式**要捕获的值**，**()** 内的参数为**形参**，可供外部调用传值。lambda表达式可以直接调用

下方代码定义了一个匿名函数后直接调用

```cpp
// 1  匿名调用
   [](string name)
   {
       cout << "this is anonymous" << endl;
       cout << "hello " << name << endl;
   }("zack");

" [](){} "可将这个看成一个函数指针
```



我们可以通过auto初始化一个变量存储lambda表达式，也就是存储函数指针，之后直接调用fname即可

```cpp
// 2 通过auto赋值
   auto fname = [](string name)
   {
       cout << "this is auto  " << endl;
       cout << "hello " << name << endl;
   };

   fname("Rolin");
```



通过auto定义fname，然后存储了lambda表达式，之后调用fname即可。也可以通过函数指针的方式接受lambda表达式

P_NameFunc定义了fname2函数指针接受了lambda表达式

```cpp
typedef void (*P_NameFunc)(string name);
// 3 函数指针
P_NameFunc fname2 = [](string name)
{
    cout << "this is P_NameFunc " << endl;
    cout << "hello " << name << endl;
};

fname2("Vivo");
```



也可以通过function对象接受lambda表达式，function类是C11新增的语法。

用一个function对象接受了lambda表达式，同样可以调用该function对象funcName达到调用lambda的效果。

```cpp
// 4 function
    function<void(string)> funcName;// 定义了一个函数指针
    funcName = [](string name)
    {
        cout << "this is function " << endl;
        cout << "hello " << name << endl;
    };

    funcName("Uncle Wang");
```



## 1.2 lambda浅谈

## 谈谈lambda的捕获

1. **值捕获**:这相当于在函数对象内部创建了 `age` 和 `name` 的**副本**。它很安全，因为副本归 Lambda 自己所有

```cpp
int age = 33;
string name = "zack";
int score = 100;
string job = "softengineer";
//值捕获
[age, name](string name_)
{
    cout << "age is " << age << " name is " << name << " self-name is " << name_ << endl;
}("Novia");
```

上述lambda表达式捕获了age和name，是以值的方式来捕获的。所以无法在lambda表达式内部修改age和name的值，如果修改age和name，编译器会报错，提示无法修改const常量，因为age和name是以值的方式被捕获的。





2. **引用捕获**:这相当于在函数对象内部存储了指向局部变量 `age` 和 `name` 的**指针**。它很危险，因为它**只存储了变量的地址，并没有延长这些变量的生命周期**。

```cpp
int age = 33;
string name = "zack";
int score = 100;
string job = "softengineer";
[&age, &name](string name_)
{
    cout << "age is " << age << " name is " << name << " self-name is " << name_ << endl;
    name = "Xiao Li";
    age = 18;
}("Novia");
```

[]里age和name前边添加了&，此时age和name是以引用方式捕获的。所以可以在lambda表达式中修改age和name的值。
C++的lambda表达式虽然可以捕获局部变量的引用，达到类似闭包的效果，但不是真的闭包，golang和python等语言通过闭包捕获局部变量后可以增加局部变量的声明周期，C++无法做到这一点，所以下面的调用会出现崩溃。

```cpp
vector<function<void(string)>> vec_Funcs;
void use_lambda2()
{
    int age = 33;
    string name = "zack";
    int score = 100;
    string job = "softengineer";

    vec_Funcs.push_back([age, name](string name_)
                        {   cout << "this is value catch " << endl;
                            cout << "age is " << age << " name is " << name << " self-name is " << name_ << endl; });
    //危险，不要捕获局部变量的引用
    vec_Funcs.push_back([&age, &name](string name_)
                        {   cout << "this is referenc catch" << endl;
                            cout << "age is " << age << " name is " << name << " self-name is " << name_ << endl; });
}

void use_lambda3()
{
    for (auto f : vec_Funcs)
    {
        f("zack");
    }
}

int main(){
    use_lambda2();
    use_lambda3();
}
```

use_lambda2中将lambda表达式存储在function类型的vector里，当use_lambda2结束后，里边的局部变量都被释放了，而vector中的lambda表达式还存储着局部变量的引用，在调用use_lambda3时调用lambda表达式，此时访问局部变量已经被释放了，所以导致程序崩溃。





3. **全部用值捕获，name用引用捕获**

```cpp
int age = 33;
string name = "zack";
int score = 100;
string job = "softengineer";
[=, &name]()
{
    cout << "age is " << age << " name is " << name << " score is " << score << " job is " << job << endl;
    name = "Cui Hua";
}();
```

通过=表示所有变量都以值的方式捕获，如果希望某个变量以引用方式捕获则单独在这个变量前加&。





4. **全部用引用捕获，只有name用值捕获**

```cpp
int age = 33;
string name = "zack";
int score = 100;
string job = "softengineer";
[&, name]()
{
     cout << "age is " << age << " name is " << name << " score is " << score << " job is " << job << endl;
}();
```

通过&方式表示所有变量都已引用方式捕获，如果希望某个变量以值方式捕获则单独在这个变量前加=，但是不需要加=，默认就是值捕获





# 2 万能的function

我们可以用function存储形参和返回值相同的一类函数指针，可调用对象，lambda表达式等。

```cpp
#include <iostream>
#include <list>
#include <functional>
using namespace std;

// 1. 定义全局函数
void globalFun(string str)
{
    cout << "this is global function call " << str << endl;
}

// 2. 定义函数对象（仿函数）
class FuncObj
{
public:
    void operator()(string str) const
    {
        cout << "this is function object call " << str << endl;
    }
};

void use_function()
{
    list< function< void(string)> > list_Funcs;
    
    // 存储函数对象
    list_Funcs.push_back(FuncObj());
    
    // 存储lambda表达式
    list_Funcs.push_back([](string str)
                         { cout << "this is lambda call " << str << endl; });
    
    // 存储全局函数
    list_Funcs.push_back(globalFun);

    // 遍历并调用所有函数
    for (const auto &f : list_Funcs)
    {
        f("hello zack");
    }
}

int main()
{
    use_function();
    return 0;
}
```







# 3 bind操作

C11同样提供了bind操作，将原函数的几个参数通过bind绑定传值，返回一个新的可调用对象。

```cpp
//绑定全局函数
auto newfun1 = bind(globalFun2, placeholders::_1, placeholders::_2, 98, "worker");

//相当于调用globalFun2("Lily",22, 98,"worker");
// "Lily"传入的是第一个占位符的位置，22传入的是第二个占位符的位置，最后函数调用是这样
// globalFun2("Lily",22, 98,"worker");
newfun1("Lily", 22);


// 多传参数没有用，相当于调用globalFun2("Lucy",28, 98,"worker");
// 100是第三个占位符,"doctor"是第四个占位符,但是在auto newfun1 = bind(globalFun2, placeholders::_1, placeholders::_2, 98, "worker");定义中没有第三个占位符与第四个占位符的位置
newfun1("Lucy", 28, 100, "doctor");


auto newfun2 = bind(globalFun2, "zack", placeholders::_1, 100, placeholders::_2);

//相当于调用globalFun2("zack",33,100,"engineer");
newfun2(33, "engineer");


auto newfun3 = bind(globalFun2, "zack", placeholders::_2, 100, placeholders::_1);

// 相当于调用globalFun2("zack",33,100,"coder");
newfun3("coder", 33);
```

placeholders表示占位符，_1表示新生成函数的第一个参数, _2表示新生成函数的第二个参数，将这些参数传递给原函数达到占位的效果，原函数的其余参数通过bind绑定固定值。



接下来定义类

```cpp
class BindTestClass
{
public:
    BindTestClass(int num_, string name_) : num(num_), name(name_) {}
    static void StaticFun(const string &str, int age);
    void MemberFun(const string &job, int score);

public:
    int num;
    string name;
};
```

实现静态函数和成员函数

```cpp
void BindTestClass::StaticFun(const string &str, int age)
{
    cout << "this is static function" << endl;
    cout << "name is " << str << endl;
    cout << "age is " << age << endl;
}
void BindTestClass::MemberFun(const string &job, int score)
{
    cout << "this is member function" << endl;
    cout << "name is " << name << endl;
    cout << "age is " << num << endl;
    cout << "job is " << job << endl;
    cout << "score is " << score << endl;
    name = "modified";  // 尝试修改成员变量
    age = 99;
}
```



我们通过bind绑定静态成员函数
[static修饰类函数](./static用法.md)

```cpp
//绑定类的静态成员函数,加不加&都可以
// auto staticbind = bind(BindTestClass::StaticFun, placeholders::_1, 33);
auto staticbind = bind(&BindTestClass::StaticFun, placeholders::_1, 33);
staticbind("zack");
```

新生成的staticbind函数可以直接传递一个参数zack就完成了调用。接下来用bind绑定成员函数

```cpp
BindTestClass bindTestClass(33, "zack");
// 绑定类的成员函数(非静态函数),一定要传递对象给bind的第二个参数，可以是类对象，也可以是类对象的指针
// 如果要修改类成员，必须传递类对象的指针
// 方式1：传对象指针（可以修改对象）
auto memberbind1 = bind(&BindTestClass::MemberFun, &bindTestClass, placeholders::_1, placeholders::_2);
memberbind1("coder", 100);  // 相当于 bindTestClass.MemberFun("coder", 100)

// 方式2：传对象本身（不能修改对象，因为是副本）
auto memberbind2 = bind(&BindTestClass::MemberFun, bindTestClass, placeholders::_1, placeholders::_2);
memberbind2("coder", 100);  // 在 bindTestClass 的副本上调用，原对象不变

auto memberbind2 = bind(&BindTestClass::MemberFun, placeholders::_3, placeholders::_1, placeholders::_2);
memberbind2("coder", 100, &bindTestClass); 
// bind(&BindTestClass::MemberFun, &bindTestClass, "coder", 100);
// 相当于bindTestClass.MemberFun("coder", 100)


//绑定类成员时，对象必须取地址
//以下代码为绑定成员变量指针
auto numbind = bind(&BindTestClass::num, placeholders::_1);
std::cout << numbind(bindTestClass) << endl;
```





当然也可以直接用function对象接受bind返回的结果

```cpp
void globalFun2(const string& name, int age, int score, const string& job)
{
    cout << "globalFun2: " << name << ", age " << age 
         << ", score " << score << ", job " << job << endl;
}


// function接受bind返回的函数
function< void(int, string)> funcbind = bind(globalFun2, "zack", placeholders::_1, 100, placeholders::_2);
funcbind(33, "engineer");


// function接受bind 成员函数
function< void(string, int)> funcbind2 = bind(&BindTestClass::MemberFun, &bindTestClass, placeholders::_1, placeholders::_2);
funcbind2("docker", 100);


function< void(string, int, BindTestClass *)> funcbind3 = bind(&BindTestClass::MemberFun, placeholders::_3, placeholders::_1, placeholders::_2);
funcbind3("driver", 100, &bindTestClass);


// function 直接接受成员函数,function的模板列表里第一个参数是类对象引用
function< void(BindTestClass &, const string &, int)> functomem = BindTestClass::MemberFun;
functomem(bindTestClass, "functomem", 88);


// function 绑定类的静态成员函数
function< void(const string &)> funbindstatic = bind(&BindTestClass::StaticFun, placeholders::_1, 33);
funbindstatic("Rolis");
```















