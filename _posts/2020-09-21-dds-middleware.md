---
title: DDS(Data Distribution Service) 中间件
tags:
  - ros2
  - dds
---

ROS2 底层改为使用 DDS 来实现，原因及相关评估见 [ROS on DDS](https://design.ros2.org/articles/ros_on_dds.html)。

总结一下就是 DDS 是一个定义良好的端到端的网络通信中间件，可以简化自己构建端到端中间件的工作量，包括代码的工作量和文档的工作量。而且这种已经规范化的中间件可以使 ROS 更好的被评审，审计以及更方便和其他系统互操作。

DDS([Data Distribution Service](https://en.wikipedia.org/wiki/Data_Distribution_Service)) 是由 [Object Management Group (OMG)](http://www.omg.org/) 最早于2004年发布的一种基于**发布-订阅（publish-subscribe）模式**的端到端网络通信中间件，它可用于实时系统中实现可靠，高性能，可扩展，可互操作的数据传输。

DDS 只是一个标准以及一套API定义，各厂商可以根据该标准实现自己的 DDS,这有点类似于 OpenGL。比较有名的 DDS 实现包括 `eProsima FastDDS`,`Eclipse Cyclone DDS`,`RTI Context`,`ADLINK Opensplice`等。ROS2 对它们的支持情况如下表所示：

![rmw/dds](/data/2020-09-21-11-23-49.png)

详见： [About different ROS 2 DDS/RTPS vendors](https://index.ros.org/doc/ros2/Concepts/DDS-and-ROS-middleware-implementations/)。

可以根据需要配置 ROS2 使用不同的 DDS 实现。

DDS 标准定义见 [About the Data Distribution Service Specification Version 1.4](https://www.omg.org/spec/DDS/)。

DDS C++ API 定义见 [About the ISO/IEC C++ 2003 Language DDS PSM Specification Version 1.0](https://www.omg.org/spec/DDS-PSM-Cxx/)。

DDS 是完全分布式的，没有中心节点，任何两个节点之间都是直接通信的，示意图如下：

![dds](/data/dds.svg)

为了实现这样的系统，首先要解决的是各节点的互相发现问题。DDS 本身并没有定义应该如何发现各节点，而是在 `RTPS` 中定义该行为。RTPS(Real-time Publish-Subscribe Protocol) 用来支撑 DDS 的实现，是 DDS 底层使用的协议，用来保证各种 DDS 的实现之间可以互操作。在 RTPS 中定义了两个独立的发现协议，分别是 PDP(Participant Discovery Protocol) 和 EDP(Endpoint Discovery Protocol)，PDP 用于发现各参与节点，EDP 用于发现每个节点提供的所有端点（Endpoints）。PDP 的实现可以通过单播列表或者组播实现。RTPS 允许实现者实现不同的 PDP 和 EDP 协议，但是至少要实现 SPDP(Simple PDP) 和 SEDP(Simple EDP) 协议。关于协议的具体规定见 [RTPS 规范](https://www.omg.org/spec/DDSI-RTPS/)。

在 ROS2 的实现中，默认采用的 DDS 实现为 `eProsima FastDDS`。eProsima FastDDS 是 Apache2 的开源协议，代码托管在 github 上 [eProsima/Fast-DDS](https://github.com/eProsima/Fast-DDS)。文档见 [DDS API — Fast DDS 2.0.0 documentation](https://fast-dds.docs.eprosima.com/en/latest/)。eProsima 提供的一张图可以很好的来表示 DDS 的实现原理：

![fastdds](/data/2020-09-21-15-23-59.png)

可以看到，DDS 基于 Topic 来实现发布-订阅模式，而且没有中心节点。是一种很好的端到端分布式通信中间件。ROS2 使用它提供了面向机器人领域的开发框架。

DDS 仅提供给予发布-订阅模式的通信，有些框架对它进行了包装，在发布-订阅的基础上提供了RPC的能力，比如同样来自 OMG 的 [DDS-RPC](https://www.omg.org/spec/DDS-RPC/)，以及来自 eProsima 的 [eProsima RPC over DDS](https://www.eprosima.com/index.php/products-all/tools/eprosima-rpc-over-dds-all)。

将基于发布订阅模式的 DDS 包装成 RPC 的方法如下：

![rpc over dds](/data/2020-09-21-15-41-44.png)

每个 RPC 调用都会是用2个 Ｔopic 来实现，一个用于发送请求，一个用于接收回复。更详细的信息可以参考官方文档。

-----
参考文档：

* [RPC over DDS](https://www.eprosima.com/index.php/resources-all/rpc-over-dds)
* [eProsima/Fast-DDS: The most complete DDS - Proven: Plenty of success cases.](https://github.com/eProsima/Fast-DDS)
* [DDS API — Fast DDS 2.0.0 documentation](https://fast-dds.docs.eprosima.com/en/latest/)
* [What’s in the DDS Standard?](https://www.dds-foundation.org/omg-dds-standard/)
* [Data Distribution Service - Wikipedia](https://en.wikipedia.org/wiki/Data_Distribution_Service)
