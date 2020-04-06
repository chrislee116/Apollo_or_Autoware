---
title: Apollo_or_ROS
date: 2020-04-03 18:09:31
toc: true
tags:
---

# 1. 面向对象

## 1.1. Apollo

- 低版本: 面向封闭场所的无人驾驶
- 高版本: 面向城市区域的无人驾驶为主

## 1.2. AutoWare 1.0

- 主要面向封闭区域

# 2. 项目架构

## 2.1. 基本架构

### 2.1.1. Apollo(完善但复杂)

![](Apollo-or-ROS/2020-04-03-18-23-16.png)

### 2.1.2. AutoWare1.0(简化但Main)

![](Apollo-or-ROS/2020-04-03-18-26-02.png)

## 2.2.操作系统

###2.2.1 调试工具与开发流程比较
||Apollo|ROS|备注|
| ---          | ---       | ---  |   ---    |
|数据可视化|cyber_monitor|rostopic echo等||
|图形化交互|dreamview，Visualizer|rviz,gazebo等，autoware下也有工具||
|开源算法|Apollo源码|其余丰富开源资料||
|系统安装|较复杂，网络条件和硬件要求高|资料清晰，简单||
|开发流程|需要在docker内编译，因各种原因，较难在自己电脑开发|流程清晰，难度不大||
|基础积累|无，一脸懵逼|开发流程框架清晰||
|入门难度|门槛高，需要较好功底|资料丰富，较易入门||
|开发难度|开发难度高|移植难度高||
|应用范围|与Apollo有合作的公司|高校和科研机构与机器人企业||

###2.2.2概念对照表与移植思路
| Apollo中概念 | ROS中概念 | 备注 | 移植思路 |
| ------------ | --------- | ---- | :------: |
|Component|无|接收到特定数据执行|可对应每一个接收topic后的callback|
|time_component|无|定时执行|可对应ros_timer，但不一定保证定时精确性|
|Node|ros_node|一个component对应一个node，在component的init中初始化node||
|channel|topic|数据通道,概念有点相似||
|reader/writer|publisher/subscriber|与channel和topic对应|可尝试转化|
|protobuf|ros_msg|pro法序列化文件效率高。Apollo通用|可尝试在ROS中安装protobuf|
|bazel|cmake|编译器|不建议改|
|gflags|无|一些标志位，数据等|可尝试在ROS安装|
|glog|ROS_LOG|信息文件|可考虑安装glog或修改为ROS的日志格式|
|Record|bag|数据记录文件||
|dag|无|配置文件|ROS在程序内配置topic，node等|

###2.2.3性能比较

Apollo 1.0-3.0一直使用**ROS**做底层的通信框架。**ROS**是主要运用于机器人领域的框架，后来才被开发者用到自动驾驶领域。从软件构架的角度说，它是一种基于消息传递通信的分布式多进程框架。开发者可以根据功能把软件拆分成为各个模块，每个模块只是负责读取和分发消息，模块间通过消息关联，并完成最终的功能。

此外，**ROS**是学术界使用最广泛的框架，对于实验各种新算法也提供了便利，正是由于ROS的以上特性，Apollo初期选择**ROS**作为研发和调试框架，用于快速验证技术方案和算法功能。总体而言， **ROS**是一种**分布式的松耦合设计**，对于开发调试来讲非常方便，但是在自动驾驶领域，**ROS**还存在一些挑战。

第一，在ROS中，计算任务都会执行在一个一个线程中，当系统中存在大量计算任务的时候，会衍生出大量的线程，对调度系统而言是一个非常大的负担，会引入很多的**线程切换**，导致系统的实时性难以保障。

第二，分布式系统的通信性能瓶颈很大。分布式架构通信性能、中心化的Master、消息格式不够灵活等问题导致调度难以满足实际需求。

通过这些年的实践和经验积累，百度无人车团队构建了一个完整的软件栈，自研了一套自动驾驶的框架，能够更加贴合使用场景，完全满足自动驾驶的需求。于是就产生了**Cyber RT**。

为了满足实时性能需求，Cyber RT做了一个专用的调度系统。它将调度的逻辑从内核空间抽离到用户空间，使用协程作为基础的调度单元。传统的多线程模型中用户的计算对应于一个个线程，而线程则对应于CPU的核心，线程的调度和切换则依赖Kernel的调度算法，可控性较低；在用户空间的调度中，用户的任务会对应于一个个协程，协程则对应于Processor，Processor对于Task的执行和切换完全在用户空间，由具体的调度器来控制。
在Cyber RT中，为了兼顾集中式部署时的高性能以及分布式部署的灵活性，提供了自适应的通信方式，尽量以最高效的方式来进行数据传输。这些通信方式对开发者来说是透明的。开发者只需要指定发送的Channel名称及类型即可， Cyber RT框架会根据模块之间的关系动态选择适合的通信方式。例如，当两个模块处于同一个进程中的时候，会直接进行指针传递，减少了序列化及内存拷贝的开销。当两个模块处于同一台机器的不同进程中的时候，会使用共享内存的方式来进行通信，避免数据的多次拷贝。当两个模块处于同一网络内的不同机器中时，则会使用Socket的方式来进行网络通信。通过自适应的通信方式，能够将模块开发与实际部署方案解耦，保证性能的同时也提供了比较好的灵活性。

###2.2.4总结：
||Apollo|ROS||
|:----:|:----:|:----:|:----:|
|通信性能|高|低||
|可靠性|高|低|
|入门难度|高|低|
|适用场景|只支持室外|多场景均有开源资料||
|难点|开发上手难|移植难度高|


想法：
1. 实验室项目无明显技术积累，系统是算法的载体。有系统建模与算法实现的能力才是核心。
2. Apollo适用于类似城市道路等要求严苛区域，实验室项目场景稍微简单，使用ROS也能够完成项目。
3. ROS开发难度较低，上手快，更加适合是整个实验室的同学一起开发的平台。在目前阶段，Apollo较难上手，但是是一个非常好的学习平台。
4. 可考虑两步走的方式。第一，尝试将Apollo系统的算法移植到ROS系统，锻炼系统建模与算法实现的能力；第二，在进行算法移植的同时进一步学习Apollo，加深对Apollo的理解，尝试利用Apollo系统进行开发调试。
5. 具体使用哪一个系统，需要先迈出实践的第一步之后，再确定。

# 3. 功能实现

## 3.1. Localization

### 3.1.1. 技术简介

#### 3.1.1.1. Apollo中的定位技术

- RTK模式: 为了方便调试，Apollo自行实现了一套RTK解算，只使用RTK的定位信息。

> 现在惯导芯片一般都会配一个板卡（NovAtel也卖这样的板卡），直接集成了RTK的定位结果。为什么我们还需要自己开发GNSS-RTK呢？ 
> 从系统的角度考虑，需要每个子模块都是可控的，举一个简单的例子，当给出一个定位结果偏了，但给出的方差很小，也就是置信度很高。我们是没办法知道原因的。

- MSF(Multiple Sensor Fusion)模式: 采用Kalman滤波器，对位置、姿态和速度进行融合。
  
  - IMU: 惯导解算
  - LIDAR: NDT匹配算法
  - GPS: RTK解算
  
  使用松耦合的方式把惯性导航解算、GNSS定位、点云定位三个子模块融合在一起。(松耦合and紧耦合: 松耦合的数据只有位置、速度、姿态，紧耦合会包括GNSS的导航参数、定位中的伪距、距离变化等。)

  使用了一个误差卡尔曼滤波器，惯性导航解算的结果用于kalman滤波器的时间更新，也就是预测；而GNSS、点云定位结果用于kalman滤波器的量测更新
  
  ![](Apollo-or-ROS/2020-04-04-11-32-05.png)

#### 3.1.1.2. Autoware(ROS)中的定位技术

- 基于3D Lidar的定位: 采用NDT点云配准算法进行定位，分别实现了
  - 基于PCL的NDT算法
  - 自行实现的NDT算法
  - 改进的NDT算法:NDT-TKU

- GNSS: 借助ROS社区，直接使用ROS开源的GNSS驱动，读取GPS-RTK的定位信息

参考文档

1. [Autoware.AI NDT文档](https://autowarefoundation.gitlab.io/autoware.auto/AutowareAuto/ndt-review.html)
2. [Autoware.AI 定位设计](https://autowarefoundation.gitlab.io/autoware.auto/AutowareAuto/localization-design.html)
3. [NDT定位算法原理](http://www.diva-portal.org/smash/get/diva2:276162/FULLTEXT02.pdf)


### 3.1.2. 技术成熟度

#### 3.1.2.1. Apollo

Apollo实现的RTK解算、MSF传感器融合定位，都有成熟的理论指导，成熟度较高。并且从Apollo最近的几个版本来看，定位模块的改动不大。

#### 3.1.2.2. Autoware

Autoware采用的NDT定位算法是2009年的一篇博士论文，目前仍被广泛应用，技术相对成熟，并且名古屋大学教授对此进行了改进，即NDT-TKU算法


### 3.1.3. 技术前沿性

从定位技术的前沿性来看，Apollo比Autoware领先，并且更加完善

#### 3.1.3.1. Apollo

从目前现有的代码来看，Apollo的RTK、MSF定位都是基于传统技术如:

- RTK解算
- GNSS/IMU解算
- NDT点云匹配
- ESKF误差卡尔曼

的堆叠，但是实际上Apollo提出了许多新的算法，如基于深度学习的点云匹配，基于深度学习的定位融合，虽然目前没有直接在Apollo代码中实现，不排除后续升级版本对定位模块的改进。

#### 3.1.3.2. Autoware

Autoware主要面向封闭环境，因此其认为，使用3D Lidar SLAM足以解决封闭环境的定位问题，而没有像Apollo那样实现GPS/IMU的融合解算，只是使用了EKF来对3D激光点云定位和GPS定位进行了融合，其`GPS定位信息转换模块`值得学习一下。

### 3.1.4. 源码开放度

#### 3.1.4.1. Apollo

Apollo虽说开源，但是核心部分还是抓的死死的，其中包括

- rtk解算`gnss_solver`
- 点云定位`lidar_locator`
- 惯导解算`sins.h`

**证据如下:**

在`modules/localization/msf/local_integ/localization_gnss_process.h`文件中，引用了

```C++
#include "include/gnss_solver.h"
```

在`modules/localization/msf/local_integ/localization_lidar.h`文件中，引用了

```C++
#include "include/lidar_locator.h"
```

在`modules/localization/msf/local_integ/localization_integ_process.h`文件中，引用了

```C++
#include "include/sins.h"
```

当然还有其他一些，上面引用的文件在源码中是找不到其影子的，因为这些头文件打包在Apollo的docker镜像中，至于对应`CPP`实现，那是不会给你哒，放心好了，早已编译成`.so`文件了

```
liblocalization_msf.so -> liblocalization_msf.so.1
liblocalization_msf.so.1 -> liblocalization_msf.so.1.0.2
liblocalization_msf.so.1.0.2
```

关于这部分的内容，更加具体的可参见文档:

1. [【Apollo】【localization】调试与分析](http://www.jeepxie.net/article/693907.html)

#### 3.1.4.2. Autoware

Autoware毕竟是基金组织，也没什么人投钱，基本上实现了的都开了

- 点云定位: NDT算法，有pcl版本的，也有自行实现版本的，有cpu版本的也有gpu版本的，最后，还有改进版本的
- GNSS: 直接使用ROS社区开源驱动

证据如下:

![](Apollo-or-ROS/2020-04-04-21-09-56.png)

### 3.1.5. 指导文档

#### 3.1.5.1. Apollo

基本为0，除了其发表的论文：“万国伟，杨晓龙，蔡仁兰，李莉，周瑶，王浩，宋诗玉。“在不同的城市场景中基于多传感器融合的精确而精确的车辆定位”，2018 IEEE国际机器人与自动化会议（ICRA），布里斯班，昆士兰州，2018年，第4670-4677页。doi：10.1109 ”

#### 3.1.5.2. Autoware

具有详细的协议栈，开发文档，API接口，程序流图，以及所实现的算法原理

![](Apollo-or-ROS/2020-04-04-23-29-50.png)

### 3.1.6. 可移植性

#### 3.1.6.1. Apollo

- 向内移植

  Apollo实现了一整套的底层接口、驱动，向内移植定位算法理论上可行。

  前提是:

  - 掌握Apollo的关于`Localization`协议栈，数据内容以及格式
  - 掌握相关底层设备的消息回调处理流程
  - 你得自己有算法

- 向外移植

  向外移植也不是不可以，但核心技术并不掌握在手中，谁会干这种事呢？

  前提工作:

  - 掌握Apollo的关于`Localization`协议栈，API接口，数据内容以及格式
  - 关于`Localization`部分所有**输入输出关系**全部掌握

#### 3.1.6.2. Autoware

完全基于ROS，移植性不言而喻。

### 3.1.7. 总结

![](Apollo-or-ROS/2020-04-04-23-29-14.png)

总结就是，还是要有自己的核心算法呀。

## 3.2. Perception

## 3.3. Decision

## 3.4. Routing/Planning
	规划(planning)模块的作用是根据感知预测的结果，当前的车辆信息和路况规划出一条车辆能够行驶的轨迹，这个轨迹会交给控制(control)模块，控制模块通过油门，刹车和方向盘使得车辆按照规划的轨迹运行。
	规划模块的轨迹是短期轨迹，即车辆短期内行驶的轨迹，长期的轨迹是routing模块规划出的导航轨迹，即起点到目的地的轨迹，规划模块会先生成导航轨迹，然后根据导航轨迹和路况的情况，沿着短期轨迹行驶，直到目的地。这点也很好理解，我们开车之前先打开导航，然后根据导航行驶，如果前面有车就会减速或者变道，超车，避让行人等，这就是短期轨迹，结合上述的方式直到行驶到目的地。

planning输入参数为:

- 预测的障碍物信息(prediction_obstacles)
- 车辆底盘(chassis)信息(车辆的速度，加速度，航向角等信息)
- 车辆当前位置(localization_estimate)
- 实际上还有高精度地图信息，不在参数中传入，而是在函数中直接读取的。

Planning模块的输出结果在"PlanningComponent::Proc()"中，为规划好的线路，发布到Control模块订阅的Topic中。
输出结果为：规划好的路径。



针对不同的场景有不同的规划器

![](Apollo-or-ROS/planning_component.png)

进行规划器设计时，需考虑多轴车辆的运动学模型，对系统进行重新设计。

需要考虑可能存在的**行驶场景**，比如说

1. 车道线行驶
2. 隧道行驶
3. 室内外交接
4. 靠边泊车
5. 窄道掉头

也需要考虑多种定位传感器的不同组合时带来的一些问题，如地图场景格式问题。



Apollo对于多种场景有多种规划器的开源，主要建模求解技术为二次规划（QP）与动态规划（DP）。以泊车为例子

![](Apollo-or-ROS\os_planner.png)

![](Apollo-or-ROS\os_step3.png)

Apollo planning模块开放度未知，还未细研究。autoware未研究。

##3.5.Control/Canbus

###3.5.1技术简介
Control是车辆与物理层交互的模型，Canbus是算法与硬件交互的链路。在接收到由planning发来的轨迹信号后，control模块驱动车辆进行对轨线的跟踪。
####3.5.1.1 Apollo中的Control/Canbus技术

- 模型建立与控制算法
	Apollo控制算法基于对车辆进行运动学与动力学建模的基础上，主要有三种，横向控制器，纵向控制器和MPC控制器。
	纵向控制通过油门和刹车控制车纵向的加减速，主要采用的是PID；而横向控制则通过控制方向盘的转动来控制前轮的方向，主要采用的是LQR，Apollo5.5更新了MARC模型参考自适应控制算法；除此之外的MPC模型预测控制，可以理解为实现同时对横向和纵向的控制。控制算法文件中还有一些组合模块（submodules)，具体使用情况不太清楚。
|方案|纵向控制|横向控制|
|:------:|:------:|:-------:|
|1|PID|LQR|
|2|MPC|MPC|
|3|PID|MARC|
|4|submodules|submodules|
以下只讨论PID与LQR控制，算法原理图如图。
![](Apollo-or-ROS\lon_pid.png)

![](Apollo-or-ROS\lat_lqr.png)

参考书：Automatic Steering Methods for Autonomous Automobile Path Tracking

- 动力学标定
	我们需要控制汽车到达某个速度，根据牛顿经典力学，只需要知道汽车的初速度和加速度，就可以知道物体一段时候后的速度。因此我们只要找到速度，加速度和油门的关系，就可以通过控制汽车的加速度来让汽车达到某个速度。也就是对速度加速度和油门的关系进行建模，得到它们之间的关系。Apollo采用的是在实际的汽车行驶过程中记录不同速度下，不同的油门值对汽车的加速度的影响，从而得到一张表格，最后通过查表的方式来得到具体的油门和刹车值大小，得到的配置最后保存在conf文件夹中。
	这个表的生成需要测试每种油门以及每种速度下的表现，但是表不可能列出无限连续的数据，因此最后还是需要通过插值的方式来得到结果。
	对于动力学标定，Apollo提供了一套工具与方法。
![](Apollo-or-ROS/offline_table.png)


- Canbus通讯与硬件底层

  Apollo中Control模块计算完cmd后按周期发送给canbus模块，再由canbus模块发送给CANBUS硬件，并定期从canbus模块接收关于底盘的相关信息，并将其打包发布。
- 输入 - 1. ControlCommand（控制命令，从control模块获得）
- 输出 - 1. Chassis（汽车底盘信息）, 2. ChassisDetail（汽车底盘信息详细信息）
  Apollo can信号采用DBC格式，消息分为接收和发送，不同消息对应不同ID，对于消息同一类型，接收和发送使用不同的消息ID。具体格式较为复杂。
  Apollo控制模块采用了代理模式，canbus模块采样了工厂模式，使Apollo能够适配不同车辆，但具体如何适配，如何将控制算法应用到实验室的前后轴转向车辆，还有很长的路要研究。

####3.5.1.2 ROS(Autoware)中的control/canbus技术

未研究。

###3.5.2 技术成熟度
Apollo采用的控制算法都是较为实用，并在工程上应用较广泛的算法。算法本身与工程实现虽有难度，但门槛不高，核心点还是在系统建模层面。

###3.5.3技术前沿性
Apollo控制算法稳定可靠，经过实际测试。

###3.5.4源码开放度
未发现Apollo控制模块代码不开放的部分。
###3.5.5指导文档

csdn某博主https://blog.csdn.net/u013914471/article/details/82775091
Apollo各类公开课，公众号资源
GitHub Apollo

###3.5.6可移植性

因为我们目标是对一辆底盘模型不同的新车进行控制。而Apollo的控制模块实现了一整套的底层接口、驱动，无论是向内移植还是向外移植，理论上可行。

前提是:

 - 掌握Apollo的关于`Control`和`Canbus`协议栈，数据内容以及格式，理解输入输出关系
 - 将单转向协议修改或适配为前后转向协议
 - 进行对新车的建模，修改代码中部分参数，掌握标定方法
 - planning模块生成轨迹时需考虑新的车辆动力学模型

###3.5.7总结
Apollo控制算法成熟度高，可靠性高，技术门槛不高，但核心点还是在于对于目标模型的系统建模与工程上的算法实现。

# 4. 平台

## 4.1. 解决方案

### 4.1.1. Apollo

- 倒车入库
- Robo Taxi
- 智能信号灯控制系统
- 以及以下无人驾驶集成方案

#### 4.1.1.1. 新石器无人车

![](Apollo-or-ROS/2020-04-04-22-57-28.png)

#### 4.1.1.2. Apolong

![](Apollo-or-ROS/2020-04-04-23-04-08.png)

<iframe width="560" height="315" src="https://www.youtube.com/embed/5__S3FWD-Vg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
### 4.1.2. Autoware

基本上只在微型小车测试，尚无大型集成方案

![](Apollo-or-ROS/2020-04-04-23-10-07.png)

<iframe width="560" height="315" src="https://www.youtube.com/embed/ojtQw9NPIaU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
## 4.2. 云服务

### 4.2.1. Apollo

Apollo提供有偿的云服务，包括:

- 高精地图
- 仿真平台
- 数据流
- V2X
- 车载平台

### 4.2.2. Autoware

无

## 4.3. 开发套件

### 4.3.1. Apollo

Apollo的`D-Kit`开发套件

### 4.3.2. Autoware

无

## 4.4. 规范认证

不详
