# ROS2 Topic、Service 与 Action 自定义接口机制总结

## 1. ROS2 三种通信方式总览

ROS2 节点之间主要通过三种通信机制：

| 通信方式 | 接口文件 | 核心作用 |
|---|---|---|
| Topic | `.msg` | 连续数据传输 |
| Service | `.srv` | 请求-响应通信 |
| Action | `.action` | 长时间任务执行 |

简单理解：

- **Topic：我有什么数据，我持续告诉你**
- **Service：你请求我做一件事，我返回结果**
- **Action：你给我一个目标，我执行任务，中间反馈过程，最后返回结果**

---

# 2. Topic（话题通信）

## 2.1 Topic通信模型

Topic采用发布-订阅模型：

```
Publisher                  Subscriber

   Node A                     Node B

      |                         |
      |                         |
      └────── Topic ───────────→

```

Publisher不断发布数据，Subscriber接收数据。

---

## 2.2 Topic为什么只有一个对象？

因为Topic只负责传递：

> 一条数据

它没有：

- 请求
- 返回
- 执行过程

因此只需要描述：

> 数据长什么样

例如：

`Data.msg`

```text
float64 temperature
float64 pressure
int32 id
```

ROS2生成：

```python
Data()
```

对象：

```
Data

├── temperature
├── pressure
└── id
```

使用：

```python
from my_interfaces.msg import Data

msg = Data()

msg.temperature = 25.5
msg.pressure = 101
msg.id = 1
```

---

## 2.3 多个Publisher发布同一个Topic

允许：

```
              /sensor

        ┌─────────────┐
        │             │
        ↓             ↓

   Sensor_A       Sensor_B

```

但是要求：

### 1. Topic名字相同

例如：

```python
"/sensor"
```

### 2. Msg类型相同

例如：

```python
SensorData.msg
```

---

如果：

Publisher A:

```
/sensor
SensorData.msg
```

Publisher B:

```
/sensor
Image.msg
```

会产生类型冲突。

因为Subscriber不知道收到的数据是什么类型。

---

# 3. Service（服务通信）

## 3.1 Service通信模型

Service采用：

> 请求-响应模型

```
Client                    Server

Request
  ─────────────────→


                    处理


Response
  ←─────────────────

```

一次请求对应一次响应。

---

# 3.2 Service为什么有两个对象？

因为Service包含两个方向：

1. Client发送什么？
2. Server返回什么？

例如：

`AddTwo.srv`

```text
int64 a
int64 b

---

int64 sum
```

其中：

`---`

表示分隔：

```
Request
------
Response
```

---

ROS2生成：

## Request对象

```python
AddTwo.Request()
```

包含：

```python
request.a
request.b
```

表示：

客户端发送的数据。


---

## Response对象

```python
AddTwo.Response()
```

包含：

```python
response.sum
```

表示：

服务器返回的数据。


结构：

```
AddTwo

├── Request
│      ├── a
│      └── b
│
└── Response
       └── sum

```

---

## Service Server

```python
def callback(self, request, response):

    response.sum = request.a + request.b

    return response
```

流程：

```
Request

   ↓

Server

   ↓

Response

```

---

# 4. Action（动作通信）

## 4.1 Action通信模型

Action用于：

> 长时间运行任务

例如：

- 机器人导航
- 机械臂抓取
- 自动充电


通信过程：

```
Client                    Server


Goal
 ─────────────────→


       ACTIVE


Feedback
 ←────────────────


Feedback
 ←────────────────


Result
 ←────────────────

```

---

# 4.2 Action为什么有三个对象？

因为一个任务包含三个阶段：

## ① Goal

表示：

> 我要做什么


例如：

`Move.action`

```text
float64 x
float64 y

---
bool success

---
float64 distance
```


Goal：

```python
Move.Goal()
```


属性：

```python
goal.x
goal.y
```


---

## ② Feedback

表示：

> 当前执行到哪里


对象：

```python
Move.Feedback()
```

例如：

```python
feedback.distance
```


服务器不断发送：

```
距离目标100m

距离目标50m

距离目标10m

```

---

## ③ Result

表示：

> 最终结果


对象：

```python
Move.Result()
```

例如：

```python
result.success
```

返回：

```
成功
失败
取消
```

---

结构：

```
Move.action


             Move

              |

 ┌────────────┼────────────┐

 Goal     Feedback      Result

```

---

# 5. Action中的goal_handle

## 5.1 为什么Action需要goal_handle？

因为一个Action Server可以同时处理多个任务。


例如：

```
             Action Server


        ┌─────────────┐
        │             │
        ↓             ↓

     Goal A        Goal B


 goal_handle_A  goal_handle_B

```

每一个Goal都需要独立管理：

- 当前状态
- Feedback
- Result
- Cancel

所以ROS2创建：

```
goal_handle
```

---

## 5.2 goal_handle是什么？

它可以理解为：

> 当前任务的管理对象

例如：

```python
async def execute_callback(self, goal_handle):
```

这里：

`goal_handle`

代表当前执行的任务。

---

## 5.3 为什么读取Goal数据是：

```python
goal_handle.request.target
```

而不是：

```python
goal_handle.target
```

结构：

```
goal_handle

     |
     |
     ↓

 request

     |
     |
     ↓

 Goal对象

     |
     |
     ↓

 target属性

```

所以：

```python
goal_handle.request.target
```

表示：

> 当前任务中，客户端发送的target参数

---

# 6. Action中的Feedback为什么由goal_handle发送？

不是：

```python
self.server.publish_feedback()
```

而是：

```python
goal_handle.publish_feedback(feedback)
```

原因：

Feedback属于某一个具体任务。

例如：

```
Action Server


Goal A
 |
 goal_handle_A
 |
 feedback_A


Goal B
 |
 goal_handle_B
 |
 feedback_B

```

服务器需要知道：

这个反馈发给哪个Client。

所以由：

```python
goal_handle
```

负责发送。

---

# 7. 三种接口对象总结


|通信|接口文件|生成对象|作用|
|-|-|-|-|
|Topic|msg|Data()|传递数据|
|Service|srv|Request + Response|请求和返回|
|Action|action|Goal + Feedback + Result|任务执行|

---

# 8. 三种通信完整流程


## Topic

```
Publisher

    ↓

Msg

    ↓

Subscriber

```


---

## Service

```
Client

    ↓

Request

    ↓

Server

    ↓

Response

```


---

## Action

```
Client

    ↓

Goal

    ↓

goal_handle

    ↓

Execute

    ↓

Feedback

    ↓

Result

```


---

# 9. 工程中的使用场景


|功能|推荐通信|
|-|-|
|IMU数据发布|Topic|
|摄像头图像|Topic|
|激光雷达数据|Topic|
|读取参数|Service|
|修改配置|Service|
|启动设备|Service|
|机器人导航|Action|
|机械臂运动|Action|
|自动充电|Action|

---

# 10. 最终记忆方法

```
Topic:

数据流

msg

我有数据，我持续告诉你


Service:

请求-响应

Request
Response

你让我做事，我完成后告诉你


Action:

任务执行

Goal
Feedback
Result

你给目标，
我执行，
告诉过程，
最后返回结果

```