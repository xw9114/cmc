# ROS2 C++ Topic 学习笔记

## 1. Topic通信模型

ROS2 Topic 是异步发布-订阅通信机制。

Publisher 发布消息，Subscriber 通过 callback 接收消息。

    Publisher ---> Topic ---> Subscriber

## 2. C++节点基本结构

``` cpp
#include "rclcpp/rclcpp.hpp"

class MyNode : public rclcpp::Node
{
public:
    MyNode() : Node("my_node")
    {
    }
};

int main(int argc, char **argv)
{
    rclcpp::init(argc, argv);

    auto node = std::make_shared<MyNode>();

    rclcpp::spin(node);

    rclcpp::shutdown();
    return 0;
}
```

## 3. Publisher

创建：

``` cpp
publisher_ =
this->create_publisher<std_msgs::msg::String>(
    "chatter",
    10
);
```

发布：

``` cpp
publisher_->publish(msg);
```

## 4. Subscriber

创建：

``` cpp
subscription_ =
this->create_subscription<std_msgs::msg::String>(
    "chatter",
    10,
    std::bind(
        &NodeClass::callback,
        this,
        std::placeholders::_1
    )
);
```

回调：

``` cpp
void callback(
    const std_msgs::msg::String::SharedPtr msg
)
{
    RCLCPP_INFO(
        this->get_logger(),
        "%s",
        msg->data.c_str()
    );
}
```

## 5. spin机制

`rclcpp::spin(node)` 启动Executor，不断检测：

-   topic消息
-   timer事件
-   service请求
-   action事件

并调用对应callback。

## 6. C++与Python对应

  Python                C++
  --------------------- ---------------------
  rclpy.node.Node       rclcpp::Node
  create_publisher      create_publisher
  create_subscription   create_subscription
  self                  this
  rclpy.spin            rclcpp::spin
  msg.data              msg-\>data

## 7. 常用命令

查看topic：

``` bash
ros2 topic list
```

查看信息：

``` bash
ros2 topic info /topic
```

查看数据：

``` bash
ros2 topic echo /topic
```

## 总结

ROS2 C++ Topic核心流程：

硬件驱动 -\> Publisher -\> Topic -\> Subscriber -\> 算法节点 -\>
控制输出
