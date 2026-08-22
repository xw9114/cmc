# ROS2 Topic + Service 综合项目知识总结

> 环境：ROS2 Humble + Ubuntu 22.04 + Python  
> 项目目标：使用自定义 `msg` 通过 Topic 实时传输 `x、y` 数据，同时使用自定义 `srv` 让 Client 按需获取 Subscriber 当前保存的最新 `x、y`。

---

## 1. 项目最终通信结构

本项目包含 3 个主要节点：

```text
Publisher
    │
    │ 发布 Data(x, y)
    ↓
 /data Topic
    │
    │ Subscriber 实时接收
    ↓
Subscriber + Service Server
    │
    ├── self.x = msg.x
    ├── self.y = msg.y
    │
    │ 保存最新值
    ↓
 /get_xy Service
    ↑
    │ Request {}
    │
Client
    │
    ↓
Response(x, y)
```

三个程序分别负责：

```text
publish.py
    → 持续产生并发布 x、y

subscribe.py
    → 订阅 /data
    → 保存最新 x、y
    → 同时提供 /get_xy Service

client.py
    → 向 /get_xy 发送 Request
    → 异步等待 Response
    → 使用 add_done_callback() 处理返回值
```

---

# 2. Topic 和 Service 的分工

## 2.1 Topic

Topic 适合：

- 连续数据
- 实时数据
- 传感器数据
- 一对多通信
- 不要求“一问一答”

本项目：

```text
Publisher → /data → Subscriber
```

Publisher 不关心有没有 Subscriber。

---

## 2.2 Service

Service 适合：

- 请求一次、返回一次
- 查询当前状态
- 执行一次动作
- Client / Server 模式

本项目：

```text
Client → Request → Service Server
Client ← Response ← Service Server
```

Service Server 平时等待，只有收到 Request 才执行回调。

---

# 3. 自定义接口包 `my_topic`

推荐目录：

```text
chapter3/
└── src/
    └── my_topic/
        ├── msg/
        │   └── Data.msg
        │
        ├── srv/
        │   └── GetXY.srv
        │
        ├── CMakeLists.txt
        └── package.xml
```

`my_topic` 是接口包，建议使用：

```text
ament_cmake
```

而不是 `ament_python`。

---

# 4. 自定义 `Data.msg`

文件：

```text
my_topic/msg/Data.msg
```

内容：

```text
float64 x
float64 y
```

因此 Python 中：

```python
msg = Data()

msg.x = 1.0
msg.y = 2.0
```

注意：

```text
Data.msg 中字段叫什么
        ↓
Python 对象属性就叫什么
```

例如：

```text
float64 data
```

对应：

```python
msg.data
```

而：

```text
float64 x
float64 y
```

对应：

```python
msg.x
msg.y
```

---

# 5. 自定义 `GetXY.srv`

文件：

```text
my_topic/srv/GetXY.srv
```

内容：

```text
---
float64 x
float64 y
```

`---` 上面为空，表示 Request 没有字段。

含义：

```text
Request：
不需要给 Server 任何参数

---

Response：
Server 返回 x
Server 返回 y
```

因此 Client 创建请求：

```python
request = GetXY.Request()
```

终端调用：

```bash
ros2 service call /get_xy my_topic/srv/GetXY "{}"
```

其中：

```text
{}
```

就是一个空 Request。

---

# 6. `my_topic/CMakeLists.txt`

如果同时生成 `msg` 和 `srv`，一定要放在**同一个** `rosidl_generate_interfaces()` 中。

正确写法：

```cmake
cmake_minimum_required(VERSION 3.8)
project(my_topic)

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/Data.msg"
  "srv/GetXY.srv"
)

ament_export_dependencies(rosidl_default_runtime)

ament_package()
```

## 常见错误：写两次 `rosidl_generate_interfaces`

错误：

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/Data.msg"
)

rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/GetXY.srv"
)
```

可能报：

```text
add_custom_target cannot create target "my_topic"
because another target with the same name already exists
```

原因：

```text
第一次生成目标 my_topic
第二次又生成目标 my_topic
        ↓
名称冲突
```

所以必须合并成：

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/Data.msg"
  "srv/GetXY.srv"
)
```

---

# 7. `my_topic/package.xml` 关键依赖

接口包中通常需要：

```xml
<buildtool_depend>ament_cmake</buildtool_depend>

<build_depend>rosidl_default_generators</build_depend>

<exec_depend>rosidl_default_runtime</exec_depend>

<member_of_group>rosidl_interface_packages</member_of_group>
```

---

# 8. Python 中导入自定义接口

自定义消息：

```python
from my_topic.msg import Data
```

自定义 Service：

```python
from my_topic.srv import GetXY
```

注意：

```python
from my_topic.smg import Data
```

是错误的。

正确：

```text
msg ✅
smg ❌
```

---

# 9. Publisher 节点

`publish.py` 示例：

```python
import rclpy
from rclpy.node import Node

from my_topic.msg import Data


class Publisher(Node):

    def __init__(self):
        super().__init__('publisher')

        self.pub = self.create_publisher(
            Data,
            'data',
            10
        )

        self.timer = self.create_timer(
            1.0,
            self.callback
        )

        self.x = 0.0

    def callback(self):

        # 当前为模拟数据
        # 真正使用时可替换为传感器数据
        self.x += 0.1
        y = 2.0 * self.x

        msg = Data()

        msg.x = self.x
        msg.y = y

        self.pub.publish(msg)

        self.get_logger().info(
            f'Publish: x={msg.x:.2f}, y={msg.y:.2f}'
        )


def main(args=None):

    rclpy.init(args=args)

    node = Publisher()

    rclpy.spin(node)

    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

# 10. Publisher 中的核心 API

创建 Publisher：

```python
self.create_publisher(
    消息类型,
    Topic名字,
    QoS深度
)
```

例如：

```python
self.pub = self.create_publisher(
    Data,
    'data',
    10
)
```

发布：

```python
self.pub.publish(msg)
```

---

# 11. Timer

创建定时器：

```python
self.timer = self.create_timer(
    1.0,
    self.callback
)
```

含义：

```text
每 1 秒
    ↓
执行一次 callback()
```

错误写法：

```python
self.create.timer(...)
```

正确：

```python
self.create_timer(...)
```

---

# 12. Subscriber + Service Server 节点

这是整个项目最关键的节点。

它同时完成：

```text
Subscriber
+
Service Server
```

参考代码：

```python
import rclpy
from rclpy.node import Node

from my_topic.msg import Data
from my_topic.srv import GetXY


class Subscriber(Node):

    def __init__(self):
        super().__init__('subscriber')

        # Topic Subscriber
        self.sub = self.create_subscription(
            Data,
            'data',
            self.topic_callback,
            10
        )

        # Service Server
        self.srv = self.create_service(
            GetXY,
            'get_xy',
            self.service_callback
        )

        # 保存最新数据
        self.x = 0.0
        self.y = 0.0

    def topic_callback(self, msg):

        self.x = msg.x
        self.y = msg.y

        self.get_logger().info(
            f'Received: x={self.x:.2f}, y={self.y:.2f}'
        )

    def service_callback(self, request, response):

        response.x = self.x
        response.y = self.y

        self.get_logger().info(
            f'Response: x={response.x:.2f}, y={response.y:.2f}'
        )

        return response


def main(args=None):

    rclpy.init(args=args)

    node = Subscriber()

    rclpy.spin(node)

    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

# 13. Subscriber 为什么需要 `self.x` 和 `self.y`

Topic callback：

```python
def topic_callback(self, msg):

    self.x = msg.x
    self.y = msg.y
```

不断更新：

```text
最新 x
最新 y
```

Service callback：

```python
def service_callback(self, request, response):

    response.x = self.x
    response.y = self.y

    return response
```

读取当前保存的最新值。

因此：

```text
self.x
self.y
```

就是 Topic 与 Service 之间的“数据桥梁”。

---

# 14. Subscriber 的正确 API

错误：

```python
self.create_subscriber(...)
```

ROS2 Python 中没有这个函数。

正确：

```python
self.create_subscription(...)
```

记忆：

```text
Publisher     → create_publisher()
Subscriber    → create_subscription()
Service Server→ create_service()
Service Client→ create_client()
Timer         → create_timer()
```

---

# 15. Service Client

本项目使用异步 Client：

```python
call_async()
+
add_done_callback()
```

参考代码：

```python
import rclpy
from rclpy.node import Node

from my_topic.srv import GetXY


class Client(Node):

    def __init__(self):
        super().__init__('client')

        self.client = self.create_client(
            GetXY,
            'get_xy'
        )

        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info(
                '等待 get_xy service...'
            )

    def send_request(self):

        request = GetXY.Request()

        future = self.client.call_async(request)

        self.get_logger().info('Request 已发送')

        future.add_done_callback(
            self.callback
        )

        return future

    def callback(self, future):

        try:
            response = future.result()

            self.get_logger().info(
                f'x={response.x:.2f}, y={response.y:.2f}'
            )

        except Exception as e:

            self.get_logger().error(
                f'失败: {e}'
            )


def main(args=None):

    rclpy.init(args=args)

    node = Client()

    node.send_request()

    rclpy.spin(node)

    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

# 16. `create_client()`

正确：

```python
self.client = self.create_client(
    GetXY,
    'get_xy'
)
```

错误：

```python
self.create.client(...)
```

原因：

`create_client` 是 Node 的一个完整方法名。

---

# 17. `wait_for_service()`

代码：

```python
while not self.client.wait_for_service(timeout_sec=1.0):

    self.get_logger().info(
        '等待 get_xy service...'
    )
```

作用：

```text
Client 启动
    ↓
检查 /get_xy 是否存在
    ↓
不存在
    ↓
最多等待 1 秒
    ↓
返回 False
    ↓
while 再检查
```

重点：

```python
timeout_sec=1.0
```

只是：

> 每次检查 Service 是否可用时最多等 1 秒。

它**不是 Response 的 1 秒超时**。

因此先启动 Client 完全可以：

```text
Client
↓
等待 get_xy service...
等待 get_xy service...
等待 get_xy service...

Server 启动
↓
Client 检测到 Service
↓
发送 Request
```

---

# 18. `GetXY.Request()`

正确：

```python
request = GetXY.Request()
```

错误：

```python
request = GetXY.request()
```

ROS2 自动生成：

```text
GetXY
├── Request
└── Response
```

所以：

```python
GetXY.Request()
GetXY.Response()
```

首字母必须大写。

---

# 19. `call_async()`

```python
future = self.client.call_async(request)
```

含义：

```text
发送 Request
    ↓
立即返回 Future
```

它不会一直阻塞等待 Response。

`future` 可以理解为：

> “未来会得到的 Service Response”。

---

# 20. `add_done_callback()`

本项目：

```python
future.add_done_callback(
    self.callback
)
```

意思：

> 当 Future 完成，也就是 Response 回来以后，自动执行 `self.callback(future)`。

完整过程：

```text
call_async(request)
      ↓
得到 future
      ↓
add_done_callback()
      ↓
rclpy.spin(node)
      ↓
Executor持续处理事件
      ↓
Server返回Response
      ↓
future完成
      ↓
自动执行 callback(future)
```

---

# 21. `future.result()`

在完成回调中：

```python
response = future.result()
```

得到：

```text
GetXY.Response
```

然后：

```python
response.x
response.y
```

读取 Server 返回的数据。

---

# 22. `rclpy.spin(node)` 的真正作用

非常重要。

```python
rclpy.spin(node)
```

不是：

> 等一次 Service Response。

而更接近：

```c
while (1)
{
    处理 ROS2 事件;
}
```

它持续处理：

- Topic 消息
- Service Request
- Service Response
- Timer
- Future
- callback
- 其他 ROS2 事件

因此：

```python
node.send_request()
rclpy.spin(node)
node.destroy_node()
```

正常流程不是：

```text
发送 Request
↓
spin 等一下
↓
spin 自动结束
↓
destroy
```

而是：

```text
发送 Request
↓
进入 spin
↓
一直运行
↓
Response 到达
↓
callback 执行
↓
spin 仍继续运行
↓
Ctrl+C / shutdown 后
↓
离开 spin
↓
destroy_node()
```

所以 Response 返回慢，并不会让 `spin()` 自动结束。

---

# 23. `get_logger()` 常见错误

错误：

```python
self.get_logger.info(...)
```

这会报：

```text
'function' object has no attribute 'info'
```

因为：

```python
self.get_logger
```

只是函数本身。

正确：

```python
self.get_logger().info(...)
```

先执行：

```python
self.get_logger()
```

获得 Logger 对象，再调用：

```python
.info(...)
```

同理：

```python
self.get_logger().error(...)
self.get_logger().warn(...)
```

---

# 24. Python 浮点数显示很多小数位

运行时可能看到：

```text
0.30000000000000004
0.7999999999999999
16.399999999999963
```

这不是 ROS2 错误。

这是二进制浮点数的正常精度现象。

例如：

```python
0.1 + 0.1 + 0.1
```

在计算机中不一定恰好等于数学上的 `0.3`。

推荐只在显示时格式化：

```python
f'{msg.x:.2f}'
```

例如：

```python
self.get_logger().info(
    f'x={msg.x:.2f}, y={msg.y:.2f}'
)
```

输出：

```text
x=16.40, y=32.80
```

---

# 25. `float64` 并不会因为显示很长而传输很慢

例如：

```text
16.399999999999963
```

虽然显示字符很多，但：

```text
float64
```

始终占：

```text
64 bit = 8 bytes
```

因此：

```srv
---
float64 x
float64 y
```

数据量非常小。

大致只有：

```text
x → 8 bytes
y → 8 bytes
```

不可能因为小数位多导致 Service Response 传输很久。

真正可能较大的情况是：

```srv
---
float64[] data
```

如果其中包含数十万甚至数百万个元素，才需要考虑：

- 序列化
- 网络传输
- DDS 分包
- 内存
- middleware 限制
- QoS

---

# 26. `failed to send response (timeout)`

项目中曾出现：

```text
RuntimeWarning:
failed to send response (timeout):
client will not receive response
```

含义：

> Service Server 已经准备返回 Response，但在发送给对应 Client 时失败。

这不等价于：

> float 太长。

也不等价于：

> `rclpy.spin()` 自动结束。

更常见原因：

```text
Client发送Request
      ↓
Server收到Request
      ↓
Client被Ctrl+C
或异常退出
或节点被销毁
或进程重新启动
      ↓
Server准备发送Response
      ↓
原来的Client已经不存在
      ↓
failed to send response
```

---

# 27. Client 常见异常

## 27.1 方法名写错

错误：

```python
self.create.client(...)
```

正确：

```python
self.create_client(...)
```

---

## 27.2 Request 大小写错误

错误：

```python
GetXY.request()
```

正确：

```python
GetXY.Request()
```

---

## 27.3 Logger 少括号

错误：

```python
self.get_logger.info(...)
```

正确：

```python
self.get_logger().info(...)
```

---

## 27.4 `future.result()` 处理异常

推荐：

```python
def callback(self, future):

    try:
        response = future.result()

        self.get_logger().info(
            f'x={response.x}, y={response.y}'
        )

    except Exception as e:

        self.get_logger().error(
            f'失败: {e}'
        )
```

也可以：

```python
if future.exception() is not None:

    self.get_logger().error(
        f'Future异常: {future.exception()}'
    )

    return
```

---

# 28. `setup.py` 注册可执行节点

`topic/setup.py`：

```python
entry_points={
    'console_scripts': [
        'pub = topic.publish:main',
        'sub = topic.subscribe:main',
    ],
},
```

`service/setup.py`：

```python
entry_points={
    'console_scripts': [
        'client = service.client:main',
    ],
},
```

格式：

```text
ros2运行名 = Python包名.Python文件名:main
```

例如：

```python
'pub = topic.publish:main'
```

对应：

```text
ros2 run topic pub
        ↓
topic/publish.py
        ↓
main()
```

---

# 29. `setup.py` 常见错误

错误：

```python
'pub=topic.publish.main'
```

正确：

```python
'pub=topic.publish:main'
```

注意：

```text
publish:main ✅
publish.main ❌
```

每一项后面也要有逗号：

```python
'pub = topic.publish:main',
'sub = topic.subscribe:main',
```

---

# 30. 创建 Python 包时依赖怎么写

如果 Python 包要使用：

```python
from my_topic.msg import Data
```

创建包时依赖写的是：

```text
my_topic
```

不是：

```text
my_topic.msg
my_topic.smg
```

例如：

```bash
ros2 pkg create topic \
  --build-type ament_python \
  --dependencies rclpy my_topic
```

Service Client 包：

```bash
ros2 pkg create service \
  --build-type ament_python \
  --dependencies rclpy my_topic
```

记忆：

```text
创建 package 时：
--dependencies my_topic
               ↑
             包名


Python 中：
from my_topic.msg import Data
     ↑        ↑          ↑
    包名     模块       接口名
```

---

# 31. 编译位置

推荐：

```bash
cd ~/chapter3
colcon build
```

不要长期在：

```text
~/chapter3/src
```

里面编译。

标准工作空间：

```text
chapter3/
├── src/
├── build/
├── install/
└── log/
```

`colcon build` 在：

```text
chapter3/
```

执行。

---

# 32. 修改代码后重新编译

标准流程：

```bash
cd ~/chapter3
colcon build
source install/setup.bash
```

如果接口发生较大修改，出现奇怪缓存问题，可尝试：

```bash
cd ~/chapter3

rm -rf build install log

colcon build

source install/setup.bash
```

---

# 33. Bash 环境加载

ROS2 Humble：

```bash
source /opt/ros/humble/setup.bash
```

工作空间：

```bash
source ~/chapter3/install/setup.bash
```

建议 Bash 终端使用：

```text
setup.bash
```

而不是优先使用：

```text
setup.sh
```

---

# 34. 检查可执行节点

Topic 包：

```bash
ros2 pkg executables topic
```

成功后可能显示：

```text
topic pub
topic sub
```

Service 包：

```bash
ros2 pkg executables service
```

显示：

```text
service client
```

---

# 35. 验证自定义接口

查看自定义 msg：

```bash
ros2 interface show my_topic/msg/Data
```

应该类似：

```text
float64 x
float64 y
```

查看自定义 srv：

```bash
ros2 interface show my_topic/srv/GetXY
```

应该类似：

```text
---
float64 x
float64 y
```

---

# 36. Topic 常用命令

## 查看所有 Topic

```bash
ros2 topic list
```

## 查看 Topic 信息

```bash
ros2 topic info /data
```

## 查看 Topic 类型

```bash
ros2 topic type /data
```

## 实时查看消息

```bash
ros2 topic echo /data
```

---

# 37. Service 四个核心命令

## 37.1 `list`

```bash
ros2 service list
```

查看当前所有 Service。

---

## 37.2 `type`

```bash
ros2 service type /get_xy
```

例如：

```text
my_topic/srv/GetXY
```

含义：

```text
服务名 → 接口类型
```

---

## 37.3 `find`

```bash
ros2 service find my_topic/srv/GetXY
```

含义：

```text
接口类型 → 服务名
```

---

## 37.4 `call`

```bash
ros2 service call /get_xy my_topic/srv/GetXY "{}"
```

手动充当 Client。

---

# 38. 手动验证整个项目

建议分 3 个终端。

---

## 终端 1：Publisher

```bash
source /opt/ros/humble/setup.bash
source ~/chapter3/install/setup.bash

ros2 run topic pub
```

---

## 终端 2：Subscriber + Service Server

```bash
source /opt/ros/humble/setup.bash
source ~/chapter3/install/setup.bash

ros2 run topic sub
```

正常应该不断出现：

```text
Received: x=0.10, y=0.20
Received: x=0.20, y=0.40
...
```

---

## 终端 3：先手动测试 Service

```bash
source /opt/ros/humble/setup.bash
source ~/chapter3/install/setup.bash

ros2 service call /get_xy my_topic/srv/GetXY "{}"
```

可能得到：

```text
response:
my_topic.srv.GetXY_Response(
    x=4.0,
    y=8.0
)
```

如果这个成功，说明：

```text
Publisher ✅
Topic ✅
Subscriber ✅
Service Server ✅
```

---

# 39. 最后测试自己的 Client

```bash
ros2 run service client
```

正常：

```text
Request 已发送
x=6.20, y=12.40
```

说明完整链路：

```text
Publisher
    ↓
Topic
    ↓
Subscriber
    ↓
Service Server
    ↑
Service Client
```

全部成功。

---

# 40. `ros2 topic list` 卡住时

曾遇到：

```bash
ros2 topic list
```

长时间不返回。

可以先：

```bash
Ctrl+C
```

然后：

```bash
source /opt/ros/humble/setup.bash
source ~/chapter3/install/setup.bash
```

检查 daemon：

```bash
ros2 daemon stop
ros2 daemon start
```

注意拼写：

```text
daemon ✅
demon  ❌
```

然后：

```bash
ros2 topic list
```

如果仍有问题，再考虑检查：

- ROS_DOMAIN_ID
- DDS/RMW
- WSL 网络
- 残留 ROS2 进程
- daemon 状态

---

# 41. 本项目中实际遇到的典型错误汇总

| 错误写法/现象 | 正确写法/原因 |
|---|---|
| `my_topic.smg` | 应为依赖包 `my_topic` |
| `from my_topic.smg import Data` | `from my_topic.msg import Data` |
| `self.create.publisher()` | `self.create_publisher()` |
| `self.create.timer()` | `self.create_timer()` |
| `self.create_subscriber()` | `self.create_subscription()` |
| `self.create.client()` | `self.create_client()` |
| `GetXY.request()` | `GetXY.Request()` |
| `self.get_logger.info()` | `self.get_logger().info()` |
| `rclpy.spin()` | 通常写 `rclpy.spin(node)` |
| `destroy.node()` | `node.destroy_node()` |
| 两次 `rosidl_generate_interfaces()` | msg 和 srv 合并到一次调用 |
| `publish.main` | `publish:main` |
| setup.py 两项之间没逗号 | 每个 console_script 项目加 `,` |
| 在 `src` 中编译 | 推荐回到 `~/chapter3` |
| `ros2 demon start` | `ros2 daemon start` |
| `0.30000000004` | 正常浮点精度，不是 ROS2 错误 |
| `failed to send response` | Client 可能提前退出/被 Ctrl+C/异常终止 |

---

# 42. Node 中几个最重要的方法

建议直接记住：

```python
self.create_publisher()
self.create_subscription()
self.create_service()
self.create_client()
self.create_timer()
```

以及：

```python
rclpy.spin(node)
```

---

# 43. Publisher / Subscriber / Server / Client 对照

```text
Publisher
= 我主动往 Topic 发数据


Subscriber
= 我从 Topic 接收别人发的数据


Service Server
= 别人问我时，我处理 Request 并返回 Response


Service Client
= 我主动发送 Request，等待别人返回 Response
```

---

# 44. callback 的触发方式

## Publisher Timer callback

```text
Timer 到时间
↓
callback()
```

---

## Subscriber callback

```text
Topic 收到消息
↓
callback(msg)
```

---

## Service Server callback

```text
收到 Client Request
↓
callback(request, response)
```

---

## Future done callback

```text
Service Response 到达
↓
Future 完成
↓
callback(future)
```

---

# 45. Executor 与 `spin()` 的关系

可以简单理解：

```text
Executor
    ↓
负责调度各种 callback
```

例如：

```text
Timer callback
Subscriber callback
Service callback
Future callback
```

而：

```python
rclpy.spin(node)
```

让 Executor 持续工作。

因此：

```text
spin ≈ ROS2节点的事件循环
```

---

# 46. 默认单线程执行时的注意事项

如果 Subscriber 和 Service Server 在同一个 Node 中，并使用：

```python
rclpy.spin(node)
```

默认一般是单线程 Executor。

执行形式大致：

```text
执行一个 callback
↓
完成
↓
再执行下一个 callback
```

所以 callback 中不要写：

```python
while True:
    ...
```

也不要进行特别长的：

```python
time.sleep(...)
```

或者长时间阻塞操作。

否则其他 callback 无法及时执行。

---

# 47. 真正接传感器时怎么改

目前 Publisher 中是模拟数据：

```python
self.x += 0.1
y = 2.0 * self.x
```

真正使用传感器时，可以替换为：

```python
x, y = self.read_sensor()
```

然后：

```python
msg = Data()

msg.x = x
msg.y = y

self.pub.publish(msg)
```

如果传感器本身已有 ROS2 驱动，则可能根本不需要自己写 Publisher，只需要直接：

```python
self.create_subscription(
    传感器消息类型,
    传感器Topic,
    callback,
    10
)
```

---

# 48. 如何判断传感器是否已有 ROS2 Topic

首先：

```bash
ros2 node list
```

再：

```bash
ros2 topic list
```

然后：

```bash
ros2 topic info /某个topic
```

查看类型：

```bash
ros2 topic type /某个topic
```

查看数据：

```bash
ros2 topic echo /某个topic
```

例如：

```text
/imu/data
/scan
/camera/image_raw
/gps/fix
```

这些通常对应：

```text
IMU
激光雷达
相机
GPS
```

---

# 49. 本项目最重要的理解

整个项目真正的核心不是代码量，而是理解：

```text
Topic
负责持续更新数据

Service
负责按需查询数据
```

通过：

```python
self.x
self.y
```

共享最新状态：

```text
Topic callback
    ↓
更新 self.x/self.y

Service callback
    ↓
读取 self.x/self.y
    ↓
返回给 Client
```

---

# 50. 最终知识框架

```text
                 ROS2 Node
                    │
        ┌───────────┼───────────┐
        │           │           │
      Topic       Service      Timer
        │           │           │
   ┌────┴────┐  ┌───┴────┐      │
   │         │  │        │      │
Publisher Subscriber Server Client callback
   │         │     │      │
publish() callback  │   call_async()
                   │      │
             request/response
                          │
                        Future
                          │
                add_done_callback()
```

---

# 51. 推荐牢记的最小代码模板

## Publisher

```python
self.pub = self.create_publisher(Data, 'data', 10)
self.pub.publish(msg)
```

## Subscriber

```python
self.sub = self.create_subscription(
    Data,
    'data',
    self.callback,
    10
)
```

## Service Server

```python
self.srv = self.create_service(
    GetXY,
    'get_xy',
    self.service_callback
)
```

## Service Client

```python
self.client = self.create_client(
    GetXY,
    'get_xy'
)
```

## 异步 Request

```python
request = GetXY.Request()

future = self.client.call_async(request)

future.add_done_callback(
    self.callback
)
```

## Future Response

```python
response = future.result()

print(response.x)
print(response.y)
```

## ROS2事件循环

```python
rclpy.spin(node)
```

---

# 52. 一句话总结这个项目

> Publisher 使用 Topic 持续发送 `x、y`，Subscriber 实时接收并保存最新值；同一个 Subscriber 节点同时作为 Service Server，当 Client 发送空 `Request` 时，Server 将当前保存的最新 `x、y` 作为 `Response` 返回，而 Client 使用 `call_async()` + `Future` + `add_done_callback()` 异步处理返回结果。
