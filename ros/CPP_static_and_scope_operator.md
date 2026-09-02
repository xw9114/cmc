# C++ static 静态成员详解

## 1. static 是什么？

C++ 中：

> static 表示变量或函数属于类本身，而不是某一个对象。

核心区别：

```
普通成员：
属于对象

static成员：
属于类
```

---

# 2. 普通成员变量

```cpp
class Robot
{
public:
    int speed;
};
```

创建对象：

```cpp
Robot r1;
Robot r2;
```

结构：

```
r1
 |
 └── speed


r2
 |
 └── speed
```

两个对象拥有不同的 speed：

```cpp
r1.speed = 10;
r2.speed = 20;
```

结果：

```
r1.speed = 10
r2.speed = 20
```

因为普通成员属于对象。

---

# 3. static 成员变量

```cpp
class Robot
{
public:
    static int count;
};
```

表示：

> count 属于 Robot 类。

结构：

```
Robot类

 |
 └── count


所有Robot对象共享
```

访问：

```cpp
Robot::count = 10;
```

所有对象看到的都是同一个值。

---

# 4. static 为什么叫静态？

static 表示生命周期固定。

普通变量：

```
创建对象
    ↓
产生变量
    ↓
对象销毁
    ↓
变量消失
```

static：

```
程序启动
    ↓
创建
    ↓
程序运行期间一直存在
    ↓
程序结束
    ↓
销毁
```

---

# 5. static 成员函数

例如：

```cpp
class Robot
{
public:

    static void hello()
    {
        cout << "hello";
    }

};
```

调用：

```cpp
Robot::hello();
```

不需要创建对象。

因为：

```
hello属于Robot类
```

---

# 6. static 函数限制

静态函数不能直接访问普通成员。

错误：

```cpp
class Robot
{
    int speed;

    static void test()
    {
        speed = 10;
    }
};
```

原因：

```
speed属于对象

static函数没有对象
```

它不知道修改哪个对象的数据。

---

# 7. :: 作用域解析符

`::` 的作用：

> 指定某个名字属于哪个作用域。

---

## 命名空间

```cpp
namespace robot
{
    void start()
    {

    }
}
```

调用：

```cpp
robot::start();
```

表示：

```
robot命名空间里的start
```

---

## 类外定义成员函数

类：

```cpp
class Car
{
public:

    void run();

};
```

实现：

```cpp
void Car::run()
{

}
```

表示：

```
run属于Car类
```

---

# 8. . 和 :: 的区别

## 对象访问

使用：

```cpp
.
```

例如：

```cpp
Robot robot;

robot.speed = 10;

robot.run();
```

表示：

```
对象robot的成员
```

---

## 类访问

使用：

```cpp
::
```

例如：

```cpp
Robot::count;
Robot::hello();
```

表示：

```
Robot类的成员
```

通常用于 static 成员。

---

# 9. ROS 2 中的例子

## rclcpp::Node

```cpp
rclcpp::Node
```

拆开：

```
rclcpp
    ↓
namespace

Node
    ↓
class
```

表示：

```
rclcpp命名空间里的Node类
```

---

## PublisherNode::timer_callback

```cpp
&PublisherNode::timer_callback
```

表示：

```
PublisherNode类里的timer_callback成员函数
```

这里不是调用，而是获取成员函数指针。

---

# 10. 成员函数指针

例如：

```cpp
&PublisherNode::timer_callback
```

含义：

```
获取函数地址
```

配合：

```cpp
std::bind(
    &PublisherNode::timer_callback,
    this
)
```

表示：

```
函数
+
当前对象
```

最终调用：

```cpp
this->timer_callback();
```

---

# 11. 总结表

|类型|属于|访问方式|
|-|-|-|
|普通变量|对象|对象.变量|
|普通函数|对象|对象.函数()|
|static变量|类|类::变量|
|static函数|类|类::函数()|

---

# 最重要记忆

```
普通成员：

对象.成员


static成员：

类::成员


::
=
告诉C++这个东西属于谁
```

在 ROS 2 C++ 中：

```cpp
rclcpp::Node
```

表示：

```
rclcpp里的Node类
```

```cpp
PublisherNode::timer_callback
```

表示：

```
PublisherNode里的成员函数
```

理解 static 和 :: 后，ROS 2 C++ 中大量出现的：

- rclcpp::Node
- rclcpp::Publisher
- ClassName::callback

都会容易理解。
