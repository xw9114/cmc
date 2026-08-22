# ROS 2 Topic 与 Service 学习笔记

这是一个用于整理 ROS 2 通信机制学习资料的个人笔记仓库，当前主要覆盖 **Topic（话题）**、**Service（服务）**、自定义 `.srv` 接口，以及基于 Python `rclpy` 的常用节点函数。

## 内容目录

### Topic

- [ROS 2 通信机制——话题（Topic）](./ROS2通信机制-话题Topic知识点总结.md)

### Service

- [ROS 2 通信机制——服务（Service）](./ROS2通信机制-服务Service知识点总结.md)
- [ROS 2 Service 自定义 `.srv` 学习笔记](./ROS2_Service_自定义srv学习笔记.md)

### Topic + Service 综合实践

- [ROS 2 Topic 与 Service 重要函数学习笔记（Python / rclpy）](./ROS2_Topic_Service_重要函数学习笔记.md)
- [ROS 2 Topic + Service 综合项目知识总结](./ROS2_Topic_Service_项目知识总结.md)

## 推荐阅读顺序

1. 先阅读 Topic 和 Service 的通信机制基础。
2. 学习自定义 `.srv` 接口及功能包组织方式。
3. 熟悉 `rclpy` 中的节点、发布者、订阅者、客户端和服务端函数。
4. 最后结合综合项目笔记理解 Topic 与 Service 的协同使用。

## 学习范围

- ROS 2 工作空间与功能包的基本结构
- Topic 发布、订阅与回调
- Service 客户端、服务端与请求响应流程
- 自定义 `.srv` 文件和接口功能包
- Python `rclpy` 常用 API
- Topic 与 Service 的综合通信项目

## 说明

本仓库以学习笔记和知识总结为主，示例中的命令、路径和代码应结合本地 ROS 2 发行版及工作空间环境进行验证。
