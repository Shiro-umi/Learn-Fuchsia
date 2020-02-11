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

## Component声明
Component declaration是一种机器可读的，对于该当component能做什么以及如何运行的描述。包括metadata（元数据，经过格式化的component信息）。Framework需要metadata去实例化这个component，并将它和其他component组合。  
每个component都有一个declaration。对于那些分布在各个packages中的components，declaration通常通过component manifest file（组件清单文件）的形式。
除了由framework声明，component也可以由resover（解析器）和runner（运行容器？）声明，并同时负责组件的运行。

举个例子来说，一个计算器应用的声明可能会包含以下信息：

- 计算器程序在其包中的位置
- runner的name
- 一个持久化存储的请求，以便再次启动的时候保存累加器状态
- 一个对于使用特定功能来显示用户加冕的请求
- 一个将自己的功能暴露给其他components的请求，以便其他components访问自己的累加寄存器，主要通过IPC

## Component URLs  
Component URLs指定从什么地方检索components declaration, 程序以及assets（拥有的文件或功能？）  

Component可以根据URL方案从多种不同的来源被检索到。  

一些典型的URL方案如下：

- `fuchsia-boot`：从系统启动镜像中解析。这种方案用于在early boot期间且在package system可用之前，检索system operation必要的component。  
  - Example: "fuchsia-boot:///#meta/devcoordinator.cm"（系统协调）
- `fuchsia-pkg`：由Fuchsia包解析器将component解析为app package，这些component可以按需下载并保持最新状态。
  - Example: "fuchsia-pkg://fuchsia.com/netstack#meta/netstack.cm" 
- `http` and `https`：Web解析器将component解析为Web APP，用于将Web内容与framework集成在一起。
  - Example: "https://fuchsia.dev/"  

## Component Topology（组件拓扑）  
Component Topology是一个用于描述component实例之间关系的数据结构。主要包含以下三项：  

- Component instanse tree（实例树）：描述component实例通过何种方式组合（父子关系）
- Capability routing graph（功能路由图）：描述component实例如何获取权限以调用其他component实例暴露在外的功能接口（供给关系）
- Compartment tree（独立关系树）：描述component实例如何独立，以及component实例的sandbox中的哪些资源有可能被分享

>TODO: Add a picture or a thousand words.

Component Topology的结构对component的生命周期和功能权限有很大影响。

### Hierarchical Composition（层级组合）
任何数量的component实例都可以通过hierarchical composition组合为更复杂的component。
在hierarchical composition中，父component创建并声明子component。新创建的子component依赖于父component提供的权限和功能。

子component可通过如下方式创建：

- Statically（静态）：parent在自己的component declaration中声明child，如果parent中的声明被移除，则children自动销毁
- Dynamically（动态）：parent通过realm service（域服务）向自己声明的component collection中添加新的child，并用类似的方法销毁child

以下几点性质需要注意：

- child永远保持对parent的依赖
- 不能被reparent（~~禁止套娃~~），不能对parent进行override
- 当parent被销毁的时候，它持有的所有child均被同时销毁

Component topology将这些父子关系的结构表示为component instanse tree。

>TODO: Add a diagram of a component instance tree.  (~~有完没完？~~)

### Encapsulation（封装）
子component是被封装在parent域中的，child的功能不能从parent域外部直接调用。该封装模型类似于面向对象思想中的封装。

### Realms（域）
Realm是经过hierachical composition格式化的component实例子树。每个realm的根都是一个component，并且包括所有children和descendansts（子孙）节点。
Realms在component topology中是非常重要的encapsulation boundaries（封装边界）。每个realm都拥有明确的权限来影响components的行为，比如：

- 声明功能如何 **进入 / 外出 / 存在** 于一个域内
- 与child components绑定并拥有访问所有children的service的权限
- 创建或销毁子component

### Monikers（别名）
Moniker通过topological path在component tree中对component instance进行标识。Monikers会被系统日志收录，并用于持久化。

详细说明 -> [Monikers](https://fuchsia.dev/fuchsia-src/concepts/components/monikers.md)

### Capability Routing（功能路由）
Components通过capability routiong获取使用其他component暴露于外部的功能。
>TODO: Refactor existing manifests and capabilities to explain the basic concepts here. Draw parallels with constructor dependency injection. Include links to capability types. (~~第三个todo了！~~)

### Compartments（隔离）
Compartment是一个component实例的独立边界。它是维持components的confidentiality（密闭性），integrity（完整性）和availability（可用性）的必要机制。

物理硬件也可以成为一种隔离。运行在同一组硬件上的components共享CPU，内存，存储以及外设。这些components可能会因为旁路，提权和物理攻击，以及运行在其他硬件上的components的威胁变得脆弱。系统安全取决于系统将功能委托给components的有效决策。

[作业（job）](https://fuchsia.dev/fuchsia-src/glossary.md#job)也是一种隔离。在自己的job中运行component可以保证component无法访问到属于其他components或jobs的内存或功能。Framework还可以结束job以终结所有独立component（没有在其他jobs中创建新的进程）进程。

[运行容器（runner）](https://fuchsia.dev/fuchsia-src/concepts/components/runners.md)为每一个当前runner运行的component提供隔离。Runner负责保护自己和所有在其中运行的进程。但是当所有项目共享同一个runtime environment时，该runtime environment会限制内核强制隔离（enforce isolation）的能力。

**隔离空间（Compartments nest）：** runner提供的隔离位于jobs的隔离之中，而jobs隔离又位于硬件隔离之中。这种封装模式表明了每一层隔离的职责：内核负责jobs的隔离，而应用程序则不需要对jobs负责。

一些隔离的强度会弱于其他一些隔离。Job的隔离优先度比runner更大，所以有的时候在不同的job隔离中分别拥有相同runner实例并由优先度更高的隔离来控制的情况也是有意义的。当然在单独的硬件上运行单独的component是最安全的，但也是不可能的。

>TODO: Fill in more details when component framework APIs for assigning components to compartments have been formalized.

### Framework Capabilities
Components通过framework capabilities和自己的运行环境进行交互：

- [Instrumentation Hooks（仪表勾？）](https://fuchsia.dev/fuchsia-src/concepts/components/instrumentation_hooks.md): 诊断和调试component
- [Hub](https://fuchsia.dev/fuchsia-src/concepts/components/hub.md): 在runtime中查询component topology
- [Realm](https://fuchsia.dev/fuchsia-src/concepts/components/realms.md): 管理/绑定子component
- [Lifecycle](https://fuchsia.dev/fuchsia-src/concepts/components/lifecycle.md): 监听并处理lifecycle事件
- [Shutdown](https://fuchsia.dev/fuchsia-src/concepts/components/shutdown.md): 发起有序shutdown
- [Work Scheduler](https://fuchsia.dev/fuchsia-src/concepts/components/work_scheduler.md): 安排可延期工作

### Framework Extensions
Components使用framework extensions将framework和软件生态系统结合起来：

- [Runners](https://fuchsia.dev/fuchsia-src/concepts/components/runners): 将编程语言的runtime与framework整合起来
- [Resolvers](https://fuchsia.dev/fuchsia-src/concepts/components/resolvers): 整合软件交付系统
  

### Framework Development
>TODO: Link to docs about how to build components, diagnostic tools, and debugging features.

## Design Principles（设计原则）

### Accountability（问责）



