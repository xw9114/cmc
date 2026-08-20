# ROS 2 Topic 与 Service 重要函数学习笔记（Python / rclpy）

> 重点整理 Topic、Service、Future、Callback、Spin、Executor 中最常用、最重要的函数。

---

# 一、ROS 2 节点基础函数

## 1. `rclpy.init()`

```python
rclpy.init(args=args)
```

作用：初始化 ROS 2 Python 客户端环境。通常在 `main()` 开头调用。

```python
def main(args=None):
    rclpy.init(args=args)
```

---

## 2. `super().__init__('node_name')`

```python
super().__init__('my_node')
```

作用：初始化一个 ROS 2 Node。

其中：

```text
my_node
```

是 ROS Graph 中的节点名。

---

## 3. `destroy_node()`

```python
node.destroy_node()
```

作用：销毁当前节点并释放资源。

---

## 4. `rclpy.shutdown()`

```python
rclpy.shutdown()
```

作用：关闭 ROS 2 Python 客户端环境。

常见整体结构：

```python
def main(args=None):
    rclpy.init(args=args)

    node = MyNode()

    rclpy.spin(node)

    node.destroy_node()
    rclpy.shutdown()
```

---

# 二、Topic：Publisher 重要函数

## 1. `create_publisher()`

```python
self.publisher = self.create_publisher(
    String,
    'hello',
    10
)
```

作用：创建 Publisher。

三个主要参数：

```text
String
→ 消息类型

hello
→ Topic 名

10
→ QoS depth
```

---

## 2. `publish()`

```python
self.publisher.publish(msg)
```

作用：真正发布一条 Topic 消息。

例如：

```python
msg = String()
msg.data = 'Hello ROS 2'

self.publisher.publish(msg)
```

关系：

```text
create_publisher()
        ↓
创建 Publisher

publish(msg)
        ↓
真正发送消息
```

---

# 三、Topic：Subscriber 重要函数

## 1. `create_subscription()`

```python
self.subscription = self.create_subscription(
    String,
    'hello',
    self.listener_callback,
    10
)
```

作用：创建 Subscriber，并注册收到消息后要执行的 callback。

参数：

```text
String
→ Message 类型

hello
→ Topic 名

self.listener_callback
→ 收到消息后执行的函数

10
→ QoS depth
```

---

## 2. Subscriber Callback

```python
def listener_callback(self, msg):
    self.get_logger().info(
        f'I heard: {msg.data}'
    )
```

这里注册的是：

```python
self.listener_callback
```

而不是：

```python
self.listener_callback()
```

区别：

```text
self.listener_callback
→ 把函数本身交给 ROS 2，等待以后调用

self.listener_callback()
→ 现在立刻执行函数
```

运行逻辑：

```text
create_subscription()
        ↓
注册 callback
        ↓
spin()
        ↓
Executor 等待消息
        ↓
消息到达
        ↓
自动执行 callback(msg)
```

---

# 四、Service Server 重要函数

## 1. `create_service()`

```python
self.srv = self.create_service(
    AddTwoNumbers,
    'add_two_ints',
    self.service_callback
)
```

作用：创建 Service Server，并注册处理 Request 的 callback。

参数：

```text
AddTwoNumbers
→ Service 接口类型

add_two_ints
→ Service 名

self.service_callback
→ Request 到来后执行的 callback
```

---

## 2. Service Callback

例如 `.srv`：

```text
int64 a
int64 b
---
int64 sum
```

Server：

```python
def service_callback(self, request, response):

    response.sum = request.a + request.b

    return response
```

其中：

```python
request.a
request.b
```

是 Client 发来的 Request 数据。

```python
response.sum
```

是 Server 填写的 Response 数据。

执行逻辑：

```text
Client 发送 Request
        ↓
Server 收到 Request
        ↓
Executor 调度 service_callback
        ↓
读取 request
        ↓
填写 response
        ↓
return response
        ↓
Response 返回 Client
```

---

# 五、Service Client 重要函数

## 1. `create_client()`

```python
self.client = self.create_client(
    AddTwoNumbers,
    'add_two_ints'
)
```

作用：创建 Service Client。

参数：

```text
AddTwoNumbers
→ Service 接口类型

add_two_ints
→ Client 想连接的 Service 名
```

Server 与 Client 的：

```text
Service 类型
Service 名
```

必须匹配。

---

## 2. `wait_for_service()`

```python
self.client.wait_for_service(timeout_sec=1.0)
```

作用：等待当前 Client 对应的 Service Server 可用。

返回值：

```text
True
→ Service 已经可用

False
→ 规定时间内没发现 Service
```

常见写法：

```python
while not self.client.wait_for_service(timeout_sec=1.0):
    self.get_logger().info('Waiting for service...')
```

逻辑：

```text
Service 不存在
    ↓
wait_for_service() = False
    ↓
not False = True
    ↓
继续 while

Service 出现
    ↓
wait_for_service() = True
    ↓
not True = False
    ↓
退出 while
```

所以可以理解成：

> 只要 Service 还不可用，就继续等待。

---

# 六、Service Request 相关函数

## 1. `Request()`

```python
request = AddTwoNumbers.Request()
```

作用：创建一个 Service Request 对象。

如果 `.srv`：

```text
int64 a
int64 b
---
int64 sum
```

那么：

```python
request.a = 10
request.b = 20
```

---

## 2. `call_async()`

```python
future = self.client.call_async(request)
```

作用：异步发送 Request。

它不会马上返回 Response，而是先返回：

```text
Future
```

因此：

```text
future ≠ response
```

Future 可以理解成：

> 一个“未来会装入结果”的对象。

执行过程：

```text
call_async(request)
        ↓
Request 发出去
        ↓
立即得到 Future
        ↓
Server 处理
        ↓
Response 回来
        ↓
Future 完成
```

---

# 七、Future 最重要的函数

## 1. `future.done()`

```python
future.done()
```

作用：检查 Future 是否已经完成。

```text
False
→ 结果还没回来

True
→ Future 已完成
```

---

## 2. `future.result()`

```python
response = future.result()
```

作用：从 Future 中取出最终结果。

对于 Service：

```python
response = future.result()
print(response.sum)
```

---

## 3. `future.exception()`

```python
error = future.exception()
```

作用：检查 Future 执行过程中是否发生异常。

正常完成一般：

```text
None
```

发生异常时会返回对应异常对象。

---

## 4. `future.add_done_callback()`

```python
future.add_done_callback(
    self.response_callback
)
```

作用：

> 注册一个“Future 完成以后自动执行”的 callback。

例如：

```python
def send_request(self, a, b):

    request = AddTwoNumbers.Request()
    request.a = a
    request.b = b

    future = self.client.call_async(request)

    future.add_done_callback(
        self.response_callback
    )
```

对应：

```python
def response_callback(self, future):

    response = future.result()

    self.get_logger().info(
        f'Result: {response.sum}'
    )
```

过程：

```text
call_async()
    ↓
得到 Future
    ↓
add_done_callback()
    ↓
登记“完成以后调用谁”
    ↓
程序继续运行
    ↓
Response 回来
    ↓
Future 完成
    ↓
response_callback(future)
    ↓
future.result()
```

这种方式更加符合 ROS 2 的事件驱动思想。

---

# 八、Spin 相关的重要函数

## 1. `rclpy.spin()`

```python
rclpy.spin(node)
```

作用：

> 让 Executor 持续等待并处理 ROS 2 事件。

可以粗略理解成：

```text
while ROS 还在运行:

    等待事件

    Topic 消息到达
        ↓
    执行 Topic callback

    Service Request 到达
        ↓
    执行 Service callback

    Timer 到时间
        ↓
    执行 Timer callback
```

所以 `spin()` 一直不退出通常不是“卡住”，而是节点正在持续工作。

---

## 2. `rclpy.spin_once()`

```python
rclpy.spin_once(node)
```

作用：

> 处理一次 ROS 2 事件，然后返回。

区别：

```text
spin()
→ 一直运行

spin_once()
→ 处理一次后返回
```

例如：

```python
while rclpy.ok():

    rclpy.spin_once(
        node,
        timeout_sec=0.1
    )

    # 可以继续执行自己的代码
```

---

## 3. `rclpy.spin_until_future_complete()`

```python
rclpy.spin_until_future_complete(
    node,
    future
)
```

作用：

> 持续处理 ROS 2 事件，直到指定 Future 完成。

例如：

```python
future = node.send_request(10, 20)

rclpy.spin_until_future_complete(
    node,
    future
)

response = future.result()
```

执行过程：

```text
发送 Request
    ↓
得到 Future
    ↓
spin_until_future_complete()
    ↓
持续处理 ROS 事件
    ↓
Future 完成
    ↓
函数返回
    ↓
future.result()
```

---

# 九、两种 Future 使用方式

## 方式 1：等待 Future

```python
future = self.client.call_async(request)

rclpy.spin_until_future_complete(
    node,
    future
)

response = future.result()
```

思想：

```text
发请求
↓
等待完成
↓
拿结果
```

适合简单的一次性 Client。

---

## 方式 2：使用 `add_done_callback()`

```python
future = self.client.call_async(request)

future.add_done_callback(
    self.response_callback
)
```

然后：

```python
def response_callback(self, future):

    response = future.result()

    print(response.sum)
```

主程序保持：

```python
rclpy.spin(node)
```

思想：

```text
发请求
↓
登记“结果回来以后干什么”
↓
继续处理其他 ROS 事件
↓
Future 完成
↓
自动执行 response_callback
```

这种方式更适合长期运行的 ROS 2 节点。

---

# 十、Callback 的共同规律

## Topic

```python
self.create_subscription(
    String,
    'topic',
    self.topic_callback,
    10
)
```

表示：

```text
Topic 消息来了
↓
执行 topic_callback
```

---

## Service

```python
self.create_service(
    AddTwoNumbers,
    'add_two_ints',
    self.service_callback
)
```

表示：

```text
Request 来了
↓
执行 service_callback
```

---

## Future

```python
future.add_done_callback(
    self.response_callback
)
```

表示：

```text
Future 完成
↓
执行 response_callback
```

共同规律：

```text
先注册 callback
      ↓
不立即调用
      ↓
等待对应事件
      ↓
事件发生
      ↓
执行 callback
```

---

# 十一、Executor 与这些函数的关系

Executor 可以理解成：

> ROS 2 节点内部负责调度 callback 的任务调度器。

例如：

```text
                 Executor
                    │
      ┌─────────────┼─────────────┐
      ▼             ▼             ▼
 Topic消息      Service请求      Timer到时
      │             │             │
      ▼             ▼             ▼
 topic_cb       service_cb      timer_cb
```

而：

```python
rclpy.spin(node)
```

就是让 Executor 持续工作。

可以简单记：

```text
Node
→ 有哪些通信对象和 callback

Executor
→ 决定什么时候执行哪个 callback

spin()
→ 让 Executor 持续运行
```

---

# 十二、Topic 与 Service 常用命令

## Topic

查看 Topic：

```bash
ros2 topic list
```

查看 Topic 类型：

```bash
ros2 topic type /topic_name
```

查看 Topic 数据：

```bash
ros2 topic echo /topic_name
```

命令行发布消息：

```bash
ros2 topic pub ...
```

---

## Service

查看 Service：

```bash
ros2 service list
```

查看 Service 类型：

```bash
ros2 service type /add_two_ints
```

查看接口结构：

```bash
ros2 interface show my_interfaces/srv/AddTwoNumbers
```

命令行调用：

```bash
ros2 service call /add_two_ints my_interfaces/srv/AddTwoNumbers "{a: 10, b: 20}"
```

---

# 十三、重要函数速查表

| 函数 | 所属 | 作用 |
|---|---|---|
| `rclpy.init()` | ROS 2 | 初始化 ROS 2 Python 环境 |
| `super().__init__()` | Node | 初始化 Node |
| `create_publisher()` | Topic | 创建 Publisher |
| `publish()` | Topic | 发布消息 |
| `create_subscription()` | Topic | 创建 Subscriber 并注册 callback |
| `create_service()` | Service | 创建 Service Server |
| `create_client()` | Service | 创建 Service Client |
| `wait_for_service()` | Service | 等待对应 Service Server 可用 |
| `Request()` | Service | 创建 Request |
| `call_async()` | Service | 异步发送 Request，返回 Future |
| `future.done()` | Future | 判断 Future 是否完成 |
| `future.result()` | Future | 获取 Future 的结果 |
| `future.exception()` | Future | 获取 Future 异常 |
| `future.add_done_callback()` | Future | Future 完成后调用指定 callback |
| `rclpy.spin()` | Executor | 持续处理 ROS 2 事件 |
| `rclpy.spin_once()` | Executor | 处理一次 ROS 2 事件 |
| `spin_until_future_complete()` | Future / Executor | spin 到 Future 完成 |
| `destroy_node()` | Node | 销毁节点 |
| `rclpy.shutdown()` | ROS 2 | 关闭 ROS 2 环境 |

---

# 十四、整体知识框架

```text
                    ROS 2 Node
                         │
        ┌────────────────┼────────────────┐
        │                │                │
      Topic           Service          Future
        │                │                │
        ▼                ▼                ▼
create_publisher   create_service      call_async()
create_subscription create_client          │
        │                │                 ▼
        │                │               Future
        │                │                 │
        ▼                ▼                 ▼
topic callback    service callback   add_done_callback()
        │                │                 │
        └────────────────┼─────────────────┘
                         │
                         ▼
                      Executor
                         │
                         ▼
                       spin()
                         │
                         ▼
                事件发生 → 执行 callback
```

---

# 十五、核心总结

Topic 最核心：

```text
create_publisher()
publish()
create_subscription()
callback
```

Service 最核心：

```text
create_service()
create_client()
wait_for_service()
Request()
call_async()
Response
```

Future 最核心：

```text
done()
result()
exception()
add_done_callback()
```

ROS 2 事件调度最核心：

```text
spin()
Executor
callback
```

把它们连起来：

```text
创建通信对象
    ↓
注册 callback
    ↓
spin 启动 Executor
    ↓
事件发生
    ↓
Executor 调度 callback
    ↓
完成 Topic / Service 通信
```
