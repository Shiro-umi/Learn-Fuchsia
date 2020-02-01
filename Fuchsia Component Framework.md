# Fuchsia Component Framework  
对framework的简单介绍

## Components 和 Component Framework  
一个**component**是一个单独运行在自己的sandbox中的程序，可以通过**interprocess communication channels**（IPC,进程间通信通道）进行交互。  
**Component Framework**是一个用于部署**component-based software**（组件化软件）的框架。  
该框架几乎负责一切在Fuchsia系统中运行的软件。
下文中**Component Framework**简称为**framework**。  

## 框架的目的
Framework强调**Separation of Concerns**（关注点分离，SoC）。SoC思想的综指是模块化，根据维基百科的描述，通过将信息封装在具有明确意义的接口代码内（信息隐藏）就可实现模块化，或者系统中的分层设计，每一层解决一个不同的问题也可以称为SoC的程序设计。Framework希望developers可以更简单的将program作为component，通过组合来支持更复杂的系统。  
每一个**component**通常只有少数职责。例如以太网驱动程序作为一个component对外公开了硬件接口服务，网络对战组建就可以使用通过这个接口进行网络帧的发送和接收。  
无论**component**由谁创建，所有的**component**都能基于**common set of protocols**（通用协议）协同工作。  

## 优点：
- **可配置性：** 系统行为可以通过对个别component简单的`添加，升级，删除，替换`进行更改。  
- **可扩展性：** 系统可以随意添加component以新增功能。
- **可靠性：** 系统可以通过`重启，停止`等手段迅速从错误中复原。
- **可复用性：** 通用目的的component可以和其他components重新组合并用来解决新的问题。
- **可测试性：** 可以在集成之前对每个component进行单元测试来解决bug。
- **Uniformity（统一性？）** 每个component都使用同样的方式仅负责自己的职能，和他们的实现语言等其他因素均无关系。  


Fuchsia系统本身也是基于component理念构建的，甚至包括驱动程序。

## Everything is a Component (Almost)  
在Fuchsia系统中Component无数不在。所有component被同一机制无缝管理。  

几乎所有在Fuchsia上作为一个component运行的程序都会包含以下内容：

- Command-line tools
- Device drivers
- End-user applications（非Developer应用）
- Filesystems
- Media codecs
- Network stacks
- Tests
- Web pages  

其中仅有几种例外情况：

- Bootloaders
- Device firmware
- kernels
- Bootstrap for the component manager itself
- Virtual machine guest operating systems

## Component是一个密闭的（Hermetic）可组合的独立程序
### Component是程序（Program）

- 是一个可执行程序单元
- 是一个**通过URL定义，loadable，可实例化（instantiated）**的代码段
- 可通过任意语言实现
- 存在一个描述该component它的功能和如何运行

### Component是独立（isolated）程序

- 每个component实例都拥有一个隔离的sandbox
- 根据最小特权原则（Principle of Least Privilege），每个component仅被赋予有限的权限
- 不能获取它固有权限之外的任何权限
- 与其他components的生命周期（lifecycle）无关
- 主要通过IPC与其他components进行通讯
- Component的错误和过失行为不会影响系统

### Component是可组合的（composable）独立程序

- 可以和其他component组合形成更复杂的component
- 可以将其他component的实例通过制定URL的方式添加为自己的子component来实现重用
- 可以通过功能路由（capability routing）向其子组件授予功能
  
### Component是密闭的（Hermetic）可组合独立程序

- Component是封装边界
- 在接口功能不变的情况下内部实现可以随意修改
- Component在分发的时候需要包含所有它的子component需要的libraries

## Component管理器和启动过程
Component Manager是整个framework的核心。它管理所有components的生命周期，提供component需要的功能，并且保持所有components相互独立。  
系统在启动过程早期就已经启动了component manager。Component manager最先启动root component，然后由root component调用component manager启动其他components，包括device manager，filesystems，network stacks和其他必要的服务。  
最终由session framework启动用户界面。  

## Component实例
每一个component instance都运行在自己的sandbox中。  
~~后面说了一堆废话~~

## Component生命周期
Component的生命周期一共有几个重要步骤：create，start，stop和destroy。  
  
### Create
当一个component被实例化的时候，framework会给该实例分配一个独一无二的ID，并添加到component拓扑（Component Topology）中，同时使它的功能可用。
一旦完成create，这个component便可以执行start和destroy。
  
### Start
Load and run。同时为该component需要的功能提供访问权限。Framework只有在需要的时候才会启动组件实例。

### Stop
停止component实例的运行，但会持久化保留其状态，以便重新启动后可以恢复状态。  
以下情况均会导致component实例停止运行： 

- 所有客户端都断开链接
- 父级component停止运行
- 需要更新软件包
- 没有足够的系统资源继续运行该component
- 其他component迫切需要系统资源
- 该component即将被destroy
- 系统关闭

每个component可以实现一个lifecycle handler以便接收即将stop的通知消息。但是在系统资源严重不足，崩溃或者电源故障的情况下组件会在无通知的情况下立刻停止。  
一旦停止，就可以执行重新启动和销毁工作。
  
### Destroy
Destory component实例将永久删除其所有关联状态，并释放其消耗的系统资源。  
一旦销毁，component实例将不复存在并且无法重新启动。仍然可以创建同一component的新实例，但是它们将具有与所有先前实例不同的自己的标识和状态。