# ROS2 Action 机制与代码细节总结

## 1. Action 是什么

ROS2 中常见的三种通信方式：

| 通信方式 | 特点 | 典型场景 |
|---|---|---|
| Topic | 持续发布/订阅，不要求返回 | 传感器数据、速度指令 |
| Service | 一次 Request → 一次 Response | 查询、配置、短任务 |
| Action | Goal → Feedback → Result，可取消 | 导航、机械臂运动、长时间任务 |

Action 最适合：

- 任务需要执行一段时间；
- Client 希望知道执行进度；
- Client 可能需要中途取消；
- Server 需要区分任务成功、失败或取消。

可以把 Action 简化理解为：

```text
Client
  │
  │ Goal
  ▼
Server
  │
  ├── Feedback
  ├── Feedback
  ├── Feedback
  │
  ▼
 Result
```

---

# 2. `.action` 文件的三个部分

例如：

```text
int32 target
---
int32 sum
---
int32 current
```

分别表示：

```text
Goal
---
Result
---
Feedback
```

编译后 ROS2 会生成：

```python
Count.Goal()
Count.Result()
Count.Feedback()
```

因此可以记：

| `.action` 部分 | Python 对象 | 用途 |
|---|---|---|
| Goal | `Count.Goal()` | Client 告诉 Server 要做什么 |
| Result | `Count.Result()` | Server 返回最终结果 |
| Feedback | `Count.Feedback()` | Server 返回执行过程中的进度 |

---

# 3. Action 不只是 Goal、Feedback、Result

Action 和普通消息不同。

除了：

```text
Goal
Feedback
Result
```

ROS2 还需要管理：

```text
Goal ID
Goal 状态
Goal 是否被接受
Cancel 请求
Feedback 属于哪个 Goal
Result 属于哪个 Goal
任务最终状态
```

所以 Action 内部会自动生成一些通信管理接口。

可以理解成两层：

```text
第一层：用户定义的数据
────────────────────
Count.Goal
Count.Feedback
Count.Result


第二层：ROS2 自动管理的通信层
────────────────────
Goal ID
SendGoal
GetResult
FeedbackMessage
Status
Cancel
GoalHandle
```

平时主要操作第一层，第二层多数由 ROS2 自动完成。

---

# 4. Action Server 的核心结构

一个典型 Server：

```python
from rclpy.action import ActionServer, GoalResponse

self.action_server = ActionServer(
    self,
    Count,
    'count',
    execute_callback=self.execute_callback,
    goal_callback=self.goal_callback
)
```

这里最重要的是两个回调：

```text
goal_callback()
execute_callback()
```

它们职责完全不同。

---

# 5. `goal_callback()`：决定接不接 Goal

例如：

```python
def goal_callback(self, goal_request):

    if goal_request.target <= 0:
        return GoalResponse.REJECT

    return GoalResponse.ACCEPT
```

这里：

```python
goal_request
```

本质上就是这次 Client 发送来的 Goal 数据。

可以理解为：

```python
goal_request ≈ Count.Goal()
```

例如 Client：

```python
goal_msg = Count.Goal()
goal_msg.target = 5
```

Server 的：

```python
goal_request.target
```

就是：

```text
5
```

---

## 5.1 ACCEPT 与 REJECT

Server 可以返回：

```python
GoalResponse.ACCEPT
```

表示接受任务。

或者：

```python
GoalResponse.REJECT
```

表示一开始就不接这个任务。

例如：

```python
def goal_callback(self, goal_request):

    if 1 <= goal_request.target <= 100:
        return GoalResponse.ACCEPT

    return GoalResponse.REJECT
```

---

# 6. ACCEPT 后，为什么变成了 `goal_handle`

严格来说：

```text
goal_request 并没有“变成” goal_handle
```

真正过程是：

```text
Client发送 Goal
      ↓
Server收到 goal_request
      ↓
goal_callback(goal_request)
      ↓
GoalResponse.ACCEPT
      ↓
ROS2 ActionServer 内部
      ↓
为这个 Goal 建立任务管理信息
      ↓
创建 ServerGoalHandle
      ↓
把原来的 Goal 关联到 goal_handle.request
      ↓
execute_callback(goal_handle)
```

所以：

```python
def execute_callback(self, goal_handle):
```

这里的：

```python
goal_handle
```

是一个：

```text
ServerGoalHandle
```

而：

```python
goal_handle.request
```

就是原来的 Goal 请求数据。

例如：

```python
goal = goal_handle.request

print(goal.target)
```

---

# 7. `goal_handle` 到底是什么

`goal_handle` 不是 Goal 对象。

它是：

> 当前这一次 Goal 任务的管理句柄。

Server 端的 `goal_handle` 可以理解成：

```text
ServerGoalHandle
│
├── request
│     └── Goal数据
│
├── publish_feedback()
│
├── succeed()
│
├── abort()
│
├── canceled()
│
└── is_cancel_requested
```

所以：

```python
goal_handle.request
```

负责：

```text
读取 Client 要求做什么
```

而：

```python
goal_handle.publish_feedback()
goal_handle.succeed()
goal_handle.abort()
goal_handle.canceled()
```

负责管理：

```text
这一次任务的执行过程和状态
```

---

# 8. Server 端的 Goal、Feedback、Result 对象

典型写法：

```python
def execute_callback(self, goal_handle):

    goal = goal_handle.request

    feedback = Count.Feedback()

    result = Count.Result()
```

因此：

```text
Goal对象
    ↓
goal_handle.request

Feedback对象
    ↓
Count.Feedback()

Result对象
    ↓
Count.Result()
```

---

# 9. Feedback 怎么使用

Server 创建：

```python
feedback = Count.Feedback()
```

设置：

```python
feedback.current = i
```

然后：

```python
goal_handle.publish_feedback(feedback)
```

例如：

```python
for i in range(1, 6):

    feedback.current = i

    goal_handle.publish_feedback(feedback)
```

Client 就会依次收到：

```text
1
2
3
4
5
```

Feedback 的含义：

> 任务还没有结束，但 Server 告诉 Client 当前执行到哪里了。

---

# 10. `succeed()` 是干什么的

例如：

```python
goal_handle.succeed()
```

它不是返回结果。

它的作用是：

> 把当前 Goal 的最终状态设置为“成功”。

真正的数据仍然通过：

```python
return result
```

返回。

因此：

```python
goal_handle.succeed()
return result
```

分别表示：

```text
succeed()
    ↓
告诉 ROS2：
这个 Goal 成功完成


return result
    ↓
告诉 Client：
最终结果的数据是什么
```

例如：

```python
result.sum = 15

goal_handle.succeed()

return result
```

其中：

```text
任务状态 = SUCCEEDED
结果数据 = sum = 15
```

---

# 11. `abort()` 和 `canceled()`

## 11.1 `abort()`

表示：

> Server 已经接受任务，但执行过程中失败了。

例如：

```python
if motor_error:

    goal_handle.abort()

    result.sum = total

    return result
```

记忆：

```text
REJECT
=
我一开始就不接


ABORT
=
我接了，但执行过程中失败
```

---

## 11.2 `canceled()`

Client 可以请求取消任务。

Server 执行过程中检查：

```python
if goal_handle.is_cancel_requested:
```

如果确实有取消请求：

```python
goal_handle.canceled()
```

然后：

```python
return result
```

所以：

```text
is_cancel_requested
=
Client 有没有请求取消


canceled()
=
Server 正式把这个 Goal 标记为已取消
```

---

# 12. ACCEPT 后还能不能变成 REJECT

不能。

Goal 一旦：

```python
GoalResponse.ACCEPT
```

就已经接受。

执行过程中如果发生问题，应该：

```python
goal_handle.abort()
```

而不是重新：

```python
GoalResponse.REJECT
```

因此：

```text
收到 Goal
   ↓
goal_callback()
   ↓
ACCEPT / REJECT
   ↓
决定结束
```

如果已经 ACCEPT：

```text
执行
 ↓
成功
→ succeed()

失败
→ abort()

取消
→ canceled()
```

---

# 13. Client 端为什么有 GoalHandle

Client：

```python
send_goal_future = self.action_client.send_goal_async(goal_msg)
```

这是：

> 异步发送 Goal。

它返回一个：

```text
Future
```

这个 Future 等待的是：

> Server 到底接不接这个 Goal？

等完成后：

```python
goal_handle = future.result()
```

这里得到的是：

```text
ClientGoalHandle
```

不是：

```text
Count.Result
```

---

# 14. ClientGoalHandle 和 ServerGoalHandle

Client：

```python
goal_handle = future.result()
```

得到：

```text
ClientGoalHandle
```

Server：

```python
def execute_callback(self, goal_handle):
```

得到：

```text
ServerGoalHandle
```

这两个不是同一个 Python 对象。

但是：

> 它们代表的是同一次 Goal 任务。

可以理解为：

```text
ClientGoalHandle
      │
      │ 同一个 Goal
      │
ServerGoalHandle
```

---

## 14.1 ClientGoalHandle 的作用

常见功能：

```python
goal_handle.accepted
goal_handle.get_result_async()
goal_handle.cancel_goal_async()
```

也就是 Client 用它：

```text
看 Server 接没接
等待最终 Result
取消任务
```

---

## 14.2 ServerGoalHandle 的作用

常见功能：

```python
goal_handle.request
goal_handle.publish_feedback()
goal_handle.succeed()
goal_handle.abort()
goal_handle.canceled()
```

也就是 Server 用它：

```text
读取 Goal
发送 Feedback
设置任务状态
```

---

# 15. Future 是什么

Future 可以简单理解成：

> 一个异步操作未来结果的占位对象。

例如：

```python
future = client.call_async(request)
```

程序不会一直卡在这里等。

而是先得到：

```text
Future
```

等结果真正回来以后：

```python
future.result()
```

才能拿到实际结果。

---

# 16. `future.result()` 是固定的吗

`.result()` 是 Future 对象的方法。

所以：

```python
future.result()
```

这个使用方式基本固定。

但是：

```python
future
```

只是变量名，可以改成：

```python
send_goal_future
```

或者：

```python
result_future
```

都可以。

例如：

```python
send_goal_future.result()
```

本质上仍然是：

```text
Future.result()
```

---

# 17. Action 为什么有两个 Future

这是 Action 最容易混淆的地方。

因为 Action 有两个不同的异步阶段。

---

## 17.1 第一个 Future：等 Server 接不接 Goal

```python
send_goal_future = self.action_client.send_goal_async(goal_msg)
```

这个 Future 等的是：

```text
Server 接不接？
```

完成后：

```python
goal_handle = send_goal_future.result()
```

得到：

```text
ClientGoalHandle
```

---

## 17.2 第二个 Future：等任务真正执行完成

Goal 被接受后：

```python
result_future = goal_handle.get_result_async()
```

又返回一个 Future。

这个 Future 等的是：

```text
任务什么时候真正完成？
```

完成以后：

```python
result_response = result_future.result()
```

再进一步：

```python
result = result_response.result
```

拿到真正的：

```text
Count.Result
```

---

# 18. 为什么 Action 不能只用一个 Future

因为：

```text
“Server 接受任务”
```

和：

```text
“Server 把任务做完”
```

不是一回事。

例如：

```text
Client：移动到目标点
   ↓
Server：好的，我接受
   ↓
机器人开始移动
   ↓
20%
40%
60%
80%
   ↓
到达
   ↓
Result
```

所以 Action 天然分成：

```text
Future 1
=
等接单


Future 2
=
等做完
```

可以记成：

```text
Goal
 ↓
Future 1
 ↓
GoalHandle
 ↓
Future 2
 ↓
Result
```

---

# 19. `result_future` 是什么

```python
result_future = goal_handle.get_result_async()
```

这里：

```text
result_future
```

是一个：

```text
Future 对象
```

表示：

> 现在最终结果还没回来，我正在异步等待。

---

# 20. `result_response` 是什么

当：

```python
result_future
```

完成后：

```python
result_response = future.result()
```

这里得到的是 ROS2 Action 自动生成的：

```text
GetResult.Response
```

它不是最终的 `Count.Result` 本体。

大致结构：

```text
GetResult.Response
│
├── status
│
└── result
      ↓
 Count.Result
```

所以真正的 Result：

```python
result = result_response.result
```

例如：

```python
print(result.sum)
```

---

# 21. 为什么 Result 外面还有一层包装

因为 ROS2 不仅需要告诉 Client：

```text
结果数据是多少
```

还需要告诉 Client：

```text
这个 Goal 最终是什么状态
```

例如：

```text
SUCCEEDED
ABORTED
CANCELED
```

所以 Result 通信中会有：

```text
GetResult.Response
├── status
└── result
```

---

# 22. Feedback 其实也有包装层

Client：

```python
def feedback_callback(self, feedback_msg):

    feedback = feedback_msg.feedback
```

这里：

```python
feedback_msg
```

并不是：

```text
Count.Feedback
```

它实际上是 ROS2 自动生成的：

```text
FeedbackMessage
```

大致结构：

```text
FeedbackMessage
│
├── goal_id
│
└── feedback
      ↓
 Count.Feedback
```

所以：

```python
feedback = feedback_msg.feedback
```

才是真正取出：

```text
Count.Feedback
```

---

# 23. 为什么 Feedback 需要 Goal ID

因为一个 Action Server 可能同时存在多个 Goal。

例如：

```text
Goal A
target = 5

Goal B
target = 10
```

假如 Server 发送：

```text
current = 3
```

ROS2 必须知道：

```text
这是 Goal A 的 Feedback
还是 Goal B 的 Feedback？
```

所以外层需要：

```text
goal_id
```

---

# 24. Feedback 和 Result 的包装层对比

Feedback：

```text
FeedbackMessage
│
├── goal_id
└── feedback
      ↓
Count.Feedback
```

Result：

```text
GetResult.Response
│
├── status
└── result
      ↓
Count.Result
```

因此 Client 常见代码：

```python
feedback = feedback_msg.feedback
```

以及：

```python
result = result_response.result
```

---

# 25. 这些包装层是谁生成的

不是你手动写的。

过程：

```text
编写 Count.action
      ↓
colcon build
      ↓
rosidl 读取 .action
      ↓
自动生成 Action 所需接口类型
      ↓
程序运行
      ↓
ROS2 自动创建对象并管理通信
```

所以准确地说：

```text
包装“类型”
=
编译 .action 时自动生成


包装“对象”
=
程序运行通信时由 ROS2 自动创建和管理
```

---

# 26. Action Client 完整主流程

典型代码：

```python
def send_goal(self, target):

    self.action_client.wait_for_server()

    goal_msg = Count.Goal()
    goal_msg.target = target

    send_goal_future = self.action_client.send_goal_async(
        goal_msg,
        feedback_callback=self.feedback_callback
    )

    send_goal_future.add_done_callback(
        self.goal_response_callback
    )
```

流程：

```text
创建 Goal
   ↓
send_goal_async()
   ↓
send_goal_future
   ↓
等待 Server 接不接
```

---

# 27. Goal 响应回调

```python
def goal_response_callback(self, future):

    goal_handle = future.result()

    if not goal_handle.accepted:
        return

    result_future = goal_handle.get_result_async()

    result_future.add_done_callback(
        self.result_callback
    )
```

这里：

```text
future.result()
      ↓
ClientGoalHandle
```

然后：

```text
ClientGoalHandle
      ↓
get_result_async()
      ↓
result_future
```

---

# 28. Feedback 回调

```python
def feedback_callback(self, feedback_msg):

    feedback = feedback_msg.feedback

    print(feedback.current)
```

关系：

```text
feedback_msg
   ↓
.feedback
   ↓
Count.Feedback
```

---

# 29. Result 回调

```python
def result_callback(self, future):

    result_response = future.result()

    result = result_response.result

    print(result.sum)
```

关系：

```text
future
 ↓
future.result()
 ↓
GetResult.Response
 ↓
.result
 ↓
Count.Result
```

---

# 30. Action Server 完整核心代码

```python
import time

import rclpy
from rclpy.node import Node
from rclpy.action import ActionServer, GoalResponse

from my_interfaces.action import Count


class CountActionServer(Node):

    def __init__(self):

        super().__init__('count_action_server')

        self.action_server = ActionServer(
            self,
            Count,
            'count',
            execute_callback=self.execute_callback,
            goal_callback=self.goal_callback
        )


    def goal_callback(self, goal_request):

        if goal_request.target <= 0:

            return GoalResponse.REJECT

        return GoalResponse.ACCEPT


    def execute_callback(self, goal_handle):

        goal = goal_handle.request

        feedback = Count.Feedback()

        result = Count.Result()

        total = 0

        for i in range(1, goal.target + 1):

            if goal_handle.is_cancel_requested:

                goal_handle.canceled()

                result.sum = total

                return result

            total += i

            feedback.current = i

            goal_handle.publish_feedback(feedback)

            time.sleep(1)

        goal_handle.succeed()

        result.sum = total

        return result


def main(args=None):

    rclpy.init(args=args)

    node = CountActionServer()

    rclpy.spin(node)

    node.destroy_node()

    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

---

# 31. Action Client 完整核心代码

```python
import rclpy
from rclpy.node import Node
from rclpy.action import ActionClient

from my_interfaces.action import Count


class CountActionClient(Node):

    def __init__(self):

        super().__init__('count_action_client')

        self.action_client = ActionClient(
            self,
            Count,
            'count'
        )


    def send_goal(self, target):

        self.action_client.wait_for_server()

        goal_msg = Count.Goal()

        goal_msg.target = target

        send_goal_future = self.action_client.send_goal_async(
            goal_msg,
            feedback_callback=self.feedback_callback
        )

        send_goal_future.add_done_callback(
            self.goal_response_callback
        )


    def goal_response_callback(self, future):

        goal_handle = future.result()

        if not goal_handle.accepted:

            self.get_logger().info('Goal rejected')

            return

        self.get_logger().info('Goal accepted')

        result_future = goal_handle.get_result_async()

        result_future.add_done_callback(
            self.result_callback
        )


    def feedback_callback(self, feedback_msg):

        feedback = feedback_msg.feedback

        self.get_logger().info(
            f'Feedback: {feedback.current}'
        )


    def result_callback(self, future):

        result_response = future.result()

        result = result_response.result

        self.get_logger().info(
            f'Result: {result.sum}'
        )

        rclpy.shutdown()


def main(args=None):

    rclpy.init(args=args)

    node = CountActionClient()

    node.send_goal(5)

    rclpy.spin(node)


if __name__ == '__main__':
    main()
```

---

# 32. Action 最核心的对象关系

## Client

```text
Count.Goal
    ↓
send_goal_async()
    ↓
send_goal_future
    ↓
future.result()
    ↓
ClientGoalHandle
    ↓
accepted
    ↓
get_result_async()
    ↓
result_future
    ↓
future.result()
    ↓
GetResult.Response
    ↓
.result
    ↓
Count.Result
```

Feedback：

```text
Server
 ↓
FeedbackMessage
 ↓
feedback_msg.feedback
 ↓
Count.Feedback
```

---

## Server

```text
Goal请求
   ↓
goal_callback(goal_request)
   ↓
ACCEPT
   ↓
ROS2创建 ServerGoalHandle
   ↓
execute_callback(goal_handle)
   ↓
goal_handle.request
   ↓
Count.Goal
```

执行过程中：

```text
Count.Feedback
      ↓
goal_handle.publish_feedback()
```

结束：

```text
成功
→ goal_handle.succeed()

失败
→ goal_handle.abort()

取消
→ goal_handle.canceled()
```

最终：

```text
Count.Result
      ↓
return result
```

---

# 33. Action 与 Service 的本质区别

Service：

```text
Request
  ↓
Future
  ↓
Response
```

Action：

```text
Goal
 ↓
Future 1
 ↓
是否接受
 ↓
GoalHandle
 ↓
执行
 ├── Feedback
 ├── Feedback
 └── Feedback
 ↓
Future 2
 ↓
Result
```

Action 多出来的核心能力：

```text
任务接受/拒绝
任务生命周期
Feedback
Cancel
Success / Abort / Canceled
```

---

# 34. 最容易混淆的几个点

## 34.1 `goal_handle` 不是 Goal

错误理解：

```text
goal_handle = Goal
```

正确：

```text
goal_handle
=
管理当前 Goal 任务的句柄
```

真正 Goal：

Server：

```python
goal_handle.request
```

Client：

```python
goal_msg = Count.Goal()
```

---

## 34.2 `future.result()` 不一定得到 Result

`future.result()` 得到什么，取决于这个 Future 来自哪个异步操作。

例如：

```python
client.call_async(request)
```

对应：

```text
future.result()
→ Service Response
```

而：

```python
send_goal_async(goal)
```

对应：

```text
future.result()
→ ClientGoalHandle
```

而：

```python
get_result_async()
```

对应：

```text
future.result()
→ GetResult.Response
```

---

## 34.3 `result_response` 不是最终 Result 对象

```python
result_response = future.result()
```

得到的是：

```text
GetResult.Response
```

真正的数据：

```python
result = result_response.result
```

---

## 34.4 Feedback 也有包装层

```python
feedback_msg
```

不是：

```text
Count.Feedback
```

而：

```python
feedback_msg.feedback
```

才是：

```text
Count.Feedback
```

---

## 34.5 `succeed()` 不等于 `return result`

```python
goal_handle.succeed()
```

表示：

```text
设置任务最终状态
```

而：

```python
return result
```

表示：

```text
返回最终数据
```

---

# 35. 一句话记住 Action

可以把 ROS2 Action 看成：

```text
Client 提交任务 Goal
        ↓
Server 决定接不接
        ↓
建立 GoalHandle
        ↓
Server 执行任务
        ↓
不断发送 Feedback
        ↓
成功 / 失败 / 取消
        ↓
返回 Result
```

Client 的核心流程：

```text
Goal
→ Future
→ GoalHandle
→ Future
→ Result
```

Server 的核心流程：

```text
goal_request
→ ACCEPT
→ goal_handle
→ Feedback
→ succeed/abort/canceled
→ Result
```

---

# 36. 推荐记忆口诀

```text
Goal
=
我要你做什么

GoalHandle
=
管理这一次任务

Feedback
=
现在做到哪里

Result
=
最终做出了什么

Future 1
=
等 Server 接单

Future 2
=
等 Server 做完

ACCEPT / REJECT
=
接不接任务

SUCCEED / ABORT / CANCELED
=
任务最终怎么结束
```

---

# 37. 最终总图

```text
                    ROS2 ACTION

Client                                        Server
  │                                             │
  │ Count.Goal                                  │
  │ target = 5                                  │
  ├────────────────────────────────────────────►│
  │                                             │
  │                                      goal_callback()
  │                                             │
  │                                      ACCEPT / REJECT
  │                                             │
  │◄──────── Goal accepted / rejected ──────────┤
  │                                             │
  │ ClientGoalHandle                    ServerGoalHandle
  │                                             │
  │                                     goal_handle.request
  │                                             │
  │                                      execute_callback()
  │                                             │
  │◄──────────── FeedbackMessage ───────────────┤
  │             ↓                               │
  │      feedback_msg.feedback                  │
  │             ↓                               │
  │       Count.Feedback                        │
  │                                             │
  │◄──────────── FeedbackMessage ───────────────┤
  │                                             │
  │                                      succeed()
  │                                         或
  │                                      abort()
  │                                         或
  │                                      canceled()
  │                                             │
  │◄────────── GetResult.Response ──────────────┤
  │               ↓                             │
  │       result_response.result                │
  │               ↓                             │
  │          Count.Result                       │
  │                                             │
```

---

## 38. 学习重点

初学 Action 时，建议优先掌握：

1. `.action` 中 Goal / Result / Feedback 的定义；
2. `ActionServer` 和 `ActionClient` 的创建；
3. `goal_callback()` 和 `execute_callback()` 的区别；
4. `goal_request` 到 `ServerGoalHandle` 的关系；
5. ClientGoalHandle 与 ServerGoalHandle 的区别；
6. 两个 Future 为什么存在；
7. Feedback 和 Result 的包装层；
8. `succeed()` / `abort()` / `canceled()`；
9. Cancel 机制；
10. 最后再学习多 Goal、Executor、Callback Group 和并发 Action。

掌握这些以后，ROS2 Action 的主干机制基本就完整了。
