# c++基础

## 1  多态

### 1.1 c++多态的构成条件

多态是指**不同继承关系的类对象**，去**调用同一函数**，**产生了不同的行为**。在继承中要想构成多态需要满足两个条件：

- 必须通过**基类的指针或者引用调用虚函数**。
- 被调用的函数**必须是虚函数**，且**派生类**必须对基类的**虚函数**进行**重写**。

### 1.2 实例

```
未实现多态：
#include <iostream>
using namespace std;

class Shape{
protected:
    int w;
    int h;
public:
    Shape(int a , int b) : w(a) , h(b) {}

    int area() const{
        cout << "Shape::area" << endl;
        return w * h;
    }
};

class Rectangle : public Shape{
public:
    Rectangle(int a , int b) : Shape(a , b) {}
    int area(){
        cout << "Rectangle::area" << endl;
        return w * h;
    }
};

int main()
{
    Shape* s = new Rectangle(3 , 4);  // 满足了条件一：使用基类的指针
    cout << s->area() << endl;   // 不满足条件二，不是虚函数。 输出：Shape::area
    delete s;
}

```

```
实现了多态：
#include <iostream>
using namespace std;

class Shape{
protected:
    int w;
    int h;
public:
    Shape(int a , int b) : w(a) , h(b) {}

    virtual int area() const{     // 虚函数
        cout << "Shape::area" << endl;
        return w * h;
    }
};

class Rectangle : public Shape{
public:
    Rectangle(int a , int b) : Shape(a , b) {}
    virtual int area() const {
        cout << "Rectangle::area" << endl;
        return w * h;
    }
};

int main()
{
    Shape* s = new Rectangle(3 , 4);
    cout << s->area() << endl;         // 输出：Rectangle::area
    delete s;
}
```

### 1.3 多态图例

![image-20230404192243564](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202304041924389.png)

如图所示，只要类内定义了虚函数，那么就会有**虚指针**（vptr）指向**虚表**（vtbl），虚表中存有虚函数的地址；(*(p->vptr)[n])**(p)**表示首先使用虚指针找到虚表，然后通过index n找到虚函数的地址，然后解引用得到虚函数，并且传入this指针p

![image-20230404193008868](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202304041930963.png)

如上图所示，myDoc在调用OnFIleOpen()时，会传入this指针；myDoc调用基类的成员函数OnFIleOpen，但是会调用myDoc的虚函数

**子类调用父类的成员函数时，会传入子类的this指针**。

## 2 (*p)[]

```
int (*p)[3];                   // p是一个指针，指向 包含 3个整形元素 的 一维数组
int a[3] = {1 , 2 , 3};
p = &a; 
cout << p << ' ' << *p << endl;     // p所指向的是一个数组，所以p的值为该数组的地址，*p为该数组首元素的地址
cout << *(*p) << endl;
cout << *(*p + 1) << endl;
cout << *(p + 1) << endl;
```

### 2.1 数组首元素地址、数组首地址、整个数组地址的分析

- 这三个地址从**数值**上来说是**相同**的

- ```
  int a[] = {1 , 2 , 3};
  cout << a << endl;   // 数组首元素的地址 == 数组首地址
  cout << &a << endl;  // 整个数组的地址
  ```

- 不同点：

  ```
  a + 1; // 加的是一个数组类型的长度
  &a + 1; // 加的是整个数组的长度
  ```

- **不理解：**

  ```
  int* a[2];
  int b = 10;
  int c = 2;
  a[0] = &b;
  a[1] = &c;
  int** p = a;
  cout << a << endl;
  cout << *p << endl;
  cout << &b << endl;
  cout << p[0] << endl;
  cout << (*p)[0] << endl;      // 为什么等于 *p[0]
  ```

- 小结一下：数组名可以当做数组的首元素的地址和数组的首地址，相当于是指向数组首元素的指针，**任何指向数组首元素的指针**被数组名**赋值**后，都可以进行数组相关操作。

## 3 抽象类

如果类中至少有一个函数被声明为**纯虚函数**，则这个类就是**抽象类**。纯虚函数是通过在声明中使用 "**= 0**" 来指定的

设计**抽象类**（通常称为 ABC）的目的，是为了给其他类提供一个**可以继承的适当的基类**。抽象类**不能**被用于实例化对象，它只能作为**接口**使用。

### 3.1 实例

```
namespace lzc2{
class Shape{
protected:
    int w;
    int h;
public:
    Shape(const int &a , const int &b) : w(a) , h(b) {}
    virtual int area() const = 0;    // 定义了纯虚函数
};

class Rectangle : public Shape{
public:
    Rectangle(int a , int b) : Shape(a , b) {}
    virtual int area() const {        // 重写
        return w * h;
    }
};

}

int main()
{
    lzc2::Rectangle r = lzc2::Rectangle(3 , 4);
    cout << r.area() << endl;
}
```

## 4 重写和重载的区别

- 定义不同： 重载是定义相同的**方法名**，**参数不同**； 重写是**子类**重写**父类**的方法。

- 范围不同：重载是在**一个类**中，重写是子类与父类之间的。

- 多态不同：重载是编译时的多态性，重写是运行时的多态性。

- 返回不同：重载对返回类型没有要求，而重写要求返回类型必须相同。

- 参数不同：重载的参数个数、参数类型、参数顺序可以不同，而重写父子方法参数**必须相同**。

- 修饰不同：重载对访问修饰没有特殊要求，重写访问修饰符的限制一定要**大于**被重写方法的访问修饰符。

- **重写**需要两个函数的**签名相同**（返回类型、函数名、参数等）：

  ```
  virtual int area() const = 0;
  virtual int area() const {        // 重写
      return w * h;
  }
  其中const 也属于签名的一部分
  ```

  

## 5 读写流

[C++ 文件和流](https://github.com/0voice/cpp_new_features/blob/main/C++%20%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B%EF%BC%8841%E8%AF%BE%E6%97%B6%EF%BC%89%20-%20%E9%98%BF%E9%87%8C%E4%BA%91%E5%A4%A7%E5%AD%A6.md#c-%E6%96%87%E4%BB%B6%E5%92%8C%E6%B5%81)

![1680674086115](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202304051355979.png)

[c++ getline()详解](https://blog.csdn.net/m0_52824954/article/details/128194817)

```
ofstream outfile;  // 定义输出流，用于写入文件
outfile.open("lzc.txt" , ios::app);   // 打开文件，并且开启追加模式

char c[100];
cin.getline(c , 100 , ',');     // 第一种读取方式，这种方式的第一个参数必须是指针

string c;
getline(cin , c , ' ');    // 第二种读取方式，这种方式的第一个参数是string

outfile << c;     // 将c的值写入文件
outfile.close();
```

```
ifstream infile;
infile.open("lzc.txt" , ios::in);

string s;
infile >> s;               // 直接输入，但是只会读一行
cout << s;
infile >> s;
cout << endl << s << endl;

while(getline(infile , s)){    // 每次读一行
    cout << s << endl;
}
infile.close();
```

## 6 异常处理

[C++ 异常处理](https://blog.csdn.net/qq_53144843/article/details/127242706)

异常是通过**抛出异常对象**来实现的，抛出来的异常对象会与catch中的**异常类型**进行匹配

可以在**任意地方**抛出异常

```
namespace lzc3{
class Exception{           // 自定义异常处理的基类
protected:
    string _errmsg;
    int _errid;
public:
    Exception(string a , int b) : _errmsg(a) , _errid(b) {}
    int GetErrid() const {    // 这里必须加 const 
        return _errid;
    }
    virtual void what() const {    // 定义虚函数，方便实现多态
        cout << _errmsg << endl;
    }
};

class MyException : public Exception {
public:
    MyException(string a , int b) : Exception(a , b) {}
    virtual void what() const {
        cout << "MyException:: "<< ' ' << _errmsg << endl;
    }
};

int fun1(int a , int b){
    if(b == 0){
        throw MyException("b == 0!" , 0);   // 抛出异常临时对象
    }
    return a / b;
}

}

int main()
{
    try{
       int a = lzc3::fun1(1 , 0);
       cout << a << endl;
    }
    catch(const lzc3::MyException &a){  // 如果what() 和 GetErrid()不加const ，而这里加了const，则会报错
    									// 因为在调用成员函数时，会传入this指针，如果这里加了const，那么说明
    									// this指针指向的类是不能修改的，但是如果成员函数不加const，那么说明
    									// 这个成员函数可能会修改类，这样造成了前后矛盾，所以报错
        a.what();
        cout << endl;
        cout << a.GetErrid() << endl;
    }
}
```

## 7 预处理器

[C++预处理详解](https://www.cnblogs.com/liangliangh/p/3585326.html)



## 8 拷贝构造和赋值运算符

[C++ 拷贝构造函数和赋值运算符](https://www.cnblogs.com/wangguchangqing/p/6141743.html)

```
namespace lzc5{
class A{
private:
    int a;
    int b;
public:
    A(const int &c , const int &d): a(c) , b(d) {
        cout << "ctor" << endl;
    }
    A(const A& aa);
    A& operator = (const A&);
    void setAB(int a , int b){
//        cout << "setAB" << endl;
        this->a = a;
        this->b = b;
    }
    int geta() const {
        return a;
    }
    int getb() const {
        return b;
    }

    ~A(){
        cout << "~A()" << endl;
    }

};

inline A& A::operator = (const A& aa){
    this->setAB(aa.geta() , aa.getb());
    cout << "A::operator=" << endl;
    return *this;
}

inline A::A(const A& aa){
    this->setAB(aa.geta() , aa.getb());
    cout << "A::A(const A& aa)" << endl;
}

A foo(const A& aa){
    return aa;
}
}

int main()
{
    lzc5::A aa = lzc5::A(1 , 2);
//    lzc5::A t = lzc5::foo(aa);
    lzc5::foo(aa);
    
    lzc5::A t = aa;  // 拷贝构造
    
    lzc5::A t1 = lzc5::A(1 , 2);
    t1 = aa;        // 赋值操作
    
    cout << 1 << endl;
}
```

### 8.1 深层次理解

#### 8.1.1 例1

![image-20230406145341741](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202304061453776.png)

此时main 函数中产生的**临时空间**，**由t 来取而代之**。所以也发生一次构造，**一次拷贝**，两次析构。此时t 的地址，同a 的地址不同。

临时空间被t取代了，所以不会再发生拷贝

#### 8.1.2 例2

![image-20230406145533378](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202304061455404.png)

此时发生了**一次构造，一次析构**。也就是main 中的**t 取代了临时空间**，而b 的构造完成了t 的构造。所在完成了一次构造，一次析构。

#### 8.1.3 例3

![image-20230406163421719](https://cdn.jsdelivr.net/gh/lzclzclzc12/BlogImage@main/img/202304061634748.png)

此时发生了两次构造，一次赋值，**两次析构**，其中b 的构造，完成了main 函数中临时空间的构造，构造的临时对象作为赋值运算符重载的参数传入。然后发生赋值。两次析构分别是临时空间和t 的析构。

## 9 运算符重载

语句 `p->m` 被解释为 `(p.operator->())->m`

```
#include <iostream>
#include <fstream>
#include <string.h>
#include <csignal>
#include <vector>
using namespace std;

class Distance{
private:
    int feet;
public:
    Distance() : feet(0) {}
    Distance(const int &a) : feet(a) {}
    Distance(const Distance &a);
    int getFeet() const {
        return feet;
    }
    void setFeet(int f) {
        feet = f;
    }
    Distance& operator = (const Distance& ) ;

    Distance operator - () const ;
    Distance& operator -= (const Distance &);
    friend ostream& operator << (ostream& os , const Distance& d); // 该函数不属于Distance类
    Distance operator + (const Distance& ) const ;
    Distance operator - (const Distance& ) const ;
    Distance& operator ++() ;
    Distance operator ++(int) ;
};

Distance::Distance(const Distance &d){
    this->feet = d.getFeet();
}

Distance& Distance::operator = (const Distance & d) {
    this->feet = d.getFeet();
    return *this;
}

Distance& Distance::operator -= (const Distance &d){
    this->feet -= d.getFeet();
    return *this;
}

Distance Distance::operator - () const{
    return Distance(-this->getFeet());
}

Distance Distance::operator + (const Distance &d) const {
    return Distance(this->getFeet() + d.getFeet());
}

Distance Distance::operator - (const Distance &d) const {
    return Distance(this->getFeet() - d.getFeet());
}

Distance& Distance::operator ++ () {
    this->feet += 1;
    return *this;
}

Distance Distance::operator ++ (int) {
    Distance d = *this;
//    Distance::operator ++ ();
    ++(*this);
    return d;
}

ostream& operator << (ostream& os , const Distance& d){   // 该函数不属于Distance类
    os << d.getFeet();
}

class DistanceVector{
private:
    vector<Distance> v;
public:
    friend class SmartPoint;
    int len(){
        return v.size();
    }
    void add(const Distance& d){
        v.push_back(d);
    }
    Distance* getBegin(){
        return &v[0];
    }


};

class DVIterator{
private:
    Distance *point;
public:
    DVIterator(Distance* d) : point(d) {};
    Distance* operator -> () const ;
    Distance& operator * () const ;
    DVIterator& operator ++ ();
    DVIterator operator ++ (int);
};

Distance* DVIterator::operator -> () const {
    return point;
}

Distance& DVIterator::operator * () const {
    return *point;
}

DVIterator& DVIterator::operator ++ (){
    this->point ++;
    return *this;
}

DVIterator DVIterator::operator ++ (int){
    DVIterator t = this->point;
    this->point ++;
    return t;
}

int main()
{
//    Distance d = Distance(10);
//    d = Distance(10);
//    cout << d + Distance(100) << endl;
//    cout << d - Distance(100) << endl;
//    d -= Distance(100);
//    cout << -d << ' ' << Distance(100) << endl;
//    cout << ++d << ' ' << d++ << ' ' << d << endl;
    DistanceVector dv;
    dv.add(Distance(1));
    dv.add(Distance(10));
    dv.add(Distance(100));
    dv.add(Distance(1000));
    cout << dv.len() << endl;
    DVIterator i = dv.getBegin();
    cout << i->getFeet() << endl;
    cout << (*(i++)).getFeet() << endl;
    cout << i->getFeet() << endl;
    cout << (*(i++)).getFeet() << endl;
    cout << i->getFeet() << endl;
    cout << (*i).getFeet() << endl;
}

```

## 10 指向类成员指针

必须有类名  类名::

```
#include <iostream>
#include <string>
using namespace std;

class A{
public:
    typedef int int_a;
    int_a a;
    void func(){
        cout << "A::func()" << endl;
    }
    A(const int &aa) : a(aa) {}
    A() = default;

};
int main()
{
    A aa = A(2);
    int A::*point_a = &A::a;
    cout << aa.*point_a << endl;

    void (A::*point_func)() = &A::func;
    (aa.*point_func)(); 
}
```

## 11 友元

友元类的**所有成员函数**都是另一个类的**友元函数**，都可以访问另一个类中的隐藏信息（包括私有成员和保护成员）。

```
#include <iostream>
#include <string>
using namespace std;

class A{
public:
    typedef int int_a;
    friend class B;          // 声明友元类
    void func(){
        cout << "A::func()" << endl;
    }
    A(const int &aa) : a(aa) {}
    A() = default;

private:
    int_a a;
};

class B: public A{
public:
    void func(const A& aa) const {
        cout << aa.a << endl;         // 可以访问A中的private成员
    }
};


int main()
{
    A aa = A(2);
    B bb;
    bb.func(aa);
}
```

### 11.1 注意事项

- 友元声明**不受**其所在类的声明区域**public private 和protected** 的影响。
- 友元关系**不能被继承**。（因为友元不属于类内成员）
- 友元关系**不具有传递性**。若类B 是类A 的友元，类C 是B 的友元，类C 不一定是类A 的友元，同样要看类中是否有相应的声明

### 11.2 派生类友元函数

由于友元函数并非类成员，因引不能被继承，在某种需求下，可能希望**派生类的友元函数能够使用基类中的友元函数**。为此可以通过**强制类型转换**，将派生类的指针或是引用强转为其类的引用或是指针，然后使用转换后的引用或是指针来调用基类中的友元函数。

```
#include <iostream>
#include <string>
using namespace std;

class A{
public:
    typedef int int_a;
    void func(){
        cout << "A::func()" << endl;
    }
    A(const int &aa) : a(aa) {}
    A() = default;
    friend ostream& operator << (ostream& , const A& );
    int_a geta() const {
        return a;
    }
private:
    int_a a;
};

ostream& operator << (ostream& output , const A& aa) {
    output << aa.a << endl;
}

class B: public A{
public:
    friend ostream& operator << (ostream& , const B&);
    B(int aa , int bb) : A(aa) , b(bb) {};
private:
    int b;
};

ostream& operator << (ostream& output , const B& bb){
    output << (A)bb;       // 强转后调用 ostream& operator << (ostream& output , const A& aa)
    output << bb.b << endl;
}


int main()
{
    A aa = A(2);
    B bb = B(2 , 20);
    cout << bb;
}

```

## 12 类型转换

### 12.1 转换构造函数

格式：

```
class 目标类{
	目标类(const 源类& 源类对象引用){
		根据需求完成从源类型到目标类型的转换
	}
}
```

```
#include <iostream>
#include <string>
using namespace std;

class B;

class A{
public:
    typedef int int_a;
    void func(){
        cout << "A::func()" << endl;
    }
    A(const int &aa) : a(aa) {}
    A() = default;
    friend ostream& operator << (ostream& , const A& );
    int_a geta() const {
        return a;
    }
    friend class B;
private:
    int_a a;
};


ostream& operator << (ostream& output , const A& aa) {
    output << aa.a << endl;
}

class B {
public:
    friend ostream& operator << (ostream& , const B&);
    B(int aa , int bb) : a(aa) , b(bb) {};
    B(const A& aa){
        this->a = aa.a;
        this->b = 10;
    }
private:
    int a;
    int b;
};

ostream& operator << (ostream& output , const B& bb){
    output << bb.a << ' ' << bb.b << endl;
}


int main()
{
    A aa = A(2);
    B bb = aa;
    cout << bb << endl;
    cout <<B(aa);
}
```

#### 12.1.1 explicit

```
#include <iostream>
#include <string>
using namespace std;

class B;

class A{
public:
    typedef int int_a;
    void func(){
        cout << "A::func()" << endl;
    }
    A(const int &aa) : a(aa) {}
    A() = default;
    friend ostream& operator << (ostream& , const A& );
    int_a geta() const {
        return a;
    }
    friend class B;
private:
    int_a a;
};


ostream& operator << (ostream& output , const A& aa) {
    output << aa.a << endl;
}

class B {
public:
    friend ostream& operator << (ostream& , const B&);
    B(int aa , int bb) : a(aa) , b(bb) {};
    explicit B(const A& aa){
        this->a = aa.a;
        this->b = 10;
    }
private:
    int a;
    int b;
};



ostream& operator << (ostream& output , const B& bb){
    output << bb.a << ' ' << bb.b << endl;
}


int main()
{
    A aa = A(2);
    B bb = aa;         // error: 不允许编译器自动类型转换
    B bb = B(aa);       // ok: 只允许显式的类型转换
    cout << bb << endl;
    cout <<B(aa);
}

```

### 12.2 用类型转换操作符函数进行转换

格式：

```
class 源类{
operator 转化目标类(void){
	根据需求完成从源类型到目标类型的转换
}
}
```

```
#include <iostream>
#include <string>
using namespace std;

class B;

class A{
public:
    typedef int int_a;
    void func(){
        cout << "A::func()" << endl;
    }
    A(const int &aa) : a(aa) {}
    A() = default;
    operator B();
    friend ostream& operator << (ostream& , const A& );
    int_a geta() const {
        return a;
    }
private:
    int_a a;
};


ostream& operator << (ostream& output , const A& aa) {
    output << aa.a << endl;
}

class B {
public:
    friend ostream& operator << (ostream& , const B&);
    B(int aa , int bb) : a(aa) , b(bb) {};
private:
    int a;
    int b;
};

ostream& operator << (ostream& output , const B& bb){
    output << bb.a << ' ' << bb.b << endl;
}

A::operator B(){
    return B(10 , 20);
}


int main()
{
    A aa = A(2);
    B bb = aa;      // 类型转换  A 转成了 B 
    cout << bb;
    cout <<B(aa);   // 类型转换
}
```

