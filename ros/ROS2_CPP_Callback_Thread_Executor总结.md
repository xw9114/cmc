# ROS2 C++ 线程、回调函数与 Executor 总结

## 1. 回调函数（Callback）

### 普通函数调用

普通 C++ 函数：

``` cpp
void hello()
{
    std::cout << "hello";
}

hello();
```

特点：

-   由程序主动调用
-   执行流程由自己控制

------------------------------------------------------------------------

### ROS2 回调函数

ROS2 中，用户不会主动调用 callback，而是注册给 ROS2：

``` cpp
void callback(
    std_msgs::msg::String::SharedPtr msg)
{
    std::cout << msg->data;
}
```

然后：

``` cpp
create_subscription(
    "topic",
    10,
    callback
);
```

含义：

> 当 ROS2 收到消息时，自动调用 callback。

流程：

    消息到达
       |
       ↓
    DDS通信
       |
       ↓
    Executor检查事件
       |
       ↓
    调用callback()

------------------------------------------------------------------------

# 2. Executor（执行器）

Executor 是 ROS2 中负责调度 callback 的组件。

它负责：

-   检查是否有消息
-   检查 timer 是否触发
-   检查 service 请求
-   检查 action 事件
-   调用对应 callback

------------------------------------------------------------------------

## spin()

常见：

``` cpp
rclcpp::spin(node);
```

本质：

启动 Executor，不断等待事件。

类似：

``` cpp
while(true)
{
    检查事件;
    执行callback;
}
```

------------------------------------------------------------------------

# 3. 单线程 Executor

默认：

``` cpp
rclcpp::spin(node);
```

使用：

    SingleThreadedExecutor

结构：

    线程1

    callback A

    结束

    callback B

    结束

    callback C

特点：

-   简单
-   不会出现线程竞争
-   但是一个 callback 阻塞会影响其他任务

例如：

``` cpp
void callback()
{
    sleep(10);
}
```

那么：

    callback执行10秒

    其他callback等待

------------------------------------------------------------------------

# 4. 多线程 Executor

使用：

``` cpp
rclcpp::executors::MultiThreadedExecutor executor;

executor.add_node(node);

executor.spin();
```

结构：

    Executor

    线程1 ---> callback A

    线程2 ---> callback B

    线程3 ---> callback C

可以同时执行多个任务。

------------------------------------------------------------------------

# 5. Callback Group（回调组）

仅使用 MultiThreadedExecutor 不一定有效。

原因：

ROS2 默认所有 callback 属于同一个：

    MutuallyExclusive Callback Group

即：

    线程1执行callback

    其他callback等待

所以需要创建不同 Callback Group。

------------------------------------------------------------------------

## 创建 Callback Group

写在 Node 类里面：

``` cpp
class UAVController : public rclcpp::Node
{

public:

    UAVController()
    : Node("uav_controller")
    {

        control_group_ =
        this->create_callback_group(
            rclcpp::CallbackGroupType::MutuallyExclusive
        );


        vision_group_ =
        this->create_callback_group(
            rclcpp::CallbackGroupType::MutuallyExclusive
        );

    }


private:

    rclcpp::CallbackGroup::SharedPtr control_group_;

    rclcpp::CallbackGroup::SharedPtr vision_group_;

};
```

位置：

    Node类
     |
     |-- 创建callback group
     |
     |-- 创建timer
     |
     |-- 创建subscriber

------------------------------------------------------------------------

# 6. Callback Group 实际作用

无人机例子：

    UAV Controller Node


    control_group

        |
        ↓

    timer_callback()

    50Hz控制循环



    vision_group

        |
        ↓

    image_callback()

    视觉处理

运行：

    MultiThreadedExecutor


    线程1
     |
    控制循环


    线程2
     |
    视觉处理

这样视觉计算不会阻塞飞控控制。

------------------------------------------------------------------------

# 7. ROS2 常见 Callback 类型

## Subscriber Callback

消息到达：

``` cpp
void callback(msg)
```

触发：

    topic收到数据

------------------------------------------------------------------------

## Timer Callback

定时执行：

``` cpp
create_wall_timer()
```

例如：

    50Hz控制循环

------------------------------------------------------------------------

## Service Callback

客户端请求：

    request
     |
     ↓
    service callback

------------------------------------------------------------------------

## Action Callback

包括：

    goal callback

    feedback callback

    result callback

------------------------------------------------------------------------

# 8. std::function 与 callback

ROS2 内部大量使用函数对象。

例如：

``` cpp
std::function<void(const std::string&)>
```

表示：

保存一个：

    返回void
    参数string

的函数。

可以保存：

-   普通函数
-   Lambda
-   成员函数

------------------------------------------------------------------------

# 9. std::bind 与 \_1

例如：

``` cpp
std::bind(
    &Node::callback,
    this,
    std::placeholders::_1
)
```

含义：

-   `&Node::callback`：绑定哪个函数
-   `this`：绑定对象
-   `_1`：第一个参数以后再传

例如：

原函数：

``` cpp
callback(msg)
```

bind后：

    callback(?)

以后 ROS2 收到消息：

    callback(msg)

------------------------------------------------------------------------

# 10. ROS2 C++ 多线程整体结构

                     MultiThreadedExecutor

                      线程1       线程2

                        |           |

                 control_group   vision_group

                        |           |

                timer_callback image_callback

                        |
                        |
                 PX4控制指令

------------------------------------------------------------------------

# 11. 对无人机项目的应用

典型设计：

    UAV Node


    控制任务:

    timer callback
    50Hz/100Hz

    发送控制指令


    传感器任务:

    subscriber callback

    IMU/GPS/Camera


    任务管理:

    Action callback

    任务规划

推荐：

    control_group

    sensor_group

    mission_group

分别管理不同任务。

------------------------------------------------------------------------

# 核心记忆

    Node
     |
    保存 callback


    Executor
     |
    决定什么时候调用 callback


    Thread
     |
    真正执行 callback


    Callback Group
     |
    决定哪些 callback 可以并行

ROS2运行逻辑：

    事件发生
        |
        ↓
    Executor发现
        |
        ↓
    选择线程
        |
        ↓
    执行callback
