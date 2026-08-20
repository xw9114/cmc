# ROS 2 Service 自定义 `.srv` 学习笔记

## 一、ROS 2 工作空间与功能包

### 1. ROS 2 工作空间基本结构

以当前工作空间 `chapter2` 为例：

```text
chapter2/
├── src/          # 自己编写的 ROS 2 功能包源码
├── build/        # 编译产生的中间文件
├── install/      # 编译完成后的文件
└── log/          # colcon 编译日志
```

#### `src`

`src` 是真正编写代码的地方，里面存放一个个 ROS 2 package，例如：

```text
chapter2/src/
├── service_example/
└── my_interfaces/
```

一个 workspace 中可以包含很多 package。

#### `build`

执行：

```bash
colcon build
```

以后产生的编译中间文件会放在这里。

一般不需要手动修改。

#### `install`

保存编译完成后的结果。

ROS 2 最终使用的是这里安装好的 package，而不是直接使用 `src` 中的源码。

#### `log`

保存 `colcon build` 过程中产生的编译日志。

---

### 2. ROS 2 Package 是什么？

Package 是 ROS 2 组织代码的基本单位。

例如：

```text
service_example
```

是一个 package，

```text
my_interfaces
```

也是一个 package。

创建 package：

```bash
cd ~/chapter2/src
ros2 pkg create my_interfaces --build-type ament_cmake
```

也可以写成：

```bash
ros2 pkg create --build-type ament_cmake my_interfaces
```

两种写法作用相同。

整体层级关系：

```text
workspace
   ↓
chapter2
   ↓
src
   ↓
package
   ↓
Python / C++ 源代码、接口文件等
```

---

### 3. `ament_python` 与 `ament_cmake`

Python 节点通常创建为：

```bash
ros2 pkg create service_example --build-type ament_python
```

它的核心配置文件是：

```text
setup.py
```

而自定义：

```text
.msg
.srv
.action
```

接口时，通常单独创建一个：

```text
ament_cmake
```

类型的接口包：

```bash
ros2 pkg create my_interfaces --build-type ament_cmake
```

它的核心构建配置文件是：

```text
CMakeLists.txt
```

可以简单记成：

```text
ament_python
    ↓
setup.py

ament_cmake
    ↓
CMakeLists.txt
```

---

### 4. 为什么创建 `my_interfaces` 后没有 `srv`？

执行：

```bash
ros2 pkg create my_interfaces --build-type ament_cmake
```

以后通常得到：

```text
my_interfaces/
├── CMakeLists.txt
├── package.xml
├── include/
└── src/
```

这是因为 ROS 2 只知道：

> 创建一个普通的 `ament_cmake` package。

ROS 2 并不知道：

```text
my_interfaces
```

这个名字意味着“我要创建接口”。

`my_interfaces` 只是我们自己给 package 起的名字。

所以需要自己建立：

```bash
cd ~/chapter2/src/my_interfaces
mkdir srv
```

此时：

```text
my_interfaces/
├── CMakeLists.txt
├── package.xml
├── include/
├── src/
└── srv/
```

对于当前这种纯接口 package：

```text
src/
include/
```

暂时可以不用。

以后需要自定义其他接口，也同样创建对应目录：

```text
msg/       → 自定义 Message
srv/       → 自定义 Service
action/    → 自定义 Action
```

---

## 二、自定义 `.srv` 接口

### 1. `.srv` 是什么？

`.srv` 是 ROS 2 Service 的接口定义文件。

它最基本的格式是：

```text
请求数据
---
响应数据
```

例如：

```text
int64 n
int64 m
---
int64 num
```

可以理解成：

```text
Request
├── n
└── m

Response
└── num
```

客户端发送：

```python
request.n
request.m
```

服务端返回：

```python
response.num
```

所以 `.srv` 本质上就是规定：

> Service 的 Request 长什么样，以及 Response 长什么样。

---

### 2. 创建自己的 `.srv`

进入接口 package：

```bash
cd ~/chapter2/src/my_interfaces
```

创建：

```bash
touch srv/AddTwoNumbers.srv
```

编辑：

```bash
nano srv/AddTwoNumbers.srv
```

写入：

```text
int64 n
int64 m
---
int64 num
```

最终目录：

```text
my_interfaces/
├── CMakeLists.txt
├── package.xml
└── srv/
    └── AddTwoNumbers.srv
```

注意数据类型：

```text
int64
```

必须连在一起。

---

### 3. 为什么 `.srv` 不能随便写 `int`？

`.srv` 不是 Python 文件，而是 ROS 2 的接口定义文件。

ROS 2 需要支持：

```text
Python
C++
不同电脑
不同操作系统
```

之间进行通信。

因此接口中的数据类型必须明确规定数据长度。

常见整数类型：

```text
int8
int16
int32
int64

uint8
uint16
uint32
uint64
```

例如：

```text
int64 n
```

表示：

> `n` 是一个 64 位有符号整数。

而不能简单写：

```text
int n
```

其他常见类型还有：

```text
float32
float64
string
bool
```

例如：

```text
string name
int32 age
float64 score
---
bool success
string message
```

---

### 4. `.srv` 文件名和字段名分别决定什么？

假设文件叫：

```text
AddTwoNumbers.srv
```

那么编译以后 Python 中导入：

```python
from my_interfaces.srv import AddTwoNumbers
```

所以：

```text
.srv 文件名
       ↓
生成的接口类型名
```

即：

```text
AddTwoNumbers.srv
        ↓
AddTwoNumbers
```

而 `.srv` 内部内容：

```text
int64 n
int64 m
---
int64 num
```

决定 Python 中：

```python
request.n
request.m
response.num
```

即：

```text
字段名
   ↓
Request / Response 对象中的属性名
```

完整关系：

```text
AddTwoNumbers.srv
        ↓
AddTwoNumbers

int64 n
int64 m
---
int64 num
        ↓
request.n
request.m
response.num
```

---

## 三、配置 `CMakeLists.txt`

### 1. `CMakeLists.txt` 是什么？

可以把：

```text
CMakeLists.txt
```

理解成：

> `ament_cmake` package 的“构建说明书”。

它主要告诉构建系统：

```text
项目叫什么
依赖什么
需要生成什么
应该如何构建
```

一个基础结构：

```cmake
cmake_minimum_required(VERSION 3.8)

project(my_interfaces)

find_package(ament_cmake REQUIRED)

...

ament_package()
```

---

### 2. `cmake_minimum_required`

```cmake
cmake_minimum_required(VERSION 3.8)
```

表示：

> 构建当前项目至少需要 CMake 3.8。

---

### 3. `project`

```cmake
project(my_interfaces)
```

用于定义当前项目名称。

执行以后：

```cmake
${PROJECT_NAME}
```

就代表：

```text
my_interfaces
```

所以：

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/AddTwoNumbers.srv"
)
```

和：

```cmake
rosidl_generate_interfaces(my_interfaces
  "srv/AddTwoNumbers.srv"
)
```

本质上都可以。

但是使用：

```cmake
${PROJECT_NAME}
```

更规范。

因为如果以后项目名改成：

```cmake
project(robot_interfaces)
```

那么：

```cmake
${PROJECT_NAME}
```

会自动变成：

```text
robot_interfaces
```

其他地方不需要再手动修改。

---

### 4. 为 `.srv` 增加接口生成器

首先增加：

```cmake
find_package(rosidl_default_generators REQUIRED)
```

意思是：

> 找到 ROS 2 的接口代码生成工具。

然后加入：

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/AddTwoNumbers.srv"
)
```

意思是：

> 将 `AddTwoNumbers.srv` 生成成 ROS 2 真正可以使用的接口代码。

因此核心内容是：

```cmake
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/AddTwoNumbers.srv"
)

ament_package()
```

其中：

```text
find_package(...)
```

负责：

> 找到需要使用的工具。

而：

```text
rosidl_generate_interfaces(...)
```

负责：

> 指定具体需要生成哪个接口。

---

### 5. 为什么 `.srv` 不能直接被 Python 使用？

刚写完：

```text
AddTwoNumbers.srv
```

时，它只是一个普通的接口描述文件：

```text
int64 n
int64 m
---
int64 num
```

此时 Python 中还不存在：

```python
AddTwoNumbers.Request()
```

ROS 2 必须经过接口生成过程：

```text
AddTwoNumbers.srv
        ↓
rosidl_generate_interfaces()
        ↓
colcon build
        ↓
生成 Python / C++ 接口代码
        ↓
Python 才可以 import
```

之后才能使用：

```python
from my_interfaces.srv import AddTwoNumbers
```

以及：

```python
request = AddTwoNumbers.Request()
```

所以 `.srv` 可以理解成：

> 接口的“设计图”。

而真正可以被 Python/C++ 使用的接口代码，需要 ROS 2 在编译时自动生成。

---

## 四、配置 `package.xml`

### 1. `package.xml` 的作用

如果：

```text
CMakeLists.txt
```

回答的是：

> 这个 package 具体怎么构建？

那么：

```text
package.xml
```

回答的是：

> 这个 package 是谁、依赖谁、属于什么类型？

因此可以把 `package.xml` 理解成：

> ROS 2 package 的“身份证 + 依赖清单”。

例如：

```xml
<name>my_interfaces</name>
<version>0.0.0</version>
<description>Custom service interfaces</description>
```

用于记录 package 的基本信息。

---

### 2. `package.xml` 中的依赖

原本存在：

```xml
<buildtool_depend>ament_cmake</buildtool_depend>
```

表示：

> 构建这个 package 需要 `ament_cmake`。

现在因为需要生成 `.srv`，所以增加：

```xml
<buildtool_depend>rosidl_default_generators</buildtool_depend>
```

表示：

> 构建阶段需要 ROS 2 的接口生成工具。

再增加：

```xml
<exec_depend>rosidl_default_runtime</exec_depend>
```

表示：

> 生成出来的接口在运行阶段需要 ROS 2 接口运行支持。

还需要：

```xml
<member_of_group>rosidl_interface_packages</member_of_group>
```

表示：

> 当前 package 属于 ROS 2 interface package。

因此自定义接口时主要增加：

```xml
<buildtool_depend>rosidl_default_generators</buildtool_depend>

<exec_depend>rosidl_default_runtime</exec_depend>

<member_of_group>rosidl_interface_packages</member_of_group>
```

---

### 3. `<export>` 的作用

```xml
<export>
  <build_type>ament_cmake</build_type>
</export>
```

用于告诉 ROS 2：

> 当前 package 使用 `ament_cmake` 作为构建类型。

因为创建 package 时使用：

```bash
ros2 pkg create my_interfaces --build-type ament_cmake
```

所以这里是：

```xml
<build_type>ament_cmake</build_type>
```

如果是 Python package，则通常为：

```xml
<build_type>ament_python</build_type>
```

---

### 4. `package.xml` 与 `CMakeLists.txt` 的区别

可以直接记：

```text
package.xml
    ↓
我是谁？
我依赖谁？
我是什么类型的 package？

CMakeLists.txt
    ↓
具体怎么构建？
具体生成哪个接口？
```

例如：

```text
package.xml
    ↓
声明：
需要 rosidl_default_generators

CMakeLists.txt
    ↓
find_package(rosidl_default_generators REQUIRED)
    ↓
rosidl_generate_interfaces(...)
```

所以两者是配合关系，而不是重复关系。

---

## 五、编译与 ROS 2 环境

### 1. 编译自定义接口

配置完成以后，需要返回 workspace 根目录：

```bash
cd ~/chapter2
```

执行：

```bash
colcon build
```

不要在：

```text
~/chapter2/src
```

里面执行。

整个过程：

```text
src/
 ↓
存放源码

colcon build
 ↓
开始构建

build/
 ↓
产生编译中间文件

install/
 ↓
保存最终构建结果
```

接口生成过程可以理解成：

```text
AddTwoNumbers.srv
        ↓
CMakeLists.txt
        ↓
rosidl_generate_interfaces()
        ↓
colcon build
        ↓
install/
        ↓
Python / C++ 可以使用
```

---

### 2. ROS 2 Humble 与自己的工作空间

当前环境实际上存在两层：

```text
ROS 2 Humble
/opt/ros/humble
        ↓
自己的 workspace
~/chapter2
```

新开终端以后，推荐按照下面顺序加载：

```bash
source /opt/ros/humble/setup.bash
source ~/chapter2/install/setup.bash
```

第一句：

```bash
source /opt/ros/humble/setup.bash
```

加载 ROS 2 Humble 基础环境。

第二句：

```bash
source ~/chapter2/install/setup.bash
```

加载自己编译的 workspace。

整体依赖关系：

```text
Ubuntu
  ↓
ROS 2 Humble
  ↓
chapter2
  ↓
service_example
my_interfaces
```

自己的 workspace 并不是一套独立 ROS 2 环境，而是建立在 ROS 2 Humble 基础环境之上的。

---

## 六、在 Python 中使用自定义 Service

### 1. 导入自定义接口

因为定义的是：

```text
my_interfaces/srv/AddTwoNumbers.srv
```

所以 Python 中：

```python
from my_interfaces.srv import AddTwoNumbers
```

其中：

```text
my_interfaces
      ↓
package 名

srv
      ↓
Service 接口

AddTwoNumbers
      ↓
由 AddTwoNumbers.srv 生成的接口类型
```

---

### 2. 创建 Client

```python
self.client = self.create_client(
    AddTwoNumbers,
    "add_two_ints"
)
```

这里的：

```python
AddTwoNumbers
```

表示：

> Client 使用什么 Service 接口类型。

而：

```python
"add_two_ints"
```

表示：

> ROS 2 中这个 Service 的名字。

---

### 3. 创建 Request

```python
request = AddTwoNumbers.Request()
```

因为 `.srv` 定义：

```text
int64 n
int64 m
---
int64 num
```

所以：

```python
request.n = 10
request.m = 20
```

发送请求：

```python
future = self.client.call_async(request)
```

等待以后得到：

```python
response = future.result()
```

读取结果：

```python
response.num
```

完整对应关系：

```text
AddTwoNumbers.srv

int64 n
int64 m
---
int64 num

        ↓

Python

request.n
request.m

response.num
```

---

### 4. Service 类型名与 Service 名的区别

例如：

```python
self.client = self.create_client(
    AddTwoNumbers,
    "add_two_ints"
)
```

这里：

```text
AddTwoNumbers
```

是：

> Service 接口类型。

来自：

```text
AddTwoNumbers.srv
```

而：

```text
add_two_ints
```

是：

> ROS Graph 中 Service 的名字。

Service 名可以自己定义。

但是 Client 和 Server 必须使用完全相同的名字：

```python
"add_two_ints"
```

才能互相找到。

---

### 5. Python 类名、Node 名、Service 名、接口名

这些名字需要明确区分。

例如：

```python
class AddTwoNumbersClient(Node):
```

这里：

```text
AddTwoNumbersClient
```

只是：

> Python 类名。

而：

```python
super().__init__("add_two_ints_client")
```

这里：

```text
add_two_ints_client
```

是：

> ROS Node 名。

因此至少存在下面几种名字：

```text
AddTwoNumbers.srv
→ 接口文件名

AddTwoNumbers
→ Service 接口类型名

add_two_ints
→ Service 名

add_two_ints_client
→ Node 名

AddTwoNumbersClient
→ Python 类名
```

它们属于不同层次，不要混在一起。

---

## 七、ROS 2 Service 基础排错

### 1. `ros2 service list` 卡住

执行：

```bash
ros2 service list
```

正常情况下应该很快返回结果。

即使当前没有自己创建的 Service，也不应该一直卡住。

如果：

```bash
ros2 service list
```

一直没有返回，可能与以下部分有关：

```text
ROS 2 基础环境
ROS 2 daemon
DDS
ROS Graph
```

本次遇到该问题时，通过重新加载：

```bash
source /opt/ros/humble/setup.bash
```

恢复正常。

因此每次新开终端，推荐：

```bash
source /opt/ros/humble/setup.bash
source ~/chapter2/install/setup.bash
```

---

### 2. ROS 2 daemon

ROS 2 CLI 的很多查询命令会使用 daemon，例如：

```bash
ros2 node list
ros2 service list
ros2 topic list
```

检查 daemon：

```bash
ros2 daemon status
```

如果异常，可以重新启动：

```bash
ros2 daemon stop
ros2 daemon start
```

然后再次测试：

```bash
ros2 service list
```

---

## 八、自定义 `.srv` 完整流程

### 第一步：创建接口 package

```bash
cd ~/chapter2/src

ros2 pkg create my_interfaces --build-type ament_cmake
```

### 第二步：创建 `srv` 文件夹

```bash
cd my_interfaces
mkdir srv
```

### 第三步：创建 `.srv`

```bash
touch srv/AddTwoNumbers.srv
```

内容：

```text
int64 n
int64 m
---
int64 num
```

### 第四步：修改 `CMakeLists.txt`

增加：

```cmake
find_package(rosidl_default_generators REQUIRED)
```

以及：

```cmake
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/AddTwoNumbers.srv"
)
```

核心结构：

```cmake
find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/AddTwoNumbers.srv"
)

ament_package()
```

### 第五步：修改 `package.xml`

增加：

```xml
<buildtool_depend>rosidl_default_generators</buildtool_depend>

<exec_depend>rosidl_default_runtime</exec_depend>

<member_of_group>rosidl_interface_packages</member_of_group>
```

### 第六步：编译

回到 workspace：

```bash
cd ~/chapter2
```

执行：

```bash
colcon build
```

### 第七步：加载环境

```bash
source /opt/ros/humble/setup.bash
source ~/chapter2/install/setup.bash
```

### 第八步：检查接口

```bash
ros2 interface show my_interfaces/srv/AddTwoNumbers
```

应该看到：

```text
int64 n
int64 m
---
int64 num
```

### 第九步：Python 中导入

```python
from my_interfaces.srv import AddTwoNumbers
```

创建 Request：

```python
request = AddTwoNumbers.Request()

request.n = 10
request.m = 20
```

获取 Response：

```python
response.num
```

---

## 九、整体知识框架

自定义 Service 并不是单独创建一个 `.srv` 文件就结束了。

完整关系是：

```text
my_interfaces/
│
├── package.xml
│      ↓
│   声明：
│   package 是谁
│   package 依赖谁
│   是否属于 interface package
│
├── CMakeLists.txt
│      ↓
│   指定：
│   使用 rosidl 接口生成工具
│   具体生成哪个 .srv
│
└── srv/
       │
       └── AddTwoNumbers.srv
                ↓
         定义 Request / Response

                ↓

          colcon build

                ↓

ROS 2 自动生成 Python / C++ 接口代码

                ↓

from my_interfaces.srv import AddTwoNumbers

                ↓

AddTwoNumbers.Request()
AddTwoNumbers.Response()

                ↓

Client
发送 Request

                ↓

Server
callback 处理

                ↓

返回 Response
```

## 核心总结

```text
workspace
→ 管理多个 ROS 2 package

package
→ ROS 2 代码组织的基本单位

.srv
→ 定义 Service 的 Request 和 Response

package.xml
→ 声明 package 是什么、依赖什么

CMakeLists.txt
→ 规定 package 怎么构建、生成哪个接口

rosidl_generate_interfaces()
→ 把 .srv 转换成 Python / C++ 可使用的接口代码

colcon build
→ 真正执行构建和接口生成

AddTwoNumbers.srv
→ 决定接口类型 AddTwoNumbers

.srv 内部的 n、m、num
→ 决定 request.n、request.m、response.num

Service 类型名
≠ Service 名
≠ Node 名
≠ Python 类名
```
