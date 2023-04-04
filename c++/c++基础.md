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

