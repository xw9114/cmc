# ROS 2 C++ 常用 API 函数速查（rclcpp）

## 1. ROS 2 C++ 基本结构

核心包：

-   rclcpp：ROS 2 C++ 客户端库
-   std_msgs / geometry_msgs / sensor_msgs：消息类型包

基本流程：

``` cpp
rclcpp::init(argc, argv);
rclcpp::spin(node);
rclcpp::shutdown();
```

------------------------------------------------------------------------

# 2. 节点 Node

创建节点：

``` cpp
class MyNode : public rclcpp::Node
{
};
```

构造：

``` cpp
MyNode()
: Node("node_name")
{}
```

常用：

## get_logger()

获取日志器：

``` cpp
this->get_logger()
```

------------------------------------------------------------------------

# 3. Publisher 发布器

创建：

``` cpp
create_publisher<MessageType>(
    topic,
    queue_size
);
```

例：

``` cpp
publisher_ =
this->create_publisher<std_msgs::msg::String>(
"/chatter",
10
);
```

发布：

``` cpp
publisher_->publish(message);
```

------------------------------------------------------------------------

# 4. Subscriber 订阅器

创建：

``` cpp
create_subscription<MessageType>(
    topic,
    queue,
    callback
);
```

回调：

``` cpp
void callback(
const MessageType::SharedPtr msg)
{
}
```

读取：

``` cpp
msg->data
```

------------------------------------------------------------------------

# 5. Timer 定时器

真实时间：

``` cpp
create_wall_timer(
1s,
callback
);
```

作用：

每隔固定时间执行回调。

ROS 时间：

``` cpp
create_timer()
```

适用于仿真时间。

------------------------------------------------------------------------

# 6. 日志

信息：

``` cpp
RCLCPP_INFO(
this->get_logger(),
"message"
);
```

警告：

``` cpp
RCLCPP_WARN()
```

错误：

``` cpp
RCLCPP_ERROR()
```

------------------------------------------------------------------------

# 7. 消息 Message

格式：

    package/msg/type

例如：

    std_msgs/msg/String
    geometry_msgs/msg/Twist

查看：

``` bash
ros2 interface show 类型
```

------------------------------------------------------------------------

# 8. 常见消息字段

String:

``` cpp
msg.data
```

Twist:

``` cpp
msg.linear.x
msg.linear.y
msg.linear.z

msg.angular.x
msg.angular.y
msg.angular.z
```

Pose:

``` cpp
msg.position.x
msg.position.y
msg.position.z
```

------------------------------------------------------------------------

# 9. Service

创建服务：

``` cpp
create_service<ServiceType>()
```

创建客户端：

``` cpp
create_client<ServiceType>()
```

------------------------------------------------------------------------

# 10. Action

服务长时间任务：

``` cpp
create_action_server()
create_action_client()
```

常用于：

-   导航
-   机械臂
-   飞行任务

------------------------------------------------------------------------

# 11. Parameter 参数

声明：

``` cpp
declare_parameter()
```

读取：

``` cpp
get_parameter()
```

设置：

``` cpp
set_parameter()
```

------------------------------------------------------------------------

# 12. 智能指针

ROS 2 常见：

``` cpp
SharedPtr
```

例如：

``` cpp
rclcpp::Publisher<String>::SharedPtr
```

本质：

``` cpp
std::shared_ptr
```

------------------------------------------------------------------------

# 13. 回调绑定

成员函数：

``` cpp
std::bind(
&ClassName::callback,
this
)
```

作用：

将类函数交给 ROS 2 调用。

Lambda：

``` cpp
[this]()
{
callback();
}
```

------------------------------------------------------------------------

# 14. QoS

创建：

``` cpp
create_publisher<T>(
topic,
10
);
```

10：

消息缓存深度。

------------------------------------------------------------------------

# 15. 常用命令

查看 Topic：

``` bash
ros2 topic list
```

查看类型：

``` bash
ros2 topic type /topic
```

查看结构：

``` bash
ros2 interface show 类型
```

查看数据：

``` bash
ros2 topic echo /topic
```

发布：

``` bash
ros2 topic pub
```

------------------------------------------------------------------------

# 16. ROS 2 C++ 节点模板

``` cpp
class NodeDemo : public rclcpp::Node
{
public:

NodeDemo()
: Node("node")
{

publisher_ =
create_publisher<Message>(
"/topic",
10
);

timer_ =
create_wall_timer(
1s,
std::bind(
&NodeDemo::callback,
this
)
);

}


private:

void callback()
{
Message msg;
publisher_->publish(msg);
}


rclcpp::Publisher<Message>::SharedPtr publisher_;

rclcpp::TimerBase::SharedPtr timer_;

};
```

------------------------------------------------------------------------

# 17. 学习顺序

推荐：

1.  Node
2.  Publisher
3.  Subscriber
4.  Timer
5.  Message
6.  Service
7.  Action
8.  Parameter
9.  QoS
10. Executor

------------------------------------------------------------------------

# 总结

ROS 2 C++ 最核心函数：

  功能     API
  -------- ---------------------
  初始化   rclcpp::init
  运行     rclcpp::spin
  关闭     rclcpp::shutdown
  发布器   create_publisher
  发布     publish
  订阅     create_subscription
  定时     create_wall_timer
  日志     RCLCPP_INFO
  服务     create_service
  客户端   create_client
  参数     declare_parameter

掌握这些即可完成大部分 ROS 2 C++ 节点开发。
