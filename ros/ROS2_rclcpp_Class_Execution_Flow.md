# ROS 2 rclcpp 类关系与执行流程详解

> 适用环境：ROS 2 Humble + C++
>
> 本文目标：
>
> - 理解 rclcpp 的核心类结构
> - 理解 Node、Publisher、Subscriber、Timer 的关系
> - 理解 spin()、Executor、Callback 的执行机制

---

# 1. ROS 2 C++ 核心结构

ROS 2 C++ 使用：

```cpp
#include "rclcpp/rclcpp.hpp"
```

提供 C++ API。

核心关系：

```
rclcpp

   |
   |
   ↓

Node

   |
   ├── Publisher
   |
   ├── Subscription
   |
   ├── Timer
   |
   ├── Service
   |
   └── Parameter
```

一个 ROS 2 程序通常就是：

> 创建 Node，然后向 Node 添加通信对象。

---

# 2. Node 节点

## 2.1 Node是什么？

Node 是 ROS 2 中程序运行的基本单位。

例如：

无人机系统：

```
控制节点

传感器节点

导航节点

视觉节点
```

每一个都是一个 Node。


---

## 2.2 创建 Node

方式：

```cpp
class PublisherNode : public rclcpp::Node
{

};
```

表示：

```
PublisherNode
        |
        ↓
继承
        |
        ↓
rclcpp::Node
```

因此拥有 ROS 2 功能。

---

## 2.3 Node名字

构造：

```cpp
PublisherNode()
: Node("cpp_publisher")
{

}
```

节点名字：

```
cpp_publisher
```

查看：

```bash
ros2 node list
```

---

# 3. Publisher 发布器

## 3.1 创建 Publisher

代码：

```cpp
publisher_ =
this->create_publisher<std_msgs::msg::String>(
    "/chatter",
    10
);
```

结构：

```
Node

 |
 |
 create_publisher()

 |
 ↓

Publisher
```

---

## 3.2 Publisher作用

Publisher负责：

```
创建消息

↓

发送消息

↓

Topic广播
```

例如：

```cpp
publisher_->publish(message);
```

---

## 3.3 发布流程

完整过程：

```
message

   |
   ↓

Publisher

   |
   ↓

rclcpp

   |
   ↓

DDS

   |
   ↓

Topic

   |
   ↓

Subscriber
```

---

# 4. Subscriber 订阅器

## 4.1 创建 Subscriber

例如：

```cpp
subscription_ =
this->create_subscription<String>(
    "/chatter",
    10,
    callback
);
```

作用：

```
监听Topic

收到数据

执行callback
```

---

# 5. Callback 回调函数

## 5.1 什么是Callback？

Callback：

> 当事件发生时，ROS 2 自动调用的函数。


例如：

```cpp
void topic_callback(
    String::SharedPtr msg
)
{

}
```


流程：

```
消息到达

↓

Executor检测

↓

调用callback()

↓

用户处理数据
```

---

# 6. Timer 定时器

## 6.1 create_wall_timer()

代码：

```cpp
timer_ =
this->create_wall_timer(
    1s,
    callback
);
```


意思：

```
每隔1秒

调用callback()
```

---

## 6.2 Timer流程

```
Timer

↓

等待时间

↓

Executor检测

↓

执行callback
```

---

# 7. rclcpp::spin()

## 7.1 作用

代码：

```cpp
rclcpp::spin(node);
```

作用：

让节点持续运行。


没有：

```
main()

↓

程序结束
```


有：

```
main()

↓

spin()

↓

等待事件

↓

执行回调

↓

继续等待
```

---

# 8. Executor 执行器

Executor 是 ROS 2 回调调度器。


结构：

```
Executor

 |
 |
 +---- Timer事件

 |
 +---- Subscriber事件

 |
 +---- Service事件

 |
 +---- Action事件
```

---

# 9. SingleThreadedExecutor

默认：

```cpp
rclcpp::spin(node);
```

使用：

```
SingleThreadedExecutor
```


特点：

只有一个线程：

```
线程1

 |
 +-- Timer

 |
 +-- Subscriber

 |
 +-- Service
```


简单稳定。

---

# 10. MultiThreadedExecutor

多线程执行：

```cpp
rclcpp::executors::MultiThreadedExecutor executor;
```

结构：

```
Executor

 |
 +--线程1

 |
 +--线程2

 |
 +--线程3
```


适合：

- 多传感器
- 高实时控制
- 无人机系统

---

# 11. 一个完整节点执行流程


程序启动：

```
main()

↓

rclcpp::init()

↓

创建Node

↓

创建Publisher

↓

创建Timer

↓

spin()

```

之后：

```
Executor运行

↓

等待事件

↓

Timer触发

↓

callback()

↓

publish()

↓

Topic发送数据

```

---

# 12. Topic完整通信流程


Publisher：

```
Node

↓

Publisher

↓

publish(message)

↓

DDS

↓

Topic

↓

DDS

↓

Subscriber

↓

callback()

```

---

# 13. ROS 2和无人机对应关系


## 速度控制

Topic：

```
/cmd_vel
```

消息：

```
geometry_msgs/msg/Twist
```


结构：

```
Twist

 |
 +-- linear

 |
 +-- angular
```


控制：

```cpp
cmd.linear.x = 1.0;

publisher_->publish(cmd);
```

---

## IMU数据

Topic：

```
/imu
```

消息：

```
sensor_msgs/msg/Imu
```


流程：

```
IMU节点

↓

Publisher

↓

/imu

↓

Controller

↓

Subscriber
```

---

# 14. Node内部结构总结


```
Node

 |
 |
 +-- Publisher

 |       |
 |       ↓
 |     Topic
 |
 |
 +-- Subscriber

 |       |
 |       ↓
 |    Callback
 |
 |
 +-- Timer

         |
         ↓
      Callback

```

---

# 15. 常见函数对应关系

|功能|函数|
|-|-|
|初始化|rclcpp::init()|
|运行节点|rclcpp::spin()|
|关闭|rclcpp::shutdown()|
|创建发布器|create_publisher()|
|发布消息|publish()|
|创建订阅器|create_subscription()|
|创建定时器|create_wall_timer()|
|日志|RCLCPP_INFO()|
|创建服务|create_service()|
|创建客户端|create_client()|

---

# 16. 学习路线

建议：

```
Node

↓

Publisher

↓

Subscriber

↓

Timer

↓

Message

↓

Executor

↓

Service

↓

Action

↓

QoS

```

---

# 总结

ROS 2 C++核心思想：

```
Node
 |
 +-- 创建通信对象
 |
 +-- Executor管理事件
 |
 +-- Callback处理数据
 |
 +-- Publisher发送
 |
 +-- Subscriber接收
```


理解：

```
Node
Publisher
Subscriber
Timer
Executor
Callback
```

即可掌握大部分 ROS 2 C++ 节点开发。
