title: Atom 服务架构演变
subtitle: Atom 服务端经历了三个版本的迭代，本文着重剖析第三个版本。
cover: https://img30.360buyimg.com/ling/s640x356_jfs/t1/102437/35/18006/125281/5e902d52Ec02fb0f6/e37850516b1f1ebe.png.webp
categories: 项目总结
tags:
  - Atom
  - 全栈开发
  - 服务端
  - 服务
  - 后端
author:
  nick: 右x镇
  github_name: Manjiz
date: 2020-04-21 08:08:08
---

> Atom 是什么？Atom 是集结业内各色资深电商行业设计师，提供一站式专业智能页面和小程序设计服务的平台。经过 2 年紧凑迭代，项目越来越庞大，需求不断变更优化，内部逻辑错综复杂，维护成本急剧拉升。同时，Atom 将要承载的业务越来越多，要向更多的内部用户和商家提供服务，为了适应这些变化，架构升级成为当时紧迫的事项，我们将解构服务端模块，让服务轻量化、模块化，更便捷地拓展业务场景。

Atom 服务端经历了三个版本的迭代，本文着重剖析第三个版本。

## 架构 1.0

这是 Atom 最古老的一个版本，在这一版本中，只规划了频道页的功能，目的是把开发人员从繁复的频道页开发中解放出来，因为功能目的纯粹，所以系统复杂度较低，服务端直接使用了 Koa 框架上手开发，这是一个单体架构的服务，所有的代码都在一个进程中运行。

在部署方面，运用的是非常原始的手工操作：开发人员登入机器，拉取代码后进行类似本地环境的安装启动，然后在不同机器重复这个过程。

另外，Quark 的旧版本使用的是具名组件，具名组件一定程度限制了 Quark 自身的扩展性，这里不作展开。

## 架构 2.0

从频道页搭建平台到多场景页面搭建平台，Atom 用了不到一年时间，更丰富的组件，更多的模板，更多的场景，更多参与进来的设计师，更多的用户，产品开发逐渐专业化，简单的手工运维已经不再适用，于是前端和服务端都进行了一次大换血，服务端用 [Salak](https://github.com/salakjs/salak) 重构，Salak 是个非常好上手的服务端框架，同时为我们带来了接口文档的自动化生成功能，前端和服务端都改为依靠 Talos（一容器式部署内部平台）来部署。服务端逐渐迈入工业时代。

然而，这个阶段仍然没解决粗放的开发方式，缺乏宏观上的规划，日益暴露了以下这些问题：

- **高度集中**

	90% 以上服务集中于一个单体架构中，业务越来越复杂，代码量越来越大，代码的可读性、可维护性和可扩展性下降，开发人员接入成本剧增，业务扩展的代价成指数上升，持续交付能力难以维持。随着用户越来越多，程序承受的并发越来越高，单体架构的应用的并发能力有限。由于系统复杂度的提高，测试的难度也越来越大。

- **耦合度高**

	单体中的各个模块间互相依赖，互相影响，互相掣肘，导致代码重用性低，新功能开发往往由于忌惮耦合逻辑中的隐藏彩蛋，而选择重新编写，这不是我们希望看到的！

- **逻辑混乱**

	除了耦合导致的逻辑混乱，Atom 作为一个从零成长起来的平台，本身就淤积了大量的历史需求，有些是不再使用的，有些是几乎不被使用的，这些代码逻辑给开发人员一个极大的挑战：在进行代码维护的时候不敢轻易改动代码。另外在迭代中需要向下兼容，让服务端有沉重的历史包袱。

- **代码冗余**

	由于框架在前期没有定义好规范标准，在开发过程比较严格遵守代码校验，代码的逻辑、常量等等重复定义，这也同时让项目变得难以维护，比如修改一个常量需要在保证没有遗漏的前提同时修改多处。

## 新架构目标

根据原有架构的优劣，我们设置了本次架构升级的目标：

- **服务模块化**
- **服务通用化**
- **插拔式站点**
- **插拔式场景**
- **标准与规范**

名词解释：

- 站点：即把服务端与平台解耦，从原来的服务即平台，到可以为互相隔离的多个平台提供相同的服务。

![站点](https://storage.360buyimg.com/temporary/200408-sites.png)

- 场景：为应对不同业务类型而设定的概念，不同场景有不同的管理方式和流程等。

## 整体架构

整体架构分为 Web 应用层、接口层、服务层 和 数据层 4 部分，这样拆分能做到入口统一，在部署上的单点部署让发布更加的便捷，独立部署则降低对服务整体的影响：

- Web 应用层：包括 Atom 平台及其他的平台应用
- 接口层：提供网关服务，应用层的请求经由网关作权限控制及请求转发
- 服务层：
	- 服务通信：异步通信使用 MQ，RPC 通信使用 HTTP
	- 业务模块：核心代码，拆解众多小模块应用
	- 基础服务：统一把控用户与权限
	- 服务管理：提升服务的稳定性、健壮性、灵活性
- 数据层：核心数据存储

![整体架构](https://img10.360buyimg.com/ling/jfs/t1/109162/31/7854/93515/5e6215d9Ed92f1af6/cc39d8c1bc9baec9.png)

其中网关作为整个服务端的流量入口，对所有流量进行处理，拦截非法请求，解析登录态并传递到下游，校验接口权限以及超时响应等，统一把控，同时减轻下游的压力。

![网关](https://storage.360buyimg.com/temporary/200420-2-gateway.png)

## 实施

### 计划/筹备/评估

在正式进入升级开发前，小组通过会议探讨架构升级的必要性和可行性，促使我们进行升级的直接原因是平台新增的站点需求和场景需求，如要在原有架构上实现这个需求，势必会在原已混乱的逻辑上增添更多的耦合逻辑，而间接原因，亦即升级必要性，则是要让系统模块化、标准化、通用化，让系统的逻辑更加清晰，提升整个系统的可维护性。

经过我们反复的探讨，对原系统按照功能进行分割，在功能的基础上再按照通用性进行进一步拆分，附加新架构的支撑性工作，评估这些工作的工作量和预计用时，最后对任务进行分配下达。

### 实施

#### 模块化

为什么要模块化？随着平台越做越大，我们想要让各个部分的功能更加独立、明确、清晰，把各部分之间的影响降到最低，对各部分单独运维，避免牵一发而动全身的情况。

这次升级按照功能和通用性把项目划分为 10+ 个模块：如专门负责编译的模块，专门负责模板管理的模块，负责定时任务的模块，作为入口的网关等等。

其中拆分出来若干通用服务，通用服务作为独立于 Atom 系统之外的服务，可以为 Atom 以及其他系统提供服务。

![模块划分](https://storage.360buyimg.com/temporary/200408-modules.png)

对项目进行模块拆解，最为头疼的是斩断关联逻辑，模块的剥离和修复必然会导致一个问题——相同的代码在不同的模块重复出现。为了解决这个问题，我们把部分这些代码放到工具 npm 包中，这些代码包括了：常量、TypeScript 类型定义、权限映射、Mongoose Schema 定义、Salak 插件和工具方法等等。

另一个问题，在原架构中，模块间可以通过代码直接调用，那新架构中如何“还原”这个功能？为了保证解耦度，新架构中仅有少数需要即时调用的功能在模块间通过接口进行直接调用，其他的都是通过 MQ 消息队列和数据库进行互通。

对于 MQ 通信，这里举个例子：编译。服务端编译通常需要的时间比较长，长时间占用连接对服务性能有所影响，而且编译结果并不需要同步响应，对编译模块来说，如果来者不拒，对服务有不小的压力，于是我们决定使用消息队列来完成各个模块之间的通信：

1. 由项目模块通过**接口**直接调用发布模块发起发布操作；
2. 发布模块向消息池推送一条“我要编译”；
3. 编译模块接收到消息后由自身情况判断是否可以进入编译，否则先不予以响应；
4. 编译的各个状态也通过消息推送；
5. 最后项目模块在接收到编译状态的消息后作各种处理。

![编译消息](https://storage.360buyimg.com/temporary/200420-3-jmqbuild.png)

#### 通用化

前面提到在模块化的工作中，我们拆出了 4 个通用的服务模块，通用服务独立于 Atom 系统之外，可以为 Atom 以及其他系统提供服务。模块的通用化是出于两点考虑：

1. 丰富部门的服务，减少重复开发功能
2. 排除 Atom 非核心代码，让系统瘦身

伴随而来的一个问题值得我们思考，如何考量一个功能是否值得抽离通用化？我们应该尽量避免陷入一个误区：系统模块化就是把系统拆得越细越好。如果拆分过细，势必增加运维工作量。在拆分模块的时候，我们考量的是一个模块内的功能是否完整且独立，以及部门或公司对这个通用服务的需求度，真正地做到**低耦合高内聚**。

#### 标准化

代码层面，下面做了个简单的对比：

| 对比项 | 旧架构 | 新架构 |
| --- | --- | --- |
| 主要语言 | JavaScript | TypeScript |
| 代码检测 | 未遵守 | 必需 |
| 接口名称 | 花样百出 | 统一形式 |
| 接口输出 | 百花齐放 | 统一形式 |

TypeScript 的好，前端人都知道，它为我们带来了自动补全、可选的类型系统，使我们能够用上更加新的 JavaScript 特性等等，更多可以参考《[为什么选择 TypeScript](https://rexdainiel.gitbooks.io/typescript/content/docs/why-typescript.html)》。出现后面三点的原因是什么？旧架构经历了从零到一的过程，项目在最初规划欠缺以及中后期没有足够的时间对系统进行修正，时间和需求的变更的双重作用导致代码淤积。

为此，我们在新架构的开发中就强调代码的标准化，对每次提交都要经过代码检测，然后是对五花八门的接口进行统一：

- 接口路径统一：旧架构中，一个列表接口的路径可能是 `/xxx/list`，也可能是 `/xxx/xxxes` 等等，我们在新架构中基于 RESTful API 规则，用资源名词组成的路径和语义化的 HTTP 协议统一接口的定义；
- 参数名统一：比如列表入参中每页数量可能叫 `pageSize` 也可能叫 `count`，于是我们把它统一成一个名字，要求在开发中遵守这个约定；
- 输出统一：在数据输出到前端前对数据进行处理筛选，剔除包括 `_id` 和 `__v` 等无关数据，在输出形式上也做了统一，要求输出中所有的 _id 都替换以 id 的名字出现等等。

![Behavior](https://storage.360buyimg.com/temporary/200420-2-behavior.png)

代码标准化的好处是让代码更加好维护，开发人员很快就能定位到对应的接口代码，对前端而言则减少对接口的识别记忆。

#### 插拔式站点

前面提到，这次架构升级的直接原因是站点需求和场景需求。如果在旧架构下迭代站点需求，只会进一步增加耦合度。为此，我们增加了站点管理模块，在几乎所有的数据项中增加了站点字段，给几乎所有的数据库查询都带上了站点参数。通过这些努力，现在新增站点只需要通过站点模块新增站点，再做一些初始化配置即可完成。

站点概念除对 Atom 功能有了更高要求，也对原来的权限体系形成了新的挑战。在升级前的版本中用户的权限仅有一个集合，要实现每个站点拥有不同的权限只能从两个角度出发：

1. 权限含义拆分（为每个站点分别提供一套独立的权限）
2. 用户权限增加一层抽象（用户的权限改变为多个集合根据站点进行切换）

在比较了两种修改形式后，拆分权限含义虽然在理解上比较容易代码也改动不多。但却大大提升了维护权限表的难度，相当于新增场景就需要增加一套权限，无法做到可插拔。最后在网关层增加了根据用户访问站点
切换权限集合的逻辑。

#### 插拔式场景

场景是站点下面一个纬度，现有活动、频道、心理学测试、SNS、店铺几大场景，如果在旧架构下新增一个场景，需要排期进行开发，而且代码上恐怕也会增加不少针对不同场景的 `if-else`。为了更便捷省心地扩展和维护场景，我们对场景相关的代码从资源管理的角度做了拆解。

ATOM下每个场景拥有的资源主要有 `模板/项目/标签/权限` 四种：

```
标签       页面
 |         |
模板------>项目

权限     
```

首先介绍项目模块目录的结构，项目模块的代码基于 [策略模式](https://en.wikipedia.org/wiki/Strategy_pattern) 组织，每个场景的业务逻辑拆分到单独文件，由调度器直接调用，避免不同场景间逻辑掺杂。

- 调度器文件命名为 `base_资源_service`
- 场景策略文件命名为 `场景小写_资源_service`
- 通用策略文件命名为 `common_资源_service`

当用户查询进来时，调度器根据查询的条件直接调用对应策略文件中的方法（**一般不允许直接调用指定场景的策略除非确认不会关联到其他场景的数据**），当调度器没有没有找到对应场景下的策略时，默认会调用 `common_service` 的逻辑，所以各场景需要继承 `common_service`。以页面管理服务为例，调度器为 `src/service/page` 目录下的 `base_page_service`，通用逻辑为 `common_page_service`，频道页场景逻辑为 `ch_page_service`。

*出于对场景下公有方法的统一抽象，服务中常用的 CRUD 方法接口 放置在 `AbstractServiceClass` 文件中*

```
├── src
│   ├── service
│   │   └── {resource}
│   │       ├── base_{resource}_service 策略文件调用器,controller/mq 直接调用
│   │       ├── common_{resource}_service 通用策略文件，例如列表查询共用的参数处理
│   │       └── {scene}_{resource}_service 场景策略文件，场景特殊的
```

### 部署

#### 数据迁移

鉴于这次升级的巨变，在新旧版本间的切换务必慎重，除了前端与服务端为此做的大量的联调外，我们还对数据进行了兼容性迁移，主要做法是通过迁移脚本把旧数据根据新架构的需要做多重处理，尔后写入新数据库中。

#### 不中断部署

在单体架构中，每一次服务的发布部署都会造成几分钟的空窗。

![不中断部署之前](https://storage.360buyimg.com/temporary/200409-beforenostop.png)

为避免这种情况，在生产环境，我们保证每个模块至少拥有两个容器，在部署的时候，把部分容器从负载均衡摘除，然后循环检测容器是否还有流量，直至没有流量才进行更新操作，服务启动后重新添加到负载均衡，然后对剩下的容器进行同样的操作，这样做的好处是，保证了整个部署过程，服务是不中断的，避免了部署过程中的空档情况。

![不中断部署](https://storage.360buyimg.com/temporary/200409-nostop.png)

#### 运维

为避免再重蹈旧架构下糟糕的运维体验及项目代码管理，我们为新架构梳理了一个运维文档，包括快速接入、开发、调试、部署方方面面的细节都尽可能详尽地记录下来。

![运维文档目录](https://storage.360buyimg.com/temporary/200311-maintenancecategorysx.png)

为系统增加了监控，监控每个接口的性能和可用性。

![方法性能监控](https://storage.360buyimg.com/temporary/200420-ump.png)

## 效果

经过这次升级，基本达成计划中的效果：

- 清晰：逻辑梳理、祛除冗余、TS 重构、ESNext
- 模块化：解耦 10+ 模块，独立运作；HTTP、MQ、数据层等多通信方式
- 标准化：强代码规范；接口统一；响应统一
- 通用化：4+ 通用模块，平台无关；抽取公共库、配置、插件、中间件等
- 易迁移：一键初始化；一键、单点、独立部署；入口统一
- 易扩展：+新增站点拓展能力；调整场景拓展；节省人力时间成本 95%+
- 易维护：追加日志；一键部署；不中断部署
- 易对接：完备的 Joi 文档；详尽的接口变更记录；尽可能的向上兼容

## 工具/方法/协作

工具对项目的顺利进行有非常重要的影响，因此在这次升级中，我们尝试了多种工具。

为了保证项目成员对自己负责模块有清晰的了解以及对模块的改造有明确的图样，团队引入流程图工具用于梳理旧架构的模块并分工，梳理勾画新架构各个模块内部的逻辑等等。

![梳理的内部逻辑图](https://storage.360buyimg.com/temporary/200410-flow.png)

在排期方面，我们实践使用到了甘特图，用甘特图按照模块对任务进行拆分，然后指派给对应的负责人并设置计划的进行时间，每天同步整体的进度，从甘特图可以清晰地了解项目的资源分配与排期，也能看到项目计划与实际的对照，有助于项目整体的进度把控。

![甘特图](https://storage.360buyimg.com/temporary/200413-gantt.png)

甘特图对项目升级的任务进行了初步的划分，对于更细化的划分，我们放到了 [IssueBoard](https://docs.gitlab.com/ee/user/project/issue_board.html)，IssueBoard 像是一个简化版的任务看板，但对我们来说已经绰绰有余了，另外，选择它的理由还包括：它支持跟 git 提交进行联动，适合开发人员使用，可以通过每次提交来关闭相应的 Issue。

![IssueBoard](https://storage.360buyimg.com/temporary/200413-ib.png)

## 总结反思

在这次升级过程中，也暴露了一些不足，主要体现在排期与预期以及在前期的沟通上。

- 排期与预期

	在升级筹划初期的排期过于乐观，而且在升级过程中没有再进行修正，当然这是客观原因造成的，团队要在有限的需求空窗期内完成升级以避免同时维护两个版本，这导致的后果是团队必须每天比计划花更多的时间。

- 沟通

	在服务端进行升级时，没有跟前端沟通具体的细节，而这次升级又是非完全向下兼容的，所以在联调的时候给造成前端一定的困扰和不便。
    
## 参考

- Atom：https://ling.jd.com/atom
- Salak：https://salakjs.github.io/docs/docs/zh-cn/introduction.html
- RESTful API：http://www.ruanyifeng.com/blog/2014/05/restful_api.html