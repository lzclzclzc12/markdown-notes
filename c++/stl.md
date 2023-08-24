# unordered_map

## 自定义比较函数和哈希函数

### 方法1：

```
class Person{
public:
	string name;
	int age;
	Person(string n, int a){
		name = n;
		age = a;
	}
};

size_t person_hash(const Person &p)  // 自定义哈希函数
{
    return p.age;
}

// 当输入的哈希值相同时，调用此函数，看是否为重复元素，因为有可能是碰撞
bool person_equal(const Person &p1 , const Person &p2){   // 自定义比较函数
    return p1.age == p2.age;
}

int main()
{
	// decltype 获取函数类型  ，必须加&， decltype(&person_hash) 直接获得函数指针
    unordered_map<Person,int,decltype(&person_hash) , decltype(&person_equal)> m(0, person_hash , person_equal);
    m.insert(pair<Person , int>(Person("lzc" , 20) , 1));
    m.insert(pair<Person , int>(Person("lzc" , 30) , 2));
    m.insert(pair<Person , int>(Person("lzc1" , 30) , 2));

    for(auto it = m.begin() ; it != m.end() ; ++it){
        cout << it -> first.age << ' ' << it -> first.name << ' ' << it -> second << endl;
    }
}
```

### 方法2：

```
int main()
{
    auto person_hash = [](const Person &p) -> size_t {
        return p.age;
    };
    auto person_equal = [](const Person &p1 , const Person &p2) -> bool {
        return p1.age == p2.age;
    };
	// 不能加&， 获得可调用对象
    unordered_map<Person,int,decltype(person_hash) , decltype(person_equal)> m(0, person_hash , person_equal);
    m.insert(pair<Person , int>(Person("lzc" , 20) , 1));
    m.insert(pair<Person , int>(Person("lzc" , 30) , 2));
    m.insert(pair<Person , int>(Person("lzc1" , 30) , 2));

    for(auto it = m.begin() ; it != m.end() ; ++it){
        cout << it -> first.age << ' ' << it -> first.name << ' ' << it -> second << endl;
    }
}
```

### 方法3：

```
class Person{
public:
	string name;
	int age;

	Person(string n, int a){
		name = n;
		age = a;
	}
	bool operator == (const Person &p2) const {
        return this->age == p2.age;
	}
};

// 重载operator()的类
struct Harsh{
    size_t operator()(const Person &p) const{
        return p.age;
    }
};

int main()
{

    unordered_map<Person,int, Harsh> m;
    m.insert(pair<Person , int>(Person("lzc" , 20) , 1));
    m.insert(pair<Person , int>(Person("lzc" , 30) , 2));
    m.insert(pair<Person , int>(Person("lzc1" , 30) , 2));

    for(auto it = m.begin() ; it != m.end() ; ++it){
        cout << it -> first.age << ' ' << it -> first.name << ' ' << it -> second << endl;
    }
}
```

