# ROS2 参数（Parameter）知识总结

> 本文整理 ROS2 中 Parameter 的核心概念、命令行操作、参数回调、Parameter Service，以及 Python / C++ 中修改自身参数和其他节点参数的方法。

---

## 1. Parameter 是什么

ROS2 中的 Parameter 可以理解成：

> **属于某个 Node 的、可以从节点外部配置的“设置项”。**

例如一个视觉节点：

```text
/vision_node
│
├── threshold = 120
├── min_area = 500
├── confidence = 0.8
└── debug = false
```

这些参数适合保存：

- 二值化阈值
- HSV 阈值
- PID 参数
- 摄像头编号
- 最小轮廓面积
- 目标颜色
- 识别置信度
- 调试开关

---

## 2. Parameter、Topic、Service 的区别

| ROS2 机制 | 主要用途 |
|---|---|
| Parameter | 配置节点 |
| Topic | 持续传输数据 |
| Service | 请求一次，响应一次 |
| Action | 长时间任务，可反馈、可取消 |

可以简单记成：

```text
Topic       → 传数据
Service     → 请求功能
Parameter   → 调设置
```

---

## 3. Parameter 属于 Node

假设节点名字是：

```python
super().__init__('vision_node')
```

那么节点名为：

```text
/vision_node
```

如果节点声明：

```python
self.declare_parameter('threshold', 120)
```

可以理解成：

```text
/vision_node
    └── threshold = 120
```

因此 Parameter 不是全局变量，而是某个节点自己的参数。

例如两个节点可以同时都有一个 `threshold`：

```text
/vision_node       threshold = 120
/camera_node       threshold = 80
```

命令行操作时要指定节点：

```bash
ros2 param get /vision_node threshold
```

---

# 4. 声明参数

## Python

```python
self.declare_parameter('threshold', 120)
self.declare_parameter('min_area', 500)
self.declare_parameter('debug', False)
```

## C++

```cpp
this->declare_parameter<int>("threshold", 120);
this->declare_parameter<int>("min_area", 500);
this->declare_parameter<bool>("debug", false);
```

含义：

```text
参数名          默认值
threshold       120
min_area        500
debug           false
```

---

# 5. 读取参数

## Python

```python
threshold = self.get_parameter('threshold').value
```

完整示例：

```python
self.declare_parameter('threshold', 120)

threshold = self.get_parameter('threshold').value

print(threshold)
```

输出：

```text
120
```

---

## C++

```cpp
int threshold =
    this->get_parameter("threshold").as_int();
```

也可以：

```cpp
auto param =
    this->get_parameter("threshold");

int threshold =
    param.as_int();
```

---

# 6. 在程序内部修改自己的参数

ROS2 提供：

```text
set_parameters()
```

用于节点在代码内部修改自己的参数。

---

## Python

```python
from rclpy.parameter import Parameter

results = self.set_parameters([
    Parameter(
        'threshold',
        Parameter.Type.INTEGER,
        160
    )
])
```

也可以一次修改多个参数：

```python
self.set_parameters([
    Parameter(
        'threshold',
        Parameter.Type.INTEGER,
        160
    ),

    Parameter(
        'min_area',
        Parameter.Type.INTEGER,
        800
    ),

    Parameter(
        'debug',
        Parameter.Type.BOOL,
        True
    )
])
```

---

## C++

```cpp
this->set_parameters({
    rclcpp::Parameter(
        "threshold",
        160
    ),

    rclcpp::Parameter(
        "min_area",
        800
    ),

    rclcpp::Parameter(
        "debug",
        true
    )
});
```

---

# 7. `set_parameters()` 为什么要传列表

Python 中：

```python
self.set_parameters([
    Parameter(...)
])
```

这里的：

```python
[param]
```

表示：

> 一个 Python 列表，里面放了一个 `Parameter` 对象。

如果有三个参数：

```python
[param1, param2, param3]
```

表示一个列表里有三个 `Parameter` 对象。

注意：

```text
[param]
```

虽然列表里只有一个对象，但是 `param` 本身是一个复杂对象，里面有很多成员。

---

# 8. `set_parameters()` 和 `set_parameters_atomically()`

普通：

```python
self.set_parameters([
    param1,
    param2,
    param3
])
```

多个参数可以分别成功或失败。

例如：

```text
param1    成功
param2    成功
param3    失败
```

而：

```python
self.set_parameters_atomically([
    param1,
    param2,
    param3
])
```

强调：

> 要么全部修改成功，要么全部失败。

例如 PID 参数：

```text
Kp
Ki
Kd
```

如果希望三个参数必须一起更新，就比较适合使用原子修改。

---

# 9. 命令行操作 Parameter

查看节点有哪些参数：

```bash
ros2 param list /vision_node
```

读取参数：

```bash
ros2 param get /vision_node threshold
```

修改参数：

```bash
ros2 param set /vision_node threshold 160
```

例如：

```text
原来：

threshold = 120

执行：

ros2 param set /vision_node threshold 160

结果：

threshold = 160
```

---

# 10. 启动节点时设置参数

可以在启动节点的时候覆盖默认值：

```bash
ros2 run my_package vision_node \
    --ros-args \
    -p threshold:=160
```

代码中虽然写的是：

```python
self.declare_parameter('threshold', 120)
```

但是启动时指定：

```text
threshold = 160
```

最终节点使用的就是 160。

---

# 11. 使用 YAML 保存参数

实际项目里通常会使用 YAML 文件保存大量参数。

例如：

```yaml
vision_node:
  ros__parameters:
    threshold: 160
    min_area: 500
    confidence: 0.8
    debug: true
```

启动：

```bash
ros2 run my_package vision_node \
    --ros-args \
    --params-file config/vision.yaml
```

实际机器人项目中可以准备：

```text
config/
├── indoor.yaml
├── outdoor.yaml
├── red_team.yaml
├── blue_team.yaml
└── debug.yaml
```

这样不需要修改代码，只需要切换配置文件。

---

# 12. Parameter 和普通程序变量不是同一个东西

例如：

```python
self.declare_parameter(
    'threshold',
    120
)

self.threshold = \
    self.get_parameter(
        'threshold'
    ).value
```

这里其实有两份数据：

```text
ROS2 Parameter：

threshold = 120


Python 成员变量：

self.threshold = 120
```

它们刚开始值相同，但不是同一个变量。

如果执行：

```bash
ros2 param set /vision_node threshold 160
```

ROS2 Parameter 会变成：

```text
threshold = 160
```

但是之前保存的：

```python
self.threshold
```

并不会因为它是普通变量就自动修改。

---

# 13. 推荐的参数读取方式

如果参数访问频率不是特别高，可以在需要使用时直接读取最新参数。

## Python

```python
def image_callback(self, msg):

    threshold = \
        self.get_parameter(
            'threshold'
        ).value

    # 使用最新 threshold
```

## C++

```cpp
void image_callback()
{
    int threshold =
        this->get_parameter(
            "threshold"
        ).as_int();

    // 使用最新 threshold
}
```

这样每次都能拿到当前 Parameter 的最新值。

---

# 14. 参数更新回调函数

ROS2 可以注册一个参数设置回调：

```python
self.add_on_set_parameters_callback(...)
```

它最重要的作用是：

> **在参数真正被设置之前，检查这次修改是否合法。**

例如限制：

```text
threshold 必须在 0 ~ 255
```

---

## Python

```python
from rcl_interfaces.msg import SetParametersResult
from rclpy.parameter import Parameter


def parameter_callback(self, params):

    for param in params:

        if param.name == 'threshold':

            if param.type_ != \
                Parameter.Type.INTEGER:

                return SetParametersResult(
                    successful=False,
                    reason='threshold 必须是整数'
                )

            if param.value < 0 or \
               param.value > 255:

                return SetParametersResult(
                    successful=False,
                    reason='threshold 必须在 0~255'
                )

    return SetParametersResult(
        successful=True
    )
```

注册：

```python
self.add_on_set_parameters_callback(
    self.parameter_callback
)
```

---

## C++

```cpp
rcl_interfaces::msg::SetParametersResult
parameter_callback(
    const std::vector<
        rclcpp::Parameter
    > & params)
{
    rcl_interfaces::msg::SetParametersResult result;

    result.successful = true;

    for (const auto & param : params)
    {
        if (
            param.get_name() ==
            "threshold")
        {
            if (
                param.get_type() !=
                rclcpp::ParameterType::
                    PARAMETER_INTEGER)
            {
                result.successful = false;

                result.reason =
                    "threshold 必须是整数";

                return result;
            }

            int value =
                param.as_int();

            if (
                value < 0 ||
                value > 255)
            {
                result.successful = false;

                result.reason =
                    "threshold 必须在 0~255";

                return result;
            }
        }
    }

    return result;
}
```

注册：

```cpp
callback_handle_ =
    this->add_on_set_parameters_callback(
        std::bind(
            &VisionNode::parameter_callback,
            this,
            std::placeholders::_1
        )
    );
```

---

# 15. 为什么参数回调里面要使用 `for`

例如：

```python
def parameter_callback(self, params):

    for param in params:
```

原因是：

```text
params
```

不是一个 Parameter，而是一组 Parameter。

可以理解成：

```python
params = [
    param1,
    param2,
    param3
]
```

所以：

```python
for param in params:
```

表示逐个取出：

```text
第一次 → param1
第二次 → param2
第三次 → param3
```

然后：

```python
if param.name == 'threshold':
```

判断当前正在检查哪个参数。

C++ 同理：

```cpp
for (const auto & param : params)
{
    if (
        param.get_name() ==
        "threshold")
    {
        // ...
    }
}
```

---

# 16. 参数修改的整体流程

命令行执行：

```bash
ros2 param set \
    /vision_node \
    threshold \
    160
```

可以理解成：

```text
命令行请求修改参数
        ↓
/vision_node 收到请求
        ↓
参数检查回调
        ↓
判断是否合法
       / \
     合法  非法
      ↓     ↓
    True   False
      ↓     ↓
参数修改   拒绝修改
```

---

# 17. Node 会自动提供 Parameter Service

创建节点：

```python
super().__init__('vision_node')
```

默认情况下 ROS2 会为节点启动参数相关 Service。

通常包括：

```text
/vision_node/get_parameters
/vision_node/set_parameters
/vision_node/list_parameters
/vision_node/describe_parameters
/vision_node/get_parameter_types
/vision_node/set_parameters_atomically
```

注意：

> 这些 Service 不是因为调用 `declare_parameter()` 才创建的。

而是在 Node 创建时，默认启动参数服务。

---

# 18. Parameter Service 的作用

这些 Service 让外部程序可以访问节点的参数。

| Service | 作用 |
|---|---|
| `get_parameters` | 读取参数 |
| `set_parameters` | 修改参数 |
| `list_parameters` | 查看有哪些参数 |
| `describe_parameters` | 获取参数描述 |
| `get_parameter_types` | 获取参数类型 |
| `set_parameters_atomically` | 原子修改多个参数 |

可以理解成：

```text
              /vision_node
                   │
          ┌────────┴────────┐
          │                 │
       Parameter        Parameter Service
          │                 │
 threshold=120       get_parameters
 min_area=500        set_parameters
 debug=false         list_parameters
                     describe_parameters
                     ...
```

Parameter 是数据。

Parameter Service 是外界访问这些数据的接口。

---

# 19. Parameter Service 的 Interface

常见 Parameter Service 对应：

| Service | Interface |
|---|---|
| `/vision_node/get_parameters` | `rcl_interfaces/srv/GetParameters` |
| `/vision_node/set_parameters` | `rcl_interfaces/srv/SetParameters` |
| `/vision_node/list_parameters` | `rcl_interfaces/srv/ListParameters` |
| `/vision_node/describe_parameters` | `rcl_interfaces/srv/DescribeParameters` |
| `/vision_node/get_parameter_types` | `rcl_interfaces/srv/GetParameterTypes` |
| `/vision_node/set_parameters_atomically` | `rcl_interfaces/srv/SetParametersAtomically` |

例如：

```bash
ros2 service type \
    /vision_node/set_parameters
```

得到：

```text
rcl_interfaces/srv/SetParameters
```

这里：

```text
/vision_node/set_parameters
        ↓
Service Name
```

而：

```text
rcl_interfaces/srv/SetParameters
        ↓
Service Interface / Type
```

其中：

```text
rcl_interfaces
        ↓
Package
```

---

# 20. 查看 Service Interface

可以：

```bash
ros2 interface show \
    rcl_interfaces/srv/SetParameters
```

核心结构可以理解成：

```text
Parameter[] parameters
---
SetParametersResult[] results
```

其中：

```text
Parameter[] parameters
```

是 Request。

```text
SetParametersResult[] results
```

是 Response。

---

# 21. 为什么是 `request.parameters = [param]`

因为 `SetParameters` 的 Request 定义的是：

```text
Parameter[] parameters
```

这里的：

```text
[]
```

表示数组。

因此 Python 中：

```python
request.parameters = [param]
```

而不是：

```python
request.parameters = param
```

因为：

```text
param
```

只是一个对象。

而：

```text
[param]
```

才是包含一个 Parameter 对象的列表。

如果有三个：

```python
request.parameters = [
    param1,
    param2,
    param3
]
```

---

# 22. Parameter 对象内部还有很多成员

例如：

```python
param = Parameter()
```

它是一个对象，但内部还有成员：

```text
param
│
├── name
│
└── value
     │
     ├── type
     ├── bool_value
     ├── integer_value
     ├── double_value
     ├── string_value
     └── ...
```

所以：

```text
[param]
```

表示：

```text
一个列表
   │
   └── 一个 Parameter 对象
             │
             └── 内部包含很多成员
```

---

# 23. Parameter 对象的属性不能随便写

ROS2 Message 的字段由接口文件提前定义。

例如 `Parameter` 可以理解成：

```text
string name
ParameterValue value
```

所以可以：

```python
param.name = 'threshold'
```

但是不能：

```python
param.age = 18
param.abc = 123
```

因为 `Parameter` 接口里没有这些字段。

要区分：

```python
param.name = 'threshold'
```

这里：

```text
name
```

是 ROS2 接口固定的成员名。

而：

```text
threshold
```

是你自己定义的参数名字。

---

# 24. 直接调用参数 Service 修改其他节点

假设：

```text
/controller_node
        │
        │
        ▼
/vision_node
```

Controller 想修改：

```text
/vision_node
    threshold = 160
```

可以直接调用：

```text
/vision_node/set_parameters
```

---

## Python：手动调用 Service

```python
from rcl_interfaces.srv import SetParameters

from rcl_interfaces.msg import (
    Parameter,
    ParameterValue,
    ParameterType
)


client = self.create_client(
    SetParameters,
    '/vision_node/set_parameters'
)


param = Parameter()

param.name = 'threshold'

param.value = ParameterValue()

param.value.type = \
    ParameterType.PARAMETER_INTEGER

param.value.integer_value = 160


request = SetParameters.Request()

request.parameters = [param]


future = client.call_async(request)
```

结构：

```text
request
│
└── parameters
      │
      └── [param]
            │
            ├── name
            │    └── threshold
            │
            └── value
                 │
                 ├── type
                 │    └── INTEGER
                 │
                 └── integer_value
                      └── 160
```

---

# 25. C++：手动调用参数 Service

```cpp
#include "rcl_interfaces/srv/set_parameters.hpp"
#include "rcl_interfaces/msg/parameter_type.hpp"


auto client =
    this->create_client<
        rcl_interfaces::srv::SetParameters>(
        "/vision_node/set_parameters"
    );


auto request =
    std::make_shared<
        rcl_interfaces::srv::
            SetParameters::Request>();


rcl_interfaces::msg::Parameter param;

param.name = "threshold";

param.value.type =
    rcl_interfaces::msg::
        ParameterType::
            PARAMETER_INTEGER;

param.value.integer_value = 160;


request->parameters.push_back(param);


auto future =
    client->async_send_request(
        request
    );
```

Python：

```python
request.parameters = [param]
```

C++：

```cpp
request->parameters.push_back(param);
```

本质上都是：

> 往 `Parameter[]` 中放一个 Parameter 对象。

---

# 26. 使用 Parameter Client 修改其他节点

自己手动创建：

```text
SetParameters
Parameter
ParameterValue
ParameterType
Request
```

比较麻烦。

ROS2 提供了专门的 Parameter Client。

---

# 27. Python：AsyncParameterClient

```python
from rclpy.parameter_client \
    import AsyncParameterClient

from rclpy.parameter \
    import Parameter


self.param_client = \
    AsyncParameterClient(
        self,
        '/vision_node'
    )
```

这里：

```text
self
 ↓
当前节点
```

而：

```text
/vision_node
 ↓
目标节点
```

注意：

```text
/vision_node
```

不是 Service 名。

它是：

> **目标 Node 名。**

Parameter Client 会自动找到：

```text
/vision_node/get_parameters
/vision_node/set_parameters
/vision_node/list_parameters
...
```

然后：

```python
future = \
    self.param_client.set_parameters([
        Parameter(
            'threshold',
            Parameter.Type.INTEGER,
            160
        )
    ])
```

---

# 28. C++：AsyncParametersClient

C++ 对应的类名是：

```text
AsyncParametersClient
```

代码：

```cpp
#include "rclcpp/rclcpp.hpp"
#include "rclcpp/parameter_client.hpp"


param_client_ =
    std::make_shared<
        rclcpp::AsyncParametersClient>(
        this,
        "/vision_node"
    );
```

修改：

```cpp
auto future =
    param_client_->set_parameters({
        rclcpp::Parameter(
            "threshold",
            160
        )
    });
```

注意：

```text
Python：
AsyncParameterClient

C++：
AsyncParametersClient
```

C++ 多了一个 `s`。

---

# 29. Python 完整示例

```python
import rclpy

from rclpy.node import Node
from rclpy.parameter import Parameter

from rcl_interfaces.msg \
    import SetParametersResult


class VisionNode(Node):

    def __init__(self):

        super().__init__(
            'vision_node'
        )

        # 声明参数
        self.declare_parameter(
            'threshold',
            120
        )

        self.declare_parameter(
            'min_area',
            500
        )

        self.declare_parameter(
            'debug',
            False
        )

        # 注册参数检查回调
        self.add_on_set_parameters_callback(
            self.parameter_callback
        )

        # 定时模拟算法执行
        self.timer = \
            self.create_timer(
                1.0,
                self.timer_callback
            )


    def parameter_callback(
        self,
        params
    ):

        for param in params:

            if param.name == \
               'threshold':

                if param.type_ != \
                   Parameter.Type.INTEGER:

                    return \
                        SetParametersResult(
                            successful=False,
                            reason=
                            'threshold 必须是整数'
                        )

                if not \
                   0 <= param.value <= 255:

                    return \
                        SetParametersResult(
                            successful=False,
                            reason=
                            'threshold 必须在0~255'
                        )

        return SetParametersResult(
            successful=True
        )


    def timer_callback(self):

        threshold = \
            self.get_parameter(
                'threshold'
            ).value

        min_area = \
            self.get_parameter(
                'min_area'
            ).value

        debug = \
            self.get_parameter(
                'debug'
            ).value

        self.get_logger().info(
            f'threshold={threshold}, '
            f'min_area={min_area}, '
            f'debug={debug}'
        )


def main(args=None):

    rclpy.init(args=args)

    node = VisionNode()

    rclpy.spin(node)

    node.destroy_node()

    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

运行：

```bash
ros2 run my_package vision_node
```

修改：

```bash
ros2 param set \
    /vision_node \
    threshold \
    180
```

成功。

如果：

```bash
ros2 param set \
    /vision_node \
    threshold \
    500
```

由于超过 `0~255`，参数回调会拒绝这次修改。

---

# 30. C++ 完整示例

```cpp
#include <memory>
#include <vector>

#include "rclcpp/rclcpp.hpp"

#include "rcl_interfaces/msg/set_parameters_result.hpp"


class VisionNode :
    public rclcpp::Node
{
public:

    VisionNode()
    : Node("vision_node")
    {
        // 声明参数

        this->declare_parameter<int>(
            "threshold",
            120
        );

        this->declare_parameter<int>(
            "min_area",
            500
        );

        this->declare_parameter<bool>(
            "debug",
            false
        );


        // 参数检查回调

        callback_handle_ =
            this->
            add_on_set_parameters_callback(
                std::bind(
                    &VisionNode::
                        parameter_callback,

                    this,

                    std::placeholders::_1
                )
            );


        // 定时模拟算法运行

        timer_ =
            this->create_wall_timer(
                std::chrono::seconds(1),

                std::bind(
                    &VisionNode::
                        timer_callback,

                    this
                )
            );
    }


private:

    rcl_interfaces::msg::
        SetParametersResult
    parameter_callback(
        const std::vector<
            rclcpp::Parameter
        > & params)
    {
        rcl_interfaces::msg::
            SetParametersResult result;

        result.successful = true;


        for (
            const auto & param :
            params)
        {
            if (
                param.get_name() ==
                "threshold")
            {
                if (
                    param.get_type() !=
                    rclcpp::
                    ParameterType::
                    PARAMETER_INTEGER)
                {
                    result.successful =
                        false;

                    result.reason =
                        "threshold 必须是整数";

                    return result;
                }


                int threshold =
                    param.as_int();


                if (
                    threshold < 0 ||
                    threshold > 255)
                {
                    result.successful =
                        false;

                    result.reason =
                        "threshold 必须在0~255";

                    return result;
                }
            }
        }


        return result;
    }


    void timer_callback()
    {
        int threshold =
            this->get_parameter(
                "threshold"
            ).as_int();


        int min_area =
            this->get_parameter(
                "min_area"
            ).as_int();


        bool debug =
            this->get_parameter(
                "debug"
            ).as_bool();


        RCLCPP_INFO(
            this->get_logger(),

            "threshold=%d "
            "min_area=%d "
            "debug=%d",

            threshold,
            min_area,
            debug
        );
    }


    rclcpp::Node::
        OnSetParametersCallbackHandle::
        SharedPtr callback_handle_;


    rclcpp::TimerBase::
        SharedPtr timer_;
};


int main(
    int argc,
    char ** argv)
{
    rclcpp::init(
        argc,
        argv
    );


    rclcpp::spin(
        std::make_shared<
            VisionNode>()
    );


    rclcpp::shutdown();


    return 0;
}
```

---

# 31. ROS2 Parameter 整体结构

```text
                         ROS2 Node
                       /vision_node
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
    Parameter 数据       Parameter API      Parameter Service
          │                 │                 │
 threshold=120       declare_parameter    get_parameters
 min_area=500        get_parameter        set_parameters
 debug=false         set_parameters       list_parameters
                     set_parameters_      describe_parameters
                     atomically           ...
          │                                   │
          │                                   │
          │                        其他节点 / 命令行访问
          │                                   │
          └───────────────────┬───────────────┘
                              │
                              ▼
                       参数修改请求
                              │
                              ▼
                  on_set_parameters_callback
                              │
                         合法性检查
                         /         \
                       合法        非法
                        │           │
                        ▼           ▼
                   参数真正修改     拒绝
```

---

# 32. 跨节点修改参数的结构

```text
/controller_node
       │
       │ Parameter Client
       │
       ▼
/vision_node/set_parameters
       │
       ▼
/vision_node
       │
       ▼
参数检查回调
       │
       ▼
threshold = 160
```

Python 通常使用：

```text
AsyncParameterClient
```

C++ 通常使用：

```text
AsyncParametersClient
```

---

# 33. 最重要的函数对照

| 功能 | Python | C++ |
|---|---|---|
| 声明参数 | `declare_parameter()` | `declare_parameter()` |
| 读取参数 | `get_parameter()` | `get_parameter()` |
| 修改自己的参数 | `set_parameters()` | `set_parameters()` |
| 原子修改参数 | `set_parameters_atomically()` | `set_parameters_atomically()` |
| 参数修改检查回调 | `add_on_set_parameters_callback()` | `add_on_set_parameters_callback()` |
| 修改其他节点参数 | `AsyncParameterClient` | `AsyncParametersClient` |
| 参数接口包 | `rcl_interfaces` | `rcl_interfaces` |

---

# 34. 常用命令总结

查看节点：

```bash
ros2 node list
```

查看参数：

```bash
ros2 param list /vision_node
```

读取参数：

```bash
ros2 param get \
    /vision_node \
    threshold
```

修改参数：

```bash
ros2 param set \
    /vision_node \
    threshold \
    160
```

查看 Service：

```bash
ros2 service list
```

查看某个 Service 的类型：

```bash
ros2 service type \
    /vision_node/set_parameters
```

查看 Interface：

```bash
ros2 interface show \
    rcl_interfaces/srv/SetParameters
```

---

# 35. 最终总结

ROS2 Parameter 可以用一句话概括：

> **Parameter 是 Node 的可配置属性。**

Node 默认通过 Parameter Service 把参数管理功能暴露给系统中的其他节点和命令行。

自己操作自己的参数：

```text
declare_parameter()
get_parameter()
set_parameters()
```

其他节点修改参数：

```text
Parameter Service
        或
Parameter Client
```

参数修改回调：

```text
add_on_set_parameters_callback()
```

主要负责：

> **检查这次参数修改是否合法。**

整个关系可以记成：

```text
创建 Node
   ↓
拥有 Parameter
   ↓
默认提供 Parameter Service
   ↓
命令行 / 其他 Node
可以读取或修改参数
   ↓
修改前经过参数回调检查
   ↓
合法 → 修改
非法 → 拒绝
```

如果把这套关系理解清楚，ROS2 中 Parameter 的核心知识基本就已经掌握了。
