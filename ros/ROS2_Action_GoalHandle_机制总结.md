# ROS2 Action：Goal 接受/拒绝与 GoalHandle 机制总结

## 1. Action 的基本流程

ROS2 Action 的通信流程可以概括为：

```text
Client
  |
  | 发送 Goal
  ↓
Server
  |
  | 判断是否接受
  ↓
Goal Response
  |
  ├── accepted = False
  │      ↓
  │   Goal 被拒绝
  │   不执行任务
  │   不进入 execute_callback()
  │   不产生正常 Feedback / Result
  │
  └── accepted = True
         ↓
      进入执行阶段
         ↓
      execute_callback(goal_handle)
         ↓
      Feedback
         ↓
      Result
```

---

## 2. Client 发送 Goal

客户端先创建 Goal 对象：

```python
goal = CountUntil.Goal()
goal.target = 5
```

然后发送：

```python
goal_future = self.client.send_goal_async(
    goal,
    feedback_callback=self.feedback_callback
)
```

`send_goal_async()` 的作用：

1. 把 Goal 发送给 Server
2. 注册 Feedback 回调
3. 返回一个 `goal_future`

注意：

```text
goal_future
```

等待的不是最终 Result，而是：

> Server 是否接受这个 Goal。

---

## 3. Goal 被接受或拒绝

当 Server 对 Goal 做出决定后：

```python
goal_handle = future.result()
```

客户端得到一个：

```text
ClientGoalHandle
```

然后检查：

```python
goal_handle.accepted
```

### 接受

```python
goal_handle.accepted == True
```

表示：

> Server 接受了这个任务，可以进入执行阶段。

### 拒绝

```python
goal_handle.accepted == False
```

表示：

> Server 拒绝了这个任务。

典型写法：

```python
goal_handle = future.result()

if not goal_handle.accepted:
    self.get_logger().info("Goal rejected")
    return
```

---

## 4. 被拒绝的 Goal 会不会返回信息？

会。

Server 至少会返回：

```text
accepted = False
```

告诉 Client：

> 这个 Goal 被拒绝了。

所以：

```text
拒绝 Goal
≠
什么都不返回
```

更准确地说：

```text
拒绝 Goal
=
返回“拒绝”这一结果
+
不执行后续任务
```

---

## 5. 被拒绝后会不会执行 execute_callback()？

正常情况下不会。

也就是说：

```text
Goal
 ↓
Server 判断
 ↓
accepted = False
 ↓
结束
```

不会进入：

```python
async def execute_callback(self, goal_handle):
```

因此也不会正常产生：

```text
Feedback
Result
```

---

## 6. Goal 被接受后会发生什么？

如果：

```python
goal_handle.accepted == True
```

Server 会真正处理这个 Goal。

例如：

```python
async def execute_callback(self, goal_handle):

    target = goal_handle.request.target

    feedback = CountUntil.Feedback()

    for i in range(target):

        feedback.current = i

        goal_handle.publish_feedback(feedback)

    goal_handle.succeed()

    result = CountUntil.Result()
    result.success = True

    return result
```

流程：

```text
accepted = True
      ↓
execute_callback()
      ↓
读取 Goal
      ↓
执行任务
      ↓
publish_feedback()
      ↓
succeed()
      ↓
return result
```

---

## 7. goal_handle 是什么？

`goal_handle` 可以理解为：

> 某一个 Goal 的控制句柄 / 管理句柄。

它不是 Goal 数据本身。

### Goal

表示：

```text
任务内容
```

例如：

```python
goal.target = 5
```

### GoalHandle

表示：

```text
对这个任务进行管理和操作的对象
```

---

## 8. ClientGoalHandle 和 ServerGoalHandle

同一个 Goal 在 Client 和 Server 两端分别有一个管理句柄。

```text
                 同一个 Goal
                     |
          -------------------------
          |                       |
          ↓                       ↓
ClientGoalHandle         ServerGoalHandle
```

它们：

- 管理的是同一个 Goal
- 通过同一个 `goal_id` 对应
- 不是同一个 Python 对象
- 功能不同

---

## 9. ClientGoalHandle 的作用

Client 端通常：

```python
goal_handle = future.result()
```

得到：

```text
ClientGoalHandle
```

常用操作：

```python
goal_handle.accepted
```

查看是否接受。

```python
goal_handle.get_result_async()
```

等待最终 Result。

```python
goal_handle.cancel_goal_async()
```

请求取消任务。

所以 ClientGoalHandle 可以理解为：

> Client 对这次 Goal 的管理入口。

---

## 10. ServerGoalHandle 的作用

Server 端：

```python
async def execute_callback(self, goal_handle):
```

这里的 `goal_handle` 实际上是：

```text
ServerGoalHandle
```

常用操作：

```python
goal_handle.request
```

读取 Goal 数据。

```python
goal_handle.publish_feedback(feedback)
```

发送 Feedback。

```python
goal_handle.succeed()
```

标记任务成功。

```python
goal_handle.abort()
```

标记任务失败。

```python
goal_handle.canceled()
```

标记任务取消。

---

## 11. goal_handle.request 是什么？

假设 Action：

```text
int32 target
---
bool success
---
int32 current
```

Client：

```python
goal = CountUntil.Goal()
goal.target = 5
```

Server：

```python
goal_handle.request.target
```

结构可以理解为：

```text
goal_handle
    |
    └── request
           |
           └── Goal 对象
                  |
                  └── target = 5
```

所以：

```python
goal_handle.request
```

本质上就是：

> 当前任务中的 Goal 数据对象。

而：

```python
goal_handle.request.target
```

就是：

> 读取 Goal 对象里的 target 属性。

---

## 12. 为什么 Feedback 由 goal_handle 发送？

Server 写：

```python
goal_handle.publish_feedback(feedback)
```

而不是：

```python
self.server.publish_feedback(feedback)
```

因为 Feedback 属于某一个具体 Goal。

一个 Action Server 可能同时管理多个 Goal：

```text
ActionServer
   |
   ├── Goal A
   |      └── goal_handle_A
   |
   └── Goal B
          └── goal_handle_B
```

所以：

```python
goal_handle_A.publish_feedback(...)
```

表示：

> 给 Goal A 发送反馈。

而：

```python
goal_handle_B.publish_feedback(...)
```

表示：

> 给 Goal B 发送反馈。

---

## 13. Feedback 在 Client 端怎么接收？

Client 发送 Goal 时：

```python
self.client.send_goal_async(
    goal,
    feedback_callback=self.feedback_callback
)
```

这里：

```python
feedback_callback=
```

不是发送 Feedback。

它的作用是：

> 注册 Feedback 接收函数。

真正发送 Feedback 的是 Server：

```python
goal_handle.publish_feedback(feedback)
```

Client 收到后：

```python
def feedback_callback(self, feedback_msg):

    feedback = feedback_msg.feedback

    print(feedback.current)
```

---

## 14. Result 怎么获取？

Goal 被接受后：

```python
result_future = goal_handle.get_result_async()
```

这里：

```text
result_future
```

等待：

> 任务真正执行完成后的最终结果。

所以 Action Client 中通常有两个 Future：

```text
goal_future
    ↓
等待 Goal 是否被接受
    ↓
ClientGoalHandle

result_future
    ↓
等待任务真正完成
    ↓
Result
```

---

## 15. accepted=False 时还能不能操作 goal_handle？

`ClientGoalHandle` 对象仍然存在。

例如：

```python
goal_handle.accepted
```

仍然可以读取。

但是：

```text
accepted = False
```

说明：

> 这个 Goal 没有真正进入执行阶段。

因此没有正常运行中的任务可以：

- 等待正常执行结果
- 获取执行中的 Feedback
- 正常取消正在执行的任务

所以标准写法是：

```python
if not goal_handle.accepted:
    return
```

这里的 `return` 不是因为 GoalHandle 对象不存在，而是：

> 防止程序继续对一个未被接受的任务进行后续操作。

---

## 16. 最终核心总结

### Goal

```text
任务数据
```

例如：

```python
goal.target = 5
```

### ClientGoalHandle

```text
Client 对这个 Goal 的管理句柄
```

主要负责：

```text
是否接受
等待 Result
请求 Cancel
```

### ServerGoalHandle

```text
Server 对这个 Goal 的管理句柄
```

主要负责：

```text
读取 Goal
执行任务
发送 Feedback
设置成功 / 失败 / 取消
```

### Goal 被拒绝

```text
返回 accepted=False
不执行 execute_callback()
不产生正常 Feedback
不产生正常任务 Result
```

### Goal 被接受

```text
accepted=True
    ↓
execute_callback()
    ↓
Feedback
    ↓
Result
```

---

## 17. 一句话记忆

> Goal 是“任务内容”，GoalHandle 是“这个任务的控制句柄”；ClientGoalHandle 负责客户端侧管理，ServerGoalHandle 负责服务端侧执行和反馈。
