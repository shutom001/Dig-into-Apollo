# Dig into Apollo - Routing ![GitHub](https://img.shields.io/github/license/daohu527/Dig-into-Apollo.svg?style=popout)

> 青，取之于蓝而青于蓝；冰，水为之而寒于水。


## Table of Contents
- [Routing模块简介](#introduction)
- [基础知识](#base)
  - [地图](#map)
  - [最短距离](#shortest_path)
  - [Demo](#demo)
- [Routing模块分析](#routing)
  - [Routing类](#routing_class)
  - [Navigator类](#navigator_class)
  

<a name="introduction" />

## Routing模块简介
Routing类似于现在开车时用到的导航模块，通常考虑的是起点到终点的最优路径（通常是最短路径），和Planning的区别是Routing考虑的是起点到终点的最短路径，而Planning则是行驶过程中，当前一小段时间如何行驶，需要考虑当前路况，是否有障碍物。Routing模块则不需要考虑这些信息，只需要做一个长期的规划路径即可，过程如下：  

![introduction](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/introduction.png)  

这也和我们开车类似，上车之后，首先搜索目的地，打开导航（Routing所做的事情），而开始驾车之后，则会根据当前路况，行人车辆信息来适当调整直到到达目的地（Planning所做的事情）。
* **Routing** - 主要关注起点到终点的长期路径，根据起点到终点之间的道路，选择一条最优路径。  
* **Planning** - 主要关注几秒钟之内汽车的行驶路径，根据当前行驶过程中的交通规则，车辆行人等信息，规划一条短期路径。  

<a name="base" />

## 基础知识

<a name="map" />

#### 地图
首先我们以openstreetmap为例来介绍下地图是如何组成的。[开放街道地图](https://www.openstreetmap.org/)（英语：OpenStreetMap，缩写为OSM）是一个建构自由内容之网上地图协作计划，目标是创造一个内容自由且能让所有人编辑的世界地图，并且让一般的移动设备有方便的导航方案。因为这个地图是一个开源地图，所以可以灵活和自由的获取地图资源。
首先我们看下openstreetmap的基本元素：
**Node**![node](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/30px-Osm_element_node.svg.png)  
节点表示由其纬度和经度定义的地球表面上的特定点。每个节点至少包括id号和一对坐标。节点也可用于定义独立点功能。例如，节点可以代表公园长椅或水井。节点也可以定义道路(Way)的形状，节点是一切形状的基础。  
```
<node id="25496583" lat="51.5173639" lon="-0.140043" version="1" changeset="203496" user="80n" uid="1238" visible="true" timestamp="2007-01-28T11:40:26Z">
    <tag k="highway" v="traffic_signals"/>
</node>
```
**Way**![way](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/30px-Osm_element_way.svg.png)![way](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/30px-Osm_element_closedway.svg.png)![way](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/30px-Osm_element_area.svg.png)  
道路是包含2到2,000个有序节点的折线组成，用于表示线性特征，例如河流和道路。道路也可以表示区域（实心多边形）的边界，例如建筑物或森林。在这种情况下，道路的第一个和最后一个节点将是相同的。这被称为“封闭的方式”。  
```
  <way id="5090250" visible="true" timestamp="2009-01-19T19:07:25Z" version="8" changeset="816806" user="Blumpsy" uid="64226">
    <nd ref="822403"/>
    <nd ref="21533912"/>
    <nd ref="821601"/>
    <nd ref="21533910"/>
    <nd ref="135791608"/>
    <nd ref="333725784"/>
    <nd ref="333725781"/>
    <nd ref="333725774"/>
    <nd ref="333725776"/>
    <nd ref="823771"/>
    <tag k="highway" v="residential"/>
    <tag k="name" v="Clipstone Street"/>
    <tag k="oneway" v="yes"/>
  </way>
```
**Relation**![relation](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/30px-Osm_element_relation.svg.png)  
关系是记录两个或更多个数据元素（节点，方式和/或其他关系）之间的关系的多用途数据结构。例子包括：  
* 路线关系，列出形成主要（编号）高速公路，自行车路线或公交路线的方式。
* 转弯限制，表示你无法从一种方式转向另一种方式。
* 描述具有孔的区域（其边界是“外部方式”）的多面体（“内部方式”）。

**Tag**![tag](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/30px-Osm_element_tag.svg.png)  
所有类型的数据元素（节点，方式和关系）以及变更集都可以包含标签。标签描述了它们所附着的特定元素的含义。标签由两个自由格式文本字段组成; 'Key'和'Vaule'。例如，“高速公路”=“住宅”定义了一条道路。元素不能有2个带有相同“key”的标签，“key”必须是唯一的。例如，您不能将元素标记为amenity = restaurant和amenity = bar。  

可以看到我们看到的地图，实际上是由一些Node和Way组成，需要展示地图时候，通过读取地图中的Node和Way的数据实时画(渲染)出来，例如2个Node组成了一条道路，那么就在这两点之间画一条直线，并且标记为道路，如果是封闭区域，并且根据数据，画出一个多边形，并把它标记为湖泊或者公园。  
有很多地图渲染引擎，下面看下openstreet推荐的地图引擎：  
1. Osmarender: 一个基于可扩展样式表语言转换 (XSLT) 的渲染器,能够创建可缩放矢量图形(SVG), SVG可以用浏览器观看或转换成位图.
2. Mapnik: 一个用C++写的非常快的渲染器,可以生成位图(png, jpeg)和矢量图形(pdf, svg, postscript)。


<a name="shortest_path" />

#### 最短距离
我们先看一下经典的例子：最短路径。  
在图论中，最短路径问题是在图中的两个顶点之间找到路径，使得其边的权重之和最小化的问题。而在地图上找到两个点之间最短路径的问题可以被建模为图中最短路径问题的特殊情况，其中顶点对应于交叉点并且边缘对应于路段，每个路段对应于路段的长度。  
![shortest_path](https://github.com/daohu527/Dig-into-Apollo/blob/master/routing/375px-Shortest_path_with_direct_weights.svg.png)  

最短路径算法：  
* Dijkstra算法
* A*算法
* Bellman-Ford算法
* SPFA算法（Bellman-Ford算法的改进版本）
* Floyd-Warshall算法
* Johnson算法
* Bi-Direction BFS算法


在地图上查找两个点之间的最短距离，我们就可以把道路的长度当做边，路口当做节点，通过把道路抽象为一个有向图，然后通过上述算法，查找到当前2点之间的最短路径，并且输出，这就是每次我们查找起点和终点的过程。  
所以要查找起点到终点之间的路径，需要经过以下几个步骤：  
1. 获取地图的原始数据，节点和道路信息。
2. 通过上述信息，构建有向图。
3. 采用最短路径算法，找到2点之间的最短距离。

> 实际上，真实场景的地图导航，查找2点之间的路径，可能不是实时计算出来的，假设有100个人在查询“北京机场”到“天安门”的路线，第一个人的路线可能是实时计算得到的，而其他99个人都是用的缓存的数据，在第一个人查到之后，后面的99个人就不需要重复计算了。  
对于频繁查找的路线，也可以在晚上统一计算，然后保存起来，可以利用存储换速度，除非道路有变化（新修路或者道路维修，桥断了），然后再重新计算。  
多样化的需求，比如可以选择高速优先还是不走高速，地铁优先，少换乘等，这些都需要构建层次和结构化的信息。  
根据用户的反馈实时的更新路况，比如路上有10个人在用地图导航，发现在某一段大家都开的很慢，或者有用户反馈堵车，就更新当前路况。  
所以地图是一个强者越强的市场，用户越多，数据就更新的就越快，地图就越准确，用的人就越多；用的人越少，数据更新的就越慢，假设一条路上只有一个用户，一个人开的慢，并不能反馈当前道路拥堵，用该地图的其它用户过去之后，发现堵车，就导致用户体验很差，下次就不会再用这个地图了。所以地图必须需要一定的用户量才能活下去，而且强者越强。  

<a name="demo" />

#### Demo
下面我们通过"OSM Pathfinding"作为例子，来详细讲解整个过程。[项目地址](https://github.com/daohu527/osm-pathfinding)  
1. 获取地图信息
由于OSM的地图都是开源的，所以我们只需要找到对应的区域，并且选择导出，就可以导出地图的原始数据。地图的数据格式为OSM格式。
2. 构建图
根据上述的信息，构建有向图，下载的格式由于对渲染比较友好，但是对查找最短路径不友好，因此要转换成有向图的格式(apollo的routing模块也是经过了如下的转换)。
3. 查找最短路径 
根据上述的信息，查找一条最短路径。
> 如果图的规模太大，以1000个举例，只算两个点之间互相有连接的情况，1000*1000就是100万个点，如果点的规模更大，那么就需要采用redis数据库来提高查找效率了。

<a name="routing" />

## Routing模块分析
分析Routing模块之前，我们只需要能够解决以下几个问题，就算是把routing模块掌握清楚了。  
1. 如何从A点到B点
2. 如何规避某些点
查找的时候发现是黑名单里的节点，则选择跳过
3. 如何途径某些点
采用分段的形式，逐段导航（改进版的算法是不给定点的顺序，自动规划最优的线路）
4. 如何设置固定线路，而且不会变？
最后routing输出的结果是什么？固定成文件的形式

下面我们开始分析Apollo Routing模块的代码流程。  
首先我们从"routing_component.h"和"routing_component.cc"开始，apollo的功能被划分为各个模块，启动时候由cyber框架根据模块间的依赖顺序加载(每个模块的dag文件定义了依赖顺序)，所以开始查看一个模块时，都是从component文件开始。  
可以看到"RoutingComponent"继承至"cyber::Component"，并且申明为"public"继承方式，"cyber::Component"是一个模板类，它定义了"Initialize"和"Process"方法。而"Proc"为纯虚函数由子类实现。  
```
template <typename M0>
class Component<M0, NullType, NullType, NullType> : public ComponentBase {
 public:
  Component() {}
  ~Component() override {}
  bool Initialize(const ComponentConfig& config) override;
  bool Process(const std::shared_ptr<M0>& msg);

 private:
  virtual bool Proc(const std::shared_ptr<M0>& msg) = 0;
};
```
// todo 模板方法中为虚函数，而继承类中为公有方法？为什么？


```
class RoutingComponent final
    : public ::apollo::cyber::Component<RoutingRequest> {
 public:
  // default用来控制默认构造函数的生成。显式地指示编译器生成该函数的默认版本。
  RoutingComponent() = default;
  ~RoutingComponent() = default;

 public:
  // 初始化 todo 从哪里override?
  bool Init() override;
  // 收到routing request的时候触发执行
  bool Proc(const std::shared_ptr<RoutingRequest>& request) override;

 private:
  // 申明routing请求发布
  std::shared_ptr<::apollo::cyber::Writer<RoutingResponse>> response_writer_ =
      nullptr;
  std::shared_ptr<::apollo::cyber::Writer<RoutingResponse>>
      response_history_writer_ = nullptr;
  // Routing类
  Routing routing_;
  std::shared_ptr<RoutingResponse> response_ = nullptr;
  // 定时器
  std::unique_ptr<::apollo::cyber::Timer> timer_;
  // 锁
  std::mutex mutex_;
};

// 在cyber框架中注册routing模块
CYBER_REGISTER_COMPONENT(RoutingComponent)
```
我们先看下"Init"函数:  
```
bool RoutingComponent::Init() {
  // 设置消息qos，控制流量，创建消息发布response_writer_
  apollo::cyber::proto::RoleAttributes attr;
  attr.set_channel_name(FLAGS_routing_response_topic);
  auto qos = attr.mutable_qos_profile();
  qos->set_history(apollo::cyber::proto::QosHistoryPolicy::HISTORY_KEEP_LAST);
  qos->set_reliability(
      apollo::cyber::proto::QosReliabilityPolicy::RELIABILITY_RELIABLE);
  qos->set_durability(
      apollo::cyber::proto::QosDurabilityPolicy::DURABILITY_TRANSIENT_LOCAL);
  response_writer_ = node_->CreateWriter<RoutingResponse>(attr);

  ...
  // 设置消息qos，创建历史消息发布，和response_writer_类似
  response_history_writer_ = node_->CreateWriter<RoutingResponse>(attr_history);
  
  // todo 启动定时器，发布历史消息，todo，为什么要赋值，并且保证锁？
  std::weak_ptr<RoutingComponent> self =
      std::dynamic_pointer_cast<RoutingComponent>(shared_from_this());
  timer_.reset(new ::apollo::cyber::Timer(
      FLAGS_routing_response_history_interval_ms,
      [self, this]() {
        auto ptr = self.lock();
        if (ptr) {
          std::lock_guard<std::mutex> guard(this->mutex_);
          if (this->response_.get() != nullptr) {
            auto response = *response_;
            auto timestamp = apollo::common::time::Clock::NowInSeconds();
            response.mutable_header()->set_timestamp_sec(timestamp);
            this->response_history_writer_->Write(response);
          }
        }
      },
      false));
  timer_->Start();

  // routing模块初始化和启动是否成功，todo routing_在哪里实例化？
  return routing_.Init().ok() && routing_.Start().ok();
}
```

接下来看"Proc"实现了哪些功能:  
```
bool RoutingComponent::Proc(const std::shared_ptr<RoutingRequest>& request) {
  auto response = std::make_shared<RoutingResponse>();
  // 响应routing_请求
  if (!routing_.Process(request, response.get())) {
    return false;
  }
  // 填充响应头部信息，并且发布
  common::util::FillHeader(node_->Name(), response.get());
  response_writer_->Write(response);
  {
    std::lock_guard<std::mutex> guard(mutex_);
    response_ = std::move(response);
  }
  return true;
}
```
从上面的分析可以看出，"RoutingComponent"模块实现的主要功能:  
1. 实现"Init"和"Proc"函数
2. 接收"RoutingRequest"消息，输出"RoutingResponse"响应。

接下来我们来看routing的具体实现。  

<a name="routing_class" />

#### Routing类
"Routing"类的实现在"routing.h"和"routing.cc"中，首先看下"Routing"类引用的头文件：  
```
#include "modules/common/monitor_log/monitor_log_buffer.h"
#include "modules/common/status/status.h"
#include "modules/map/hdmap/hdmap_util.h"
#include "modules/routing/core/navigator.h"
#include "modules/routing/proto/routing_config.pb.h"
```
看代码之前先看下头文件是个很好的习惯。通过头文件，我们可以知道当前模块的依赖项，从而搞清楚各个模块之间的依赖关系。可以看到"Routing"模块是一个相对比较独立的模块，只依赖于地图。  
Routing类的实现：  
```
class Routing {
 public:
  Routing();
  // 初始化
  apollo::common::Status Init();
  // 启动
  apollo::common::Status Start();
  // 执行
  bool Process(const std::shared_ptr<RoutingRequest> &routing_request,
               RoutingResponse *const routing_response);

 private:
  // 导航
  std::unique_ptr<Navigator> navigator_ptr_;
  // routing模块配置
  RoutingConfig routing_conf_;
  // 高精度地图，用来获取高精度地图信息
  const hdmap::HDMap *hdmap_ = nullptr;
};
```
下面看下"routing.cc"具体的实现：  
Routing模块初始化
```
apollo::common::Status Routing::Init() {
  // 根据规划地图文件(todo 规划地图文件和地图的区别？)，生成导航
  const auto routing_map_file = apollo::hdmap::RoutingMapFile();
  navigator_ptr_.reset(new Navigator(routing_map_file));
  // 加载规划配置
  CHECK(
      cyber::common::GetProtoFromFile(FLAGS_routing_conf_file, &routing_conf_))
      << "Unable to load routing conf file: " + FLAGS_routing_conf_file;
  // 读取地图
  hdmap_ = apollo::hdmap::HDMapUtil::BaseMapPtr();
}
```
Routing模块启动
```
apollo::common::Status Routing::Start() {
  // 导航是否准备好
  if (!navigator_ptr_->IsReady()) {
    return apollo::common::Status(ErrorCode::ROUTING_ERROR,
                                  "Navigator not ready");
  }
  // 规划模块启动成功
  AINFO << "Routing service is ready.";
  monitor_logger_buffer_.INFO("Routing started");
  return apollo::common::Status::OK();
}
```
Routing模块运行
```
bool Routing::Process(const std::shared_ptr<RoutingRequest>& routing_request,
                      RoutingResponse* const routing_response) {
  // 填充缺失Lane信息
  const auto& fixed_request = FillLaneInfoIfMissing(*routing_request);
  // 是否能够找到规划路径
  if (!navigator_ptr_->SearchRoute(fixed_request, routing_response)) {

    monitor_logger_buffer_.WARN("Routing failed! " +
                                routing_response->status().msg());
    return false;
  }
  monitor_logger_buffer_.INFO("Routing success!");
  return true;
}
```
我们再详细看下"FillLaneInfoIfMissing"模块的实现
```
RoutingRequest Routing::FillLaneInfoIfMissing(
    const RoutingRequest& routing_request) {
  RoutingRequest fixed_request(routing_request);
  // 遍历routing请求的点
  for (int i = 0; i < routing_request.waypoint_size(); ++i) {
    const auto& lane_waypoint = routing_request.waypoint(i);
    // lane是否有id
    if (lane_waypoint.has_id()) {
      continue;
    }
    auto point = common::util::MakePointENU(lane_waypoint.pose().x(),
                                            lane_waypoint.pose().y(),
                                            lane_waypoint.pose().z());

    double s = 0.0;
    double l = 0.0;
    hdmap::LaneInfoConstPtr lane;
    // FIXME(all): select one reasonable lane candidate for point=>lane
    // is one to many relationship.
    // 找到当前点最近的lane信息
    if (hdmap_->GetNearestLane(point, &lane, &s, &l) != 0) {
      AERROR << "Failed to find nearest lane from map at position: "
             << point.DebugString();
      return routing_request;
    }
    auto waypoint_info = fixed_request.mutable_waypoint(i);
    waypoint_info->set_id(lane->id().id());
    waypoint_info->set_s(s);
  }
  return fixed_request;
}
```
遍历规划请求的点，简单的规划只有起点和终点，而复杂的规划可以在起点和终点之间设置途经点。如果规划请求中的某一点找不到道路(lane)信息，那么则直接返回，如果找到，则填充缺失信息。而规划的主要流程就是在"navigator_ptr_->SearchRoute()"中。


<a name="navigator_class" />

#### Navigator类
Navigator初始化
```
bool Navigator::Init(const RoutingRequest& request, const TopoGraph* graph,
                     std::vector<const TopoNode*>* const way_nodes,
                     std::vector<double>* const way_s) {
  // 清除上次规划
  Clear();
  // 根据请求获取节点
  if (!GetWayNodes(request, graph_.get(), way_nodes, way_s)) {
    AERROR << "Failed to find search terminal point in graph!";
    return false;
  }
  // 黑名单
  black_list_generator_->GenerateBlackMapFromRequest(request, graph_.get(),
                                                     &topo_range_manager_);
}
```
GetWayNodes查看规划请求的点是否能够在图中找到，如果找到则放入"way_nodes"，并且记录起始点。
```
bool GetWayNodes(const RoutingRequest& request, const TopoGraph* graph,
                 std::vector<const TopoNode*>* const way_nodes,
                 std::vector<double>* const way_s) {
  for (const auto& point : request.waypoint()) {
    const auto* cur_node = graph->GetNode(point.id());
    if (cur_node == nullptr) {
      AERROR << "Cannot find way point in graph! Id: " << point.id();
      return false;
    }
    way_nodes->push_back(cur_node);
    way_s->push_back(point.s());
  }
  return true;
}
```
生成黑名单路段，// todo 即不经过该区域？
```
void BlackListRangeGenerator::GenerateBlackMapFromRequest(
    const RoutingRequest& request, const TopoGraph* graph,
    TopoRangeManager* const range_manager) const {
  AddBlackMapFromLane(request, graph, range_manager);
  AddBlackMapFromRoad(request, graph, range_manager);
  range_manager->SortAndMerge();
}
```

SearchRoute查找规划线路
```
bool Navigator::SearchRoute(const RoutingRequest& request,
                            RoutingResponse* const response) {
  ...
  // 初始化规划点和起点
  std::vector<const TopoNode*> way_nodes;
  std::vector<double> way_s;
  if (!Init(request, graph_.get(), &way_nodes, &way_s)) {
    return false;
  }
  // 根据节点和起点，查找返回结果，注意这里返回的是一段范围
  std::vector<NodeWithRange> result_nodes;
  if (!SearchRouteByStrategy(graph_.get(), way_nodes, way_s, &result_nodes)) {
    return false;
  }
  if (result_nodes.empty()) {
    return false;
  }
  // 插入起点和终点
  result_nodes.front().SetStartS(request.waypoint().begin()->s());
  result_nodes.back().SetEndS(request.waypoint().rbegin()->s());
  // 生成通道区域
  if (!result_generator_->GeneratePassageRegion(
          graph_->MapVersion(), request, result_nodes, topo_range_manager_,
          response)) {
    return false;
  }
  ...
}
```
接着看"SearchRouteByStrategy"如何处理
```
bool Navigator::SearchRouteByStrategy(
    const TopoGraph* graph, const std::vector<const TopoNode*>& way_nodes,
    const std::vector<double>& way_s,
    std::vector<NodeWithRange>* const result_nodes) const {
  std::unique_ptr<Strategy> strategy_ptr;
  // 设置Astar策略来查找规划路径
  strategy_ptr.reset(new AStarStrategy(FLAGS_enable_change_lane_in_result));

  result_nodes->clear();
  std::vector<NodeWithRange> node_vec;
  for (size_t i = 1; i < way_nodes.size(); ++i) {
    const auto* way_start = way_nodes[i - 1];
    const auto* way_end = way_nodes[i];
    double way_start_s = way_s[i - 1];
    double way_end_s = way_s[i];

    TopoRangeManager full_range_manager = topo_range_manager_;
    black_list_generator_->AddBlackMapFromTerminal(
        way_start, way_end, way_start_s, way_end_s, &full_range_manager);
    // 获取起点
    SubTopoGraph sub_graph(full_range_manager.RangeMap());
    const auto* start = sub_graph.GetSubNodeWithS(way_start, way_start_s);
    if (start == nullptr) {
      return false;
    }
    // 获取终点
    const auto* end = sub_graph.GetSubNodeWithS(way_end, way_end_s);
    if (end == nullptr) {
      return false;
    }
    // 通过策略器(Astar)查找结果
    std::vector<NodeWithRange> cur_result_nodes;
    if (!strategy_ptr->Search(graph, &sub_graph, start, end,
                              &cur_result_nodes)) {
      return false;
    }
    // 插入节点
    node_vec.insert(node_vec.end(), cur_result_nodes.begin(),
                    cur_result_nodes.end());
  }

  // 合并Route
  if (!MergeRoute(node_vec, result_nodes)) {
    return false;
  }
  return true;
}
```
#### 建图
apollo的建图在"routing/topo_creator"中，通过下图可以看下建图的流程。  
通过读取base_map_file_path_为pbmap_来建图。  
![topo_creator]()  
而Grap图中数据结构在"routing/graph"中。
topo_graph.cc主要是存储了点和边的信息
topo_node.cc 点和边的数据结构
sub_topo_graph.cc blackmap的点和边数据结构


#### 最短路径
apollo的最短路径算法采用的是Astar算法，代码实现在"routing/strategy"目录。可以看到strategy中实现了一个"Strategy"的基类，也就是说后面可以扩展其他的路径策略。  
```
class Strategy {
 public:
  virtual ~Strategy() {}
  // 在派生类中实现查找方法
  virtual bool Search(const TopoGraph* graph, const SubTopoGraph* sub_graph,
                      const TopoNode* src_node, const TopoNode* dest_node,
                      std::vector<NodeWithRange>* const result_nodes) = 0;
};
```
下面看"AStarStrategy"的"Search"方法：  
```
bool AStarStrategy::Search(const TopoGraph* graph,
                           const SubTopoGraph* sub_graph,
                           const TopoNode* src_node, const TopoNode* dest_node,
                           std::vector<NodeWithRange>* const result_nodes) {

  std::priority_queue<SearchNode> open_set_detail;

  SearchNode src_search_node(src_node);
  src_search_node.f = HeuristicCost(src_node, dest_node);
  open_set_detail.push(src_search_node);

  open_set_.insert(src_node);
  g_score_[src_node] = 0.0;
  enter_s_[src_node] = src_node->StartS();

  SearchNode current_node;
  std::unordered_set<const TopoEdge*> next_edge_set;
  std::unordered_set<const TopoEdge*> sub_edge_set;
  while (!open_set_detail.empty()) {
    current_node = open_set_detail.top();
    const auto* from_node = current_node.topo_node;
    if (current_node.topo_node == dest_node) {
      if (!Reconstruct(came_from_, from_node, result_nodes)) {
        AERROR << "Failed to reconstruct route.";
        return false;
      }
      return true;
    }
    open_set_.erase(from_node);
    open_set_detail.pop();

    if (closed_set_.count(from_node) != 0) {
      // if showed before, just skip...
      continue;
    }
    closed_set_.emplace(from_node);

    // if residual_s is less than FLAGS_min_length_for_lane_change, only move
    // forward
    const auto& neighbor_edges =
        (GetResidualS(from_node) > FLAGS_min_length_for_lane_change &&
         change_lane_enabled_)
            ? from_node->OutToAllEdge()
            : from_node->OutToSucEdge();
    double tentative_g_score = 0.0;
    next_edge_set.clear();
    for (const auto* edge : neighbor_edges) {
      sub_edge_set.clear();
      sub_graph->GetSubInEdgesIntoSubGraph(edge, &sub_edge_set);
      next_edge_set.insert(sub_edge_set.begin(), sub_edge_set.end());
    }

    for (const auto* edge : next_edge_set) {
      const auto* to_node = edge->ToNode();
      if (closed_set_.count(to_node) == 1) {
        continue;
      }
      if (GetResidualS(edge, to_node) < FLAGS_min_length_for_lane_change) {
        continue;
      }
      tentative_g_score =
          g_score_[current_node.topo_node] + GetCostToNeighbor(edge);
      if (edge->Type() != TopoEdgeType::TET_FORWARD) {
        tentative_g_score -=
            (edge->FromNode()->Cost() + edge->ToNode()->Cost()) / 2;
      }
      double f = tentative_g_score + HeuristicCost(to_node, dest_node);
      if (open_set_.count(to_node) != 0 && f >= g_score_[to_node]) {
        continue;
      }
      // if to_node is reached by forward, reset enter_s to start_s
      if (edge->Type() == TopoEdgeType::TET_FORWARD) {
        enter_s_[to_node] = to_node->StartS();
      } else {
        // else, add enter_s with FLAGS_min_length_for_lane_change
        double to_node_enter_s =
            (enter_s_[from_node] + FLAGS_min_length_for_lane_change) /
            from_node->Length() * to_node->Length();
        // enter s could be larger than end_s but should be less than length
        to_node_enter_s = std::min(to_node_enter_s, to_node->Length());
        // if enter_s is larger than end_s and to_node is dest_node
        if (to_node_enter_s > to_node->EndS() && to_node == dest_node) {
          continue;
        }
        enter_s_[to_node] = to_node_enter_s;
      }

      g_score_[to_node] = f;
      SearchNode next_node(to_node);
      next_node.f = f;
      open_set_detail.push(next_node);
      came_from_[to_node] = from_node;
      if (open_set_.count(to_node) == 0) {
        open_set_.insert(to_node);
      }
    }
  }

}
```



## 需求分析
1. 如何从A点到B点
2. 如何规避某些点
3. 如何途径某些点
4. 如何设置固定线路，而且不会变？

## routing for osm
https://wiki.openstreetmap.org/wiki/Routing

http://www.patrickklose.com/posts/parsing-osm-data-with-python/

城市道路分析：
https://geoffboeing.com/2016/11/osmnx-python-street-networks/
https://automating-gis-processes.github.io/2018/notebooks/L6/network-analysis.html

https://socialhub.technion.ac.il/wp-content/uploads/2017/08/revise_version-final.pdf

https://stackoverflow.com/questions/29639968/shortest-path-using-openstreetmap-datanodes-and-ways


## openstreetmap 查找节点
If it's a polygon, then it's a closed way in the OSM database. You can find ways by id as simple as this: http://www.openstreetmap.org/way/305293190

If a specific node (the building blocks of ways) is giving a problem, the link would be http://www.openstreetmap.org/node/305293190 .

If it is a multipolygon (for example a building with a hole in it), the link would be http://www.openstreetmap.org/relation/305293190


## 地图介绍
https://blog.csdn.net/scy411082514/article/details/7484497

## 地图下载
https://www.openstreetmap.org/export#map=15/22.5163/113.9380


## Reference
[OpenstreetMap](https://www.openstreetmap.org/)  
[OpenstreetMap Elements](https://wiki.openstreetmap.org/wiki/Elements)  
[OpenstreetMap地图渲染](https://wiki.openstreetmap.org/wiki/Zh-hans:Beginners_Guide_1.5)  
[最短路径问题](https://en.wikipedia.org/wiki/Shortest_path_problem)  
