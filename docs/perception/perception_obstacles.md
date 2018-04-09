# Apollo 2.0 感知模块：障碍物感知阅读笔记

本文档结合代码详细地解释感知模块中障碍物感知的流程与功能，也可以官方参考文档([Apollo 2.0 Obstacles Perception](https://github.com/ApolloAuto/apollo/blob/master/docs/specs/3d_obstacle_perception_cn.md)

***初次接触三维重建，如有笔误，欢迎指正!***

## 1. 障碍物感知: 3D Obstacles Perception

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/perception_obstacles_framework.png)

上图为子节点之间的关系与数据流动，障碍物感知共存在三个子节点(线程)，分别为：

- 激光雷达处理子节点 LidarProcessSubnode
- 雷达处理子节点 RadarProcessSubnode
- 融合子节点 FusionSubnode

每个子节点的输入数据与输出数据在边上标出。

**(1) 激光雷达子节点**

LidarProcessSubnode::OnPointCloud以ROS消息订阅与发布机制触发回调函数，处理结果保存在LidarObjectData共享数据容器中。主要解决的问题有：

- 高精地图ROI过滤器(HDMap ROI Filter)
- 基于卷积神经网络分割(CNN Segmentation)
- MinBox 障碍物边框构建(MinBox Builder)
- HM对象跟踪(HM Object Tracker)

**(2) 雷达子节点**

RadarProcessSubnode::OnRadar同样以ROS消息订阅与发布机制触发回调函数，处理结果保存在RadarObjectData共享数据容器中。

**(3) 融合子节点**

FusionSubnode::ProcEvents以自定义ProcEvents+EventManeger消息处理机制，从LidarObjectData和RadarObjectData共享数据容器中提取数据，融合并存储在FusionObjectData共享容器中。

## 2. 激光雷达感知 Perception: Lidar Obstacles Perception




### 2.3  MinBox障碍物边框构建

>对象构建器组件为检测到的障碍物建立一个边界框。因为LiDAR传感器的遮挡或距离，形成障碍物的点云可以是稀疏的，并且仅覆盖一部分表面。因此，盒构建器将恢复给定多边形点的完整边界框。即使点云稀疏，边界框的主要目的还是预估障碍物（例如，车辆）的方向。同样地，边框也用于可视化障碍物。
>
>算法背后的想法是找到给定多边形点边缘的所有区域。在以下示例中，如果AB是边缘，则Apollo将其他多边形点投影到AB上，并建立具有最大距离的交点对，这是属于边框的边缘之一。然后直接获得边界框的另一边。通过迭代多边形中的所有边，在以下图4所示，Apollo确定了一个6边界边框，将选择具有最小面积的方案作为最终的边界框。

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/object_building.png)

这部分代码看了比较久，有些地方一直没想明白，直到推理了很久才找到了一种合适的说法，下面我们依旧从代码入手，一步步解析障碍物边框构建的流程。

上一步CNN分割与后期处理，可以得到lidar一定区域内的障碍物集群。接下去我们将对这些障碍物集群建立其标定框。标定框的作用除了标识物体，还有一个作用就是标记障碍物的长length，宽width，高height。其中规定长length大于宽width，障碍物方向就是长的方向direction。MinBox构建过程如下：

- 计算障碍物2d投影(高空鸟瞰xy平面)下的多边形polygon(如下图B)
- 根据上述多边形，计算最适边框(如下图C)

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/minbox_framework.png)

大致的代码框架如下：

```c++
/// file in apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
  /// call hdmap to get ROI
  ...
  /// call roi_filter
  ...
  /// call segmentor
  ...
  /// call object builder
  if (object_builder_ != nullptr) {
    ObjectBuilderOptions object_builder_options;
    if (!object_builder_->Build(object_builder_options, &objects)) {
      ...
    }
  }
}

/// file in apollo/modules/perception/obstacle/lidar/object_builder/min_box/min_box.cc
bool MinBoxObjectBuilder::Build(const ObjectBuilderOptions& options, std::vector<ObjectPtr>* objects) {
  for (size_t i = 0; i < objects->size(); ++i) {
    if ((*objects)[i]) {
      BuildObject(options, (*objects)[i]);
    }
  }
}
void MinBoxObjectBuilder::BuildObject(ObjectBuilderOptions options, ObjectPtr object) {
  ComputeGeometricFeature(options.ref_center, object);
}
void MinBoxObjectBuilder::ComputeGeometricFeature(const Eigen::Vector3d& ref_ct, ObjectPtr obj) {
  // step 1: compute 2D xy plane's polygen
  ComputePolygon2dxy(obj);
  // step 2: construct box
  ReconstructPolygon(ref_ct, obj);
}
```

上述是MinBox障碍物边框构建的主题框架代码，构建的两个过程分别在`ComputePolygon2dxy`和`ReconstructPolygon`函数完成，下面篇幅我们就具体深入这两个函数，详细了解一下Apollo对障碍物构建的一个流程，和其中一些令人费解的代码段。

#### 2.3.1 MinBox构建--计算2DXY平面投影

这个阶段主要作用是障碍物集群做XY平面下的凸包多边形计算，最终得到这个多边形的一些角点。第一部分相对比较简单，没什么难点，计算凸包是调用PCL库的`ConvexHull`组件(具体请参考[pcl::ConvexHull](http://docs.pointclouds.org/trunk/classpcl_1_1_convex_hull.html))。下面是Apollo的凸包计算代码：

```c++
/// file in apollo/modules/perception/obstacle/lidar/object_builder/min_box/min_box.cc
void MinBoxObjectBuilder::ComputePolygon2dxy(ObjectPtr obj) {
  ...
  ConvexHull2DXY<pcl_util::Point> hull;
  hull.setInputCloud(pcd_xy);
  hull.setDimension(2);
  std::vector<pcl::Vertices> poly_vt;
  PointCloudPtr plane_hull(new PointCloud);
  hull.Reconstruct2dxy(plane_hull, &poly_vt);

  if (poly_vt.size() == 1u) {
    std::vector<int> ind(poly_vt[0].vertices.begin(), poly_vt[0].vertices.end());
    TransformPointCloud(plane_hull, ind, &obj->polygon);
  } else {
    ...
  }
}

/// file in apollo/modules/perception/common/convex_hullxy.h
template <typename PointInT>
class ConvexHull2DXY : public pcl::ConvexHull<PointInT> {
public:
  void Reconstruct2dxy(PointCloudPtr hull, std::vector<pcl::Vertices> *polygons) {
    PerformReconstruction2dxy(hull, polygons, true);
  }

  void PerformReconstruction2dxy(PointCloudPtr hull, std::vector<pcl::Vertices> *polygons, bool fill_polygon_data = false) {  
    coordT *points = reinterpret_cast<coordT *>(calloc(indices_->size() * dimension, sizeof(coordT)));
    // step1. Build input data, using appropriate projection
    int j = 0;
    for (size_t i = 0; i < indices_->size(); ++i, j += dimension) {
      points[j + 0] = static_cast<coordT>(input_->points[(*indices_)[i]].x);
      points[j + 1] = static_cast<coordT>(input_->points[(*indices_)[i]].y);
    }
    // step2. Compute convex hull
    int exitcode = qh_new_qhull(dimension, static_cast<int>(indices_->size()), points, ismalloc, const_cast<char *>(flags), outfile, errfile);
    std::vector<std::pair<int, Eigen::Vector4f>, Eigen::aligned_allocator<std::pair<int, Eigen::Vector4f>>> idx_points(num_vertices);
    FORALLvertices {
      hull->points[i] = input_->points[(*indices_)[qh_pointid(vertex->point)]];
      idx_points[i].first = qh_pointid(vertex->point);
      ++i;
    }
    // step3. Sort
    Eigen::Vector4f centroid;
    pcl::compute3DCentroid(*hull, centroid);
    for (size_t j = 0; j < hull->points.size(); j++) {
      idx_points[j].second[0] = hull->points[j].x - centroid[0];
      idx_points[j].second[1] = hull->points[j].y - centroid[1];
    }
    std::sort(idx_points.begin(), idx_points.end(), pcl::comparePoints2D);
    polygons->resize(1);
    (*polygons)[0].vertices.resize(hull->points.size());
    for (int j = 0; j < static_cast<int>(hull->points.size()); j++) {
      hull->points[j] = input_->points[(*indices_)[idx_points[j].first]];
      (*polygons)[0].vertices[j] = static_cast<unsigned int>(j);
    }
  }
}
```

从上面代码的注释我们可以很清楚的了解到这个多边形顶点的求解流程，具体函数由`PerformReconstruction2dxy`函数完成，这个函数其实跟PCL库自带的很像[pcl::ConvexHull<PointInT>::performReconstruction2D/Line76](http://docs.pointclouds.org/trunk/convex__hull_8hpp_source.html)，其实Apollo开发人员几乎将pcl库的`performReconstruction2D`原封不动的搬过来了，去掉了一些冗余额外的信息。这个过程主要有：

1. 构建输入数据，将输入的点云复制到coordT \*points做处理
2. 计算障碍物点云的凸包，得到的结果是多边形顶点。调用`qh_new_qhull`函数
3. 顶点排序，从[pcl::comparePoints2D/Line59](http://docs.pointclouds.org/trunk/surface_2include_2pcl_2surface_2convex__hull_8h_source.html)可以看到排序是角度越大越靠前，atan2函数的结果是[-pi,pi]。所以就相当于是顺时针对顶点进行排序。

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/minbox_polygons.png)

这个过程只要自己稍加关注一点就可以了解他的原理和过程，这里不再过度解释每个模块。上图是计算多边形交点的流程示意图。

#### 2.3.2 MinBox构建--边框构建

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/minbox_box.png)

边框构建的大致思想是对过程中1得到的多边形的每一条边，将剩下的所有点都投影到这条边上可以计算边框Box的一条边长度(最远的两个投影点距离)，同时选择距离该条边最远的点计算Box的高，这样就可以得到一个Box(上图case1-7分别是以这个多边形7条边作投影得到的7个Box)，最终选择Box面积最小的边框作为障碍物的边框。上图中case7得到的Box面积最小，所以case7中的Box就是最终障碍物的边框。当边框确定以后，就可以得到障碍物的长度length(大边长)，宽度(小边长)，方向(大边上对应的方向)，高度（点云的平均高度，CNN分割与后期处理阶段得到）。

但是实际这个过程有部分代码块是比较难理解的，而且加入了很多实际问题来优化这个过程。这里我将对这些问题一一进行解释，希望能够让大家理解。根据代码我简单地将这个过程归结为3步：

1. 投影边长的选择(为什么要选择？因为背对lidar那一侧的点云是稀疏的，那一侧的多边形顶点是不可靠的，不用来计算Box)
2. 每个投影边长计算Box

在进入正式的代码详解以前，这里有几个知识点需要我们了解。

假设向量a=(x0,y0)，向量b=(x1,y1)，那么有
- 两个向量的点乘, a·b = x0x1 + y0y1\
- 计算向量a在向量b上的投影: v = a·b/(b^2)·b，投影点的坐标就是v+(b.x, b.y)
- 两个向量的叉乘, axb = |a|·|b|sin(theta) = x0y1 - x1y0，叉乘方向与ab平面垂直，遵循右手法则。**叉乘模大小另一层意义是: ab向量构成的平行四边形面积**

**如果两个向量a，b共起点，那么axb小于0，那么a to b的逆时针夹角大于180度；等于则共线；大于0，a to b的逆时针方向夹角小于180度。**

接下来我们就正式的解剖`ReconstructPolygon`Box构建的代码

(1) Step1：投影边长的选择

```c++
/// file in apollo/modules/perception/obstacle/lidar/object_builder/min_box/min_box.cc
void MinBoxObjectBuilder::ReconstructPolygon(const Eigen::Vector3d& ref_ct, ObjectPtr obj) {
  // compute max_point and min_point
  size_t max_point_index = 0;
  size_t min_point_index = 0;
  Eigen::Vector3d p;
  p[0] = obj->polygon.points[0].x;
  p[1] = obj->polygon.points[0].y;
  p[2] = obj->polygon.points[0].z;
  Eigen::Vector3d max_point = p - ref_ct;
  Eigen::Vector3d min_point = p - ref_ct;
  for (size_t i = 1; i < obj->polygon.points.size(); ++i) {
    Eigen::Vector3d p;
    p[0] = obj->polygon.points[i].x;
    p[1] = obj->polygon.points[i].y;
    p[2] = obj->polygon.points[i].z;
    Eigen::Vector3d ray = p - ref_ct;
    // clock direction
    if (max_point[0] * ray[1] - ray[0] * max_point[1] < EPSILON) {
      max_point = ray;
      max_point_index = i;
    }
    // unclock direction
    if (min_point[0] * ray[1] - ray[0] * min_point[1] > EPSILON) {
      min_point = ray;
      min_point_index = i;
    }
  }
}
```

首先我们看到这一段代码，第一眼看过去是计算`min_point`和`max_point`两个角点，那么这个角点到底是什么意思呢？里面这个关于`EPSILON`的比较条件代表了什么，下面我们图。有一个前提我们已经在polygons多边形角点计算中可知：obj的polygon中所有角点都是顺时针按照arctan角度由大到小排序。那么这个过程我们可以从下面的图中了解到作用：

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/minbox_maxminpt.png)

图中叉乘与0(EPSILON)的大小就是根据前面提到的，两个向量的逆时针夹角。从上图我们可以很清晰的看到：**`max_point`和`min_point`就代表了lidar能检测到障碍物的两个极端点！**


```c++
/// file in apollo/modules/perception/obstacle/lidar/object_builder/min_box/min_box.cc
void MinBoxObjectBuilder::ReconstructPolygon(const Eigen::Vector3d& ref_ct, ObjectPtr obj) {
  // compute max_point and min_point
  ...
  // compute valid edge
  Eigen::Vector3d line = max_point - min_point;
  double total_len = 0;
  double max_dis = 0;
  bool has_out = false;
  for (size_t i = min_point_index, count = 0; count < obj->polygon.points.size(); i = (i + 1) % obj->polygon.points.size(), ++count) {
    //Eigen::Vector3d p_x = obj->polygon.point[i]
    size_t j = (i + 1) % obj->polygon.points.size();
    if (j != min_point_index && j != max_point_index) {
      // Eigen::Vector3d p = obj->polygon.points[j];
      Eigen::Vector3d ray = p - min_point;
      if (line[0] * ray[1] - ray[0] * line[1] < EPSILON) {
        ...
      }else {
        ...
      }
    } else if ((i == min_point_index && j == max_point_index) || (i == max_point_index && j == min_point_index)) {
      ...
    } else if (j == min_point_index || j == max_point_index) {
      // Eigen::Vector3d p = obj->polygon.points[j];
      Eigen::Vector3d ray = p_x - min_point;
      if (line[0] * ray[1] - ray[0] * line[1] < EPSILON) {
        ...
      } else {
        ...
      }
    }
  }
}
```

当计算得到`max_point`和`min_point`后就需要执行这段代码，这段代码是比较难理解的，为什么需要对每条边做一个条件筛选？请看下图

![img](https://github.com/YannZyl/Apollo-Note/blob/master/images/perception_obstacles/minbox_edge_selection.png)

**上图中A演示了这段代码对一个汽车的点云多边形进行处理，最后的处理结果可以看到只有Edge45、Edge56、Edge67是有效的，最终会被计入`total_len`和`max_dist`。而且你会发现这些边都是在`line(max_point-min_point)`这条分界线的一侧，而且是靠近lidar这一侧。说明了靠近lidar这一侧点云检测效果好，边稳定；而背离lidar那一侧，会因为遮挡原因，往往很难(有时候不可能)得到真正的顶点，如上图B所示。**

经过这么分析，其实上述的代码理解起来还是比较能接受的，希望能帮到你。

(2) Step2：投影边长Box计算

投影边长Box计算由`ComputeAreaAlongOneEdge`函数完成，分析这个函数的代码：

```c++
/// file in apollo/modules/perception/obstacle/lidar/object_builder/min_box/min_box.cc
double MinBoxObjectBuilder::ComputeAreaAlongOneEdge(
    ObjectPtr obj, size_t first_in_point, Eigen::Vector3d* center,
    double* lenth, double* width, Eigen::Vector3d* dir) {
  // first for
  std::vector<Eigen::Vector3d> ns;
  Eigen::Vector3d v(0.0, 0.0, 0.0);      // 记录以(first_in_point,first_in_point+1)两个定点为边，所有点投影，距离这条边最远的那个点
  Eigen::Vector3d vn(0.0, 0.0, 0.0);     // 最远的点在(first_in_point,first_in_point+1)这条边上的投影坐标
  Eigen::Vector3d n(0.0, 0.0, 0.0);      // 用于临时存储
  double len = 0;
  double wid = 0;
  size_t index = (first_in_point + 1) % obj->polygon.points.size();
  for (size_t i = 0; i < obj->polygon.points.size(); ++i) {
    if (i != first_in_point && i != index) {
      // o = obj->polygon.points[i]
      // a = obj->polygon.points[first_in_point]
      // b = obj->polygon.points[first_in_point+1]
      // 计算向量ao在ab向量上的投影，根据公式:k = ao·ab/(ab^2), 计算投影点坐标，根据公式k·ab+(ab.x, ab.y)
      double k =  ((a[0] - o[0]) * (b[0] - a[0]) + (a[1] - o[1]) * (b[1] - a[1]));
      k = k / ((b[0] - a[0]) * (b[0] - a[0]) + (b[1] - a[1]) * (b[1] - a[1]));
      k = k * -1;
      n[0] = (b[0] - a[0]) * k + a[0];
      n[1] = (b[1] - a[1]) * k + a[1];
      n[2] = 0;
      // 计算由ab作为边，o作为顶点的平行四边形的面积,利用公式|ao x ab|，叉乘的模就是四边形的面积，
      Eigen::Vector3d edge1 = o - b;
      Eigen::Vector3d edge2 = a - b;
      double height = fabs(edge1[0] * edge2[1] - edge2[0] * edge1[1]);
      // 利用公式： 面积/length(ab)就是ab边上的高，即o到ab边的垂直距离， 记录最大的高
      height = height / sqrt(edge2[0] * edge2[0] + edge2[1] * edge2[1]);
      if (height > wid) {
        wid = height;
        v = o;
        vn = n;
      }
    } else {
      ...
    }
    ns.push_back(n);
  }
}
```

从上面的部分代码可以看得出，`ComputeAreaAlongOneEdge`函数接受的输入包括多边形顶点集合，起始边`first_in_point`。代码将以`first_in_point`和`first_in_point+1`两个顶点构建一条边，将集合中其他点都投影到这条边上，并计算顶点距离这条边的高，也就是垂直距离。最终的结果保存到`ns`中。代码中`k`的计算利用了两个向量点乘来计算投影点的性质；`height`的计算利用了两个向量叉乘的模等于两个向量组成的四边形面积的性质。

```c++
/// file in apollo/modules/perception/obstacle/lidar/object_builder/min_box/min_box.cc
double MinBoxObjectBuilder::ComputeAreaAlongOneEdge(
  // first for
  ...
  // second for
  size_t point_num1 = 0;
  size_t point_num2 = 0;
  // 遍历first_in_point和first_in_point+1两个点以外的，其他点的投影高，选择height最大的点，来一起组成Box
  // 这两个for循环是寻找ab边上相聚最远的投影点，因为要把所有点都包括到Box中，所以Box沿着ab边的边长就是最远两个点的距离，可以参考边框构建。
  for (size_t i = 0; i < ns.size() - 1; ++i) {
    Eigen::Vector3d p1 = ns[i];
    for (size_t j = i + 1; j < ns.size(); ++j) {
      Eigen::Vector3d p2 = ns[j];
      double dist = sqrt((p1[0] - p2[0]) * (p1[0] - p2[0]) + (p1[1] - p2[1]) * (p1[1] - p2[1]));
      if (dist > len) {
        len = dist;
        point_num1 = i;
        point_num2 = j;
      }
    }
  }
  // vp1和vp2代表了Box的ab边对边的那条边的两个顶点，分别在v的两侧，方向和ab方向一致。
  Eigen::Vector3d vp1 = v + ns[point_num1] - vn;
  Eigen::Vector3d vp2 = v + ns[point_num2] - vn;
  // 计算中心点和面积
  (*center) = (vp1 + vp2 + ns[point_num1] + ns[point_num2]) / 4;
  (*center)[2] = obj->polygon.points[0].z;
  if (len > wid) {
    *dir = ns[point_num2] - ns[point_num1];
  } else {
    *dir = vp1 - ns[point_num1];
  }
  *lenth = len > wid ? len : wid;
  *width = len > wid ? wid : len;
  return (*lenth) * (*width);
}
```

剩下的代码就是计算Box的四个顶点坐标，以及他的面积Area。

综上所述，Box经过上述(1)(2)两个阶段，可以很清晰的得到每条有效边(靠近lidar一侧，在`min_point`和`max_point`之间)对应的Box四个顶点坐标、宽、高。最终选择Box面积最小的作为障碍物预测Box。这个过程的代码部分在理解上存在一定难度，经过本节的讲解，应该做MinBox边框构建有了一定的了解。

### 2.4 HM对象跟踪

>HM对象跟踪器跟踪分段检测到的障碍物。通常，它通过将当前检测与现有跟踪列表相关联，来形成和更新跟踪列表，如不再存在，则删除旧的跟踪列表，并在识别出新的检测时生成新的跟踪列表。 更新后的跟踪列表的运动状态将在关联后进行估计。 在HM对象跟踪器中，匈牙利算法(Hungarian algorithm)用于检测到跟踪关联，并采用鲁棒卡尔曼滤波器(Robust Kalman Filter) 进行运动估计。

上述是Apollo官方文档对HM对象跟踪的描述，这部分意思比较明了，主要的跟踪流程可以分为:

- 预处理。(lidar->local ENU坐标系变换、跟踪对象创建、跟踪目标保存)
- 卡尔曼滤波器滤波，预测物体当前位置与速度(卡尔曼滤波阶段1：Predict阶段)
- 匈牙利算法比配，关联检测物体和跟踪物体
- 卡尔曼滤波，更新跟踪物体位置与速度信息(卡尔曼滤波阶段2：Update阶段)

进入HM物体跟踪的入口依旧在`LidarProcessSubnode::OnPointCloud`中：

```c++
/// file in apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
  /// call hdmap to get ROI
  ...
  /// call roi_filter
  ...
  /// call segmentor
  ...
  /// call object builder
  ...
  /// call tracker
  if (tracker_ != nullptr) {
    TrackerOptions tracker_options;
    tracker_options.velodyne_trans = velodyne_trans;
    tracker_options.hdmap = hdmap;
    tracker_options.hdmap_input = hdmap_input_;
    if (!tracker_->Track(objects, timestamp_, tracker_options, &(out_sensor_objects->objects))) {
    ...
    }
  }
}
```

在这部分，总共有三个比较绕的对象类，分别是Object、TrackedObject和ObjectTrack，在这里统一说明一下区别：

- Object类：常见的物体类，里面包含物体原始点云、多边形轮廓、物体类别、物体分类置信度、方向、长宽、速度等信息。**全模块通用**。
- TrackedObject类：封装Object类，记录了跟踪物体类属性，额外包含了中心、重心、速度、加速度、方向等信息。
- ObjectTrack类：封装了TrackedObject类，实际的跟踪解决方案，不仅包含了需要跟踪的物体(TrackedObject)，同时包含了跟踪物体滤波、预测运动趋势等函数。

所以可以看到，跟踪过程需要将原始Object封装成TrackedObject，创立跟踪对象；最后跟踪对象创立跟踪过程ObjectTrack，可以通过ObjectTrack里面的函数来对ObjectTrack所标记的TrackedObject进行跟踪。

#### 2.4.1 预处理


```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
bool HmObjectTracker::Track(const std::vector<ObjectPtr>& objects,
                            double timestamp, const TrackerOptions& options,
                            std::vector<ObjectPtr>* tracked_objects) {
  // A. track setup
  if (!valid_) {
    valid_ = true;
    return Initialize(objects, timestamp, options, tracked_objects);
  }
  // B. preprocessing
  // B.1 transform given pose to local one
  TransformPoseGlobal2Local(&velo2world_pose);
  // B.2 construct objects for tracking
  std::vector<TrackedObjectPtr> transformed_objects;
  ConstructTrackedObjects(objects, &transformed_objects, velo2world_pose,options);
  ...
}

bool HmObjectTracker::Initialize(const std::vector<ObjectPtr>& objects,
                                 const double& timestamp,
                                 const TrackerOptions& options,
                                 std::vector<ObjectPtr>* tracked_objects) {
  global_to_local_offset_ = Eigen::Vector3d(-velo2world_pose(0, 3), -velo2world_pose(1, 3), -velo2world_pose(2, 3));
  // B. preprocessing
  // B.1 coordinate transformation
  TransformPoseGlobal2Local(&velo2world_pose);
  // B.2 construct tracked objects
  std::vector<TrackedObjectPtr> transformed_objects;
  ConstructTrackedObjects(objects, &transformed_objects, velo2world_pose, options);
  // C. create tracks
  CreateNewTracks(transformed_objects, unassigned_objects);
  time_stamp_ = timestamp;
  // D. collect tracked results
  CollectTrackedResults(tracked_objects);
  return true;
}
```

预处理阶段主要分两个模块：A.跟踪建立(track setup)和B.预处理(preprocess)。跟踪建立过程，主要是对上述得到的物体对象进行跟踪目标的建立，这是Track第一次被调用的时候进行的，后续只需要进行跟踪对象更新即可。建立过程相对比较简单，主要包含：

1. 物体对象坐标系转换。(原先的lidar坐标系-->lidar局部ENU坐标系/有方向性)
2. 对每个物体创建跟踪对象，加入跟踪列表。
3. 记录现在被跟踪的对象

从上面代码来看，预处理阶段两模块重复度很高，这里我们就介绍`Initialize`对象跟踪建立函数。

(1) 第一步是进行坐标系的变换，这里我们注意到一个平移向量`global_to_local_offset_`，他是lidar坐标系到世界坐标系的变换矩阵`velo2world_trans`的平移成分，前面高精地图ROI过滤器小节我们讲过: **local局部ENU坐标系跟world世界坐标系之间只有平移成分，没有旋转。所以这里取了转变矩阵的平移成分，其实就是world世界坐标系转换到lidar局部ENU坐标系的平移矩阵(变换矩阵)。P_local = P_world + global_to_local_offset_**

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
void HmObjectTracker::TransformPoseGlobal2Local(Eigen::Matrix4d* pose) {
  (*pose)(0, 3) += global_to_local_offset_(0);
  (*pose)(1, 3) += global_to_local_offset_(1);
  (*pose)(2, 3) += global_to_local_offset_(2);
}
```

从上面的`TransformPoseGlobal2Local`函数代码我们可以得到一个没有平移成分，只有旋转成分的变换矩阵`velo2world_pose`，这个矩阵有什么作用？很简单，**这个矩阵就是lidar坐标系到lidar局部ENU坐标系的转换矩阵。**

(2) 第二步中需要根据前面CNN检测到的物体来创建跟踪对象，也就是将`Object`包装到`TrackedObject`中，那我们先来看一下`TrackedObject`类里面的成分：

| 名称 | 备注 |
| ---- | ---- |
| ObjectPtr object_ptr | Object对象指针 |
| Eigen::Vector3f barycenter | 重心，取该类所有点云xyz的平均值得到 |
| Eigen::Vector3f center | 中心， bbox4个角点外加平均高度计算得到 |
| Eigen::Vector3f velocity | 速度，卡尔曼滤波器预测得到 |
| Eigen::Matrix3f velocity_uncertainty | 不确定速度 |
| Eigen::Vector3f acceleration | 加速度 | 
| ObjectType type | 物体类型，行人、自行车、车辆等 |
| float association_score | -- |

从上面表格可以看到，`TrackedObject`封装了`Object`，并且只增加了少量速度，加速度等额外信息。

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
void HmObjectTracker::ConstructTrackedObjects(
    const std::vector<ObjectPtr>& objects,
    std::vector<TrackedObjectPtr>* tracked_objects, const Eigen::Matrix4d& pose,
    const TrackerOptions& options) {
  int num_objects = objects.size();
  tracked_objects->clear();
  tracked_objects->resize(num_objects);
  for (int i = 0; i < num_objects; ++i) {
    ObjectPtr obj(new Object());
    obj->clone(*objects[i]);
    (*tracked_objects)[i].reset(new TrackedObject(obj));                  // create new TrackedObject with object
    // Computing shape featrue
    if (use_histogram_for_match_) {
      ComputeShapeFeatures(&((*tracked_objects)[i]));                     // compute shape feature
    }
    // Transforming all tracked objects
    TransformTrackedObject(&((*tracked_objects)[i]), pose);               // transform coordinate from lidar frame to local ENU frame
    // Setting barycenter as anchor point of tracked objects
    Eigen::Vector3f anchor_point = (*tracked_objects)[i]->barycenter;
    (*tracked_objects)[i]->anchor_point = anchor_point;
    // Getting lane direction of tracked objects
    pcl_util::PointD query_pt;                                            // get lidar's world coordinate equals lidar2world_trans's translation part  
    query_pt.x = anchor_point(0) - global_to_local_offset_(0);
    query_pt.y = anchor_point(1) - global_to_local_offset_(1);
    query_pt.z = anchor_point(2) - global_to_local_offset_(2);
    Eigen::Vector3d lane_dir;
    if (!options.hdmap_input->GetNearestLaneDirection(query_pt, &lane_dir)) {
      lane_dir = (pose * Eigen::Vector4d(1, 0, 0, 0)).head(3);            // get nearest line direction from hd map
    }
    (*tracked_objects)[i]->lane_direction = lane_dir.cast<float>();
  }
}
```

`ConstructTrackedObjects`是由物体对象来创建跟踪对象的代码，这个过程相对来说比较简单易懂，没大的难点，下面就解释一下具体的功能。

- 针对`vector<ObjectPtr>& objects`中的每个对象，创建对应的`TrackedObject`，并且计算他的shape feature，这个特征计算比较简单，先计算物体xyz三个坐标轴上的最大和最小值，分别将其划分成10等份，对每个点xyz坐标进行bins投影与统计。最后的到的特征就是[x_bins,y_bins,z_bins]一共30维，归一化(除点云数量)后得到最终的shape feature。
- `TransformTrackedObject`函数进行跟踪物体的方向、中心、原始点云、多边形角点、重心等进行坐标系转换。lidar坐标系变换到local ENU坐标系。
- 根据lidar的世界坐标系坐标查询高精地图HD map计算车道线方向

(3) 第三步就是讲第二步中创建的跟踪对象(TrackedObject)建立跟踪，正式进行跟踪(加入进ObjectTrack)。

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
void HmObjectTracker::CreateNewTracks(
    const std::vector<TrackedObjectPtr>& new_objects,
    const std::vector<int>& unassigned_objects) {
  // Create new tracks for objects without matched tracks
  for (size_t i = 0; i < unassigned_objects.size(); i++) {
    int obj_id = unassigned_objects[i];
    ObjectTrackPtr track(new ObjectTrack(new_objects[obj_id]));
    object_tracks_.AddTrack(track);
  }
}
```

同时函数`CollectTrackedResults`会将当前正在跟踪的对象(世界坐标系坐标形式)保存到向量中，该部分代码比较简单就不贴出来了。

#### 2.4.2 卡尔曼滤波，跟踪物体对象(卡尔曼滤波阶段1： Predict)

在预处理阶段，每个物体Object类经过封装以后，产生一个对应的ObjectTrack过程类，里面封装了对应要跟踪的物体(TrackedObject，由Object封装而来)。这个阶段的工作就是对跟踪物体TrackedObject进行卡尔曼滤波并预测其运动方向。

首先，在这里我们简单介绍一下卡尔曼滤波的一些基础公式，方便下面理解。

-----------------------------------------------------------------------------------------
一个系统拥有一个状态方程和一个观测方程。观测方程是我们能宏观看到的一些属性，在这里比如说汽车重心xyz的位置和速度；而状态方程是整个系统里面的一些状态，包含能观测到的属性(如汽车重心xyz的位置和速度)，也可能包含其他一些看不见的属性，这些属性甚至我们都不能去定义它的物理意义。**因此观测方程的属性是状态方程的属性的一部分**现在有：

状态方程: $X_t = A_{t,t-1}X_{t-1} + W_t$, 其中$W_t \to N(0,Q) $

观测方程: $Z_t = C_tX_t + V_t$, 其中$V_t \to N(0,R) $

卡尔曼滤波分别两个阶段，分别是预测Predict与更新Update：

- Predict预测阶段
	- 利用上时刻t-1最优估计$X_{t-1}$预测当前时刻状态$X_{t,t-1} = A_{t,t-1}X_{t-1}$，这个$X_{t,t-1}$不是t时刻的最优状态，只是估计出来的状态
	- 利用上时刻t-1最优协方差矩阵$P_{t-1}$预测当前时刻协方差矩阵$P_{t,t-1} = A_{t,t-1}P_{t-1}{A_{t,t-1}}^T + Q$，这个$P_{t,t-1}$也不是t时刻最优协方差
- Update更新阶段
	- 利用$X_{t,t-1}$估计出t时刻最优状态$X_t = X_{t,t-1} + H_t[Z_t - C_tX_{t,t-1}]$, 其中$H_t = P_{t,t-1}{C_t}^T[C_tP_{t,t-1}{C_t}^T + R]^{-1}$
	- 利用$P_{t,t-1}$估计出t时刻最优协方差矩阵$P_t = [I - H_tC_t]P_{t,t-1}$

最终t从1开始递归计算k时刻的最优状态$X_k$与最优协方差矩阵$P_t$

--------------------------------------------------------------------------------------------

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
bool HmObjectTracker::Track(const std::vector<ObjectPtr>& objects,
                            double timestamp, const TrackerOptions& options,
                            std::vector<ObjectPtr>* tracked_objects) {
  // A. track setup
  ...
  // B. preprocessing
  // B.1 transform given pose to local one
  ...
  // B.2 construct objects for tracking
  ...
  // C. prediction
  std::vector<Eigen::VectorXf> tracks_predict;
  ComputeTracksPredict(&tracks_predict, time_diff);
  ...
}

void HmObjectTracker::ComputeTracksPredict(std::vector<Eigen::VectorXf>* tracks_predict, const double& time_diff) {
  // Compute tracks' predicted states
  std::vector<ObjectTrackPtr>& tracks = object_tracks_.GetTracks();
  for (int i = 0; i < no_track; ++i) {
    (*tracks_predict)[i] = tracks[i]->Predict(time_diff);   // track every tracked object in object_tracks_(ObjectTrack class) 
  }
}
```

从代码中我们可以看到，这个过程其实就是对object_tracks_列表中每个物体调用其Predict函数进行滤波跟踪(object_tracks_是上阶段Object--TrackedObject--ObjectTrack的依次封装)。接下去我们就对这个Predict函数进行深层次的挖掘和分析，看看它实现了卡尔曼过滤器的那个阶段工作。

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/object_track.cc
Eigen::VectorXf ObjectTrack::Predict(const double& time_diff) {
  // Get the predict of filter
  Eigen::VectorXf filter_predict = filter_->Predict(time_diff);
  // Get the predict of track
  Eigen::VectorXf track_predict = filter_predict;
  track_predict(0) = belief_anchor_point_(0) + belief_velocity_(0) * time_diff;
  track_predict(1) = belief_anchor_point_(1) + belief_velocity_(1) * time_diff;
  track_predict(2) = belief_anchor_point_(2) + belief_velocity_(2) * time_diff;
  track_predict(3) = belief_velocity_(0);
  track_predict(4) = belief_velocity_(1);
  track_predict(5) = belief_velocity_(2);
  return track_predict;
}

/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/kalman_filter.cc
Eigen::VectorXf KalmanFilter::Predict(const double& time_diff) {
  // Compute predict states
  Eigen::VectorXf predicted_state;
  predicted_state.resize(6);
  predicted_state(0) = belief_anchor_point_(0) + belief_velocity_(0) * time_diff;
  predicted_state(1) = belief_anchor_point_(1) + belief_velocity_(1) * time_diff;
  predicted_state(2) = belief_anchor_point_(2) + belief_velocity_(2) * time_diff;
  predicted_state(3) = belief_velocity_(0);
  predicted_state(4) = belief_velocity_(1);
  predicted_state(5) = belief_velocity_(2);
  // Compute predicted covariance
  Propagate(time_diff);
  return predicted_state;
}

void KalmanFilter::Propagate(const double& time_diff) {
  // Only propagate tracked motion
  ity_covariance_ += s_propagation_noise_ * time_diff * time_diff;
}
```

**从上面两个函数可以明显看到这个阶段就是卡尔曼滤波器的Predict阶段。同时可以看到**：

1. `track_predict/predicted_state`相当于卡尔曼滤波其中的$X_{t,t-1}$, `belief_anchor_point_`和`belief_velocity_`相当于$X_t$, `ity_covariance_`交替存储$P_t$和$P_{t,t-1}$(Why?可以从上面的卡尔曼滤波器公式看到$P_t$在估测完$P_{t,t-1}$以后就没用了，所以可以覆盖存储，节省部分空间)

2. 状态方程和观测方程其实本质上是一样，也就是相同维度的。都是6维，分别表示重心的xyz坐标和重心xyz的速度。同时在这个应用中，短时间间隔内。当前时刻重心位置=上时刻重心位置 + 上时刻速度\*时间差，所以可知卡尔曼滤波器中$A_{t,t-1}\equiv1$, $Q = I\*ts^2$

3. 该过程工作:首先利用上时刻的最优估计`belief_anchor_point_`和`belief_velocity_`(等同于$X_{t-1}$)估计出t时刻的状态`predicted_state`(等同于$X_{t,t-1}$); 然后估计当前时刻的协方差矩`ity_covariance_`($P_{t-1}$和$P_{t,t-1}$交替存储)。

#### 2.4.3 匈牙利算法比配，关联检测物体和跟踪物体

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
bool HmObjectTracker::Track(const std::vector<ObjectPtr>& objects,
                            double timestamp, const TrackerOptions& options,
                            std::vector<ObjectPtr>* tracked_objects) {
  // A. track setup
  ...
  // B. preprocessing
  // B.1 transform given pose to local one
  ...
  // B.2 construct objects for tracking
  ...
  // C. prediction
  ...
  // D. match objects to tracks
  std::vector<TrackObjectPair> assignments;
  std::vector<int> unassigned_objects;
  std::vector<int> unassigned_tracks;
  std::vector<ObjectTrackPtr>& tracks = object_tracks_.GetTracks();
  if (matcher_ != nullptr) {
    matcher_->Match(&transformed_objects, tracks, tracks_predict, &assignments, &unassigned_tracks, &unassigned_objects);
  }
  ...
}

/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hungarian_matcher.cc
void HungarianMatcher::Match(std::vector<TrackedObjectPtr>* objects,
                             const std::vector<ObjectTrackPtr>& tracks,
                             const std::vector<Eigen::VectorXf>& tracks_predict,
                             std::vector<TrackObjectPair>* assignments,
                             std::vector<int>* unassigned_tracks,
                             std::vector<int>* unassigned_objects) {
  // A. computing association matrix
  Eigen::MatrixXf association_mat(tracks.size(), objects->size());
  ComputeAssociateMatrix(tracks, tracks_predict, (*objects), &association_mat);

  // B. computing connected components
  std::vector<std::vector<int>> object_components;
  std::vector<std::vector<int>> track_components;
  ComputeConnectedComponents(association_mat, s_match_distance_maximum_,
                             &track_components, &object_components);
  // C. matching each sub-graph
  ...
}
```

这个阶段主要的工作是匹配CNN分割+MinBox检测到的物体和当前ObjectTrack的跟踪物体。主要的工作为：

- A. Object和TrackedObject之间关联矩阵`association_mat`计算
- B. 子图划分，利用上述的关联矩阵和设定的阈值(两两评分小于阈值则互相关联，即节点之间链接)，将矩阵分割成一系列子图
- C. 匈牙利算法进行二分图匹配，得到cost最小的(Object,TrackedObject)连接对

1. 关联矩阵`association_mat`计算

经过CNN分割与MinBox构建以后可以得到N个Object，而目前跟踪列表中有M个TrackedObject，所以需要构建一个NxM的关联矩阵，矩阵中每个元素(即关联评分)的计算共分为5个子项。

- 重心位置坐标距离差异评分
- 物体方向差异评分
- 标定框尺寸差异评分
- 点云数量差异评分
- 外观特征差异评分

最终以0.6，0.2，0.1，0.1，0.5的权重加权求和得到关联评分。

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/track_object_distance.cc
float TrackObjectDistance::ComputeDistance(const ObjectTrackPtr& track,
                                           const Eigen::VectorXf& track_predict,
                                           const TrackedObjectPtr& new_object) {
  // Compute distance for given trakc & object
  float location_distance = ComputeLocationDistance(track, track_predict, new_object);
  float direction_distance = ComputeDirectionDistance(track, track_predict, new_object);
  float bbox_size_distance = ComputeBboxSizeDistance(track, new_object);
  float point_num_distance = ComputePointNumDistance(track, new_object);
  float histogram_distance = ComputeHistogramDistance(track, new_object);

  float result_distance = s_location_distance_weight_ * location_distance +            // s_location_distance_weight_ = 0.6
                          s_direction_distance_weight_ * direction_distance +          // s_direction_distance_weight_ = 0.2
                          s_bbox_size_distance_weight_ * bbox_size_distance +          // s_bbox_size_distance_weight_ = 0.1
                          s_point_num_distance_weight_ * point_num_distance +          // s_point_num_distance_weight_ = 0.1
                          s_histogram_distance_weight_ * histogram_distance;           // s_histogram_distance_weight_ = 0.5
  return result_distance;
}
```

各个子项的计算方式，这里以文字形式描述，假设：

Object重心坐标为(x1,y1,z1)，方向为(dx1,dy1,dz1)，bbox尺寸为(l1,w1,h1), shape featrue为30维向量sf1，包含原始点云数量n1

TrackedObject重心坐标为(x2,y2,z2)，方向为(dx2,dy2,dz2)，bbox尺寸为(l2,w2,h2), shape featrue为30维向量sf2，包含原始点云数量n2

则有：

- 重心位置坐标距离差异评分location_distance计算

$location-distance = \sqrt{{(x1 - x2)}^2 + {(y1 - y2)}^2}$ 

如果速度太大，则需要用方向向量去正则惩罚，具体可以参考代码


- 物体方向差异评分direction_distance计算

方向差异其实就是计算两个向量的夹角:

$cos\theta = a·b/(|a|·|b|)$

夹角越大，差异越大，cos值越小；夹角越大，差异越大，cos值越大

最后使用1-cos计算评分，差异越小，评分越大。

- 标定框尺寸差异评分bbox_size_distance计算

代码中首先计算两个量`dot_val_00`和`dot_val_01`：

```c++
/// file in apollo/master/modules/perception/obstacle/lidar/tracker/hm_tracker/track_object_distance.cc
float TrackObjectDistance::ComputeBboxSizeDistance(const ObjectTrackPtr& track, const TrackedObjectPtr& new_object) {
  double dot_val_00 = fabs(old_bbox_dir(0) * new_bbox_dir(0) + old_bbox_dir(1) * new_bbox_dir(1));
  double dot_val_01 = fabs(old_bbox_dir(0) * new_bbox_dir(1) - old_bbox_dir(1) * new_bbox_dir(0));
  bool bbox_dir_close = dot_val_00 > dot_val_01;

  if (bbox_dir_close) {
    float diff_1 = fabs(old_bbox_size(0) - new_bbox_size(0)) / std::max(old_bbox_size(0), new_bbox_size(0));
    float diff_2 = fabs(old_bbox_size(1) - new_bbox_size(1)) / std::max(old_bbox_size(1), new_bbox_size(1));
    size_distance = std::min(diff_1, diff_2);
  } else {
    float diff_1 = fabs(old_bbox_size(0) - new_bbox_size(1)) / std::max(old_bbox_size(0), new_bbox_size(1));
    float diff_2 = fabs(old_bbox_size(1) - new_bbox_size(0)) / std::max(old_bbox_size(1), new_bbox_size(0));
    size_distance = std::min(diff_1, diff_2);
  }
  return size_distance;
}
```

这两个量有什么意义？这里简单解释一下，从计算方式可以看到：

其实`dot_val_00`是两个坐标的点积，数学计算形式上是方向1投影到方向2向量上得到向量3，最后向量3模乘以方向2模长，这么做可以估算方向差异。因为，当方向1和方向2两个向量夹角靠近0或180度时，投影向量很长，`dot_val_00`这个点积的值会很大。**`dot_val_00`越大说明两个方向越接近。**

同理`dot_val_01`上文我们提到过，差积的模可以衡量两个向量组成的四边形面积大小，这么做也可以估算方向差异。因为，当方向1和方向2两个向量夹角靠近90度时，组成的四边形面积最大，`dot_val_01`这个差积的模会很大。**`dot_val_00`越大说明两个方向越背离。**

- 点云数量差异评分point_num_distance计算

$point-num_distance = |n1-n2|/max(n1,n2)$

- 外观特征差异评分histogram_distance计算

$histogram-distance = \sum_{m=0}^{30} |sf1[m]-sf2[m]|$


2. 子图划分

子图划分首先根据上步骤计算的`association_mat`矩阵，利用超参数`s_match_distance_maximum_=4`，关联值小于阈值的判定为连接，否则不连接。最终得到的连接矩阵大小为(N+M)x(N+M)

```c++
void HungarianMatcher::ComputeConnectedComponents(
    const Eigen::MatrixXf& association_mat, const float& connected_threshold,
    std::vector<std::vector<int>>* track_components,
    std::vector<std::vector<int>>* object_components) {
  // Compute connected components within given threshold
  int no_track = association_mat.rows();
  int no_object = association_mat.cols();
  std::vector<std::vector<int>> nb_graph;
  nb_graph.resize(no_track + no_object);
  for (int i = 0; i < no_track; i++) {
    for (int j = 0; j < no_object; j++) {
      if (association_mat(i, j) <= connected_threshold) {
        nb_graph[i].push_back(no_track + j);
        nb_graph[j + no_track].push_back(i);
      }
    }
  }

  std::vector<std::vector<int>> components;
  ConnectedComponentAnalysis(nb_graph, &components);  // sub_graph segment
  ...
}
```

主要的子图划分工作在`ConnectedComponentAnalysis`函数完成，具体的可以参考代码，相对来说比较简单。最后得到的`components`二维向量中，每一行为一个子图的组成元素。

3. 匈牙利算法对每个子图匹配

匹配的算法主要还是匈牙利算法的矩阵形式，跟wiki百科的基本描述一致，可以参考主页[匈牙利算法](https://en.wikipedia.org/wiki/Hungarian_algorithm)


#### 2.4.4 卡尔曼滤波，更新跟踪物体位置与速度信息(卡尔曼滤波阶段2：Update阶段)

这个阶段做的工作比较重要，对上述Hungarian Matcher得到的<OldTrackedObject, NewObject>追踪对。

- 计算真实的观测变量，包括真实观测到的车速度与加速度${Zv_t}$、$Za_t$
- 由上时刻最优速度与加速度$Xv_{t-1}$、$Xa_{t-1}$ 估测当前时刻的速度与加速度$Xv_{t,t-1}$、$Xa_{t,t-1}$
- 由估测速度与加速度$Xv_{t,t-1}$、$Xa_{t,t-1}$，更新得到t时刻最优速度与加速度${Xv_t}$、$Xa_t$

另外这里还要一个说明，ObjectTrack类不仅封装了TrackedObject类，同时也封装了KalmanFilter类，KalmanFilter自己保存了上时刻最优状态，上时刻最优协方差矩阵，当前时刻最优状态等信息。TrackedObject、ObjectTrack里面也有这些状态。

**注意：KalmanFilter里面是滤波后的原始数据(没有实际应用的限制条件加入)；TrackedObject和ObjectTrack类同样保存了一份状态信息，这些状态信息是从KalmanFilter中得到的原始信息，并且加入实际应用限制滤波以后的状态信息；ObjectTrack类里面的状态是需要依赖TrackedObject类的，所以务必先更新TrackedObject再更新ObjectTrack的状态。总之三者状态更新顺序为：`KalmanFilter -> TrackedObject -> ObjectTrack`**

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/hm_tracker.cc
bool HmObjectTracker::Track(const std::vector<ObjectPtr>& objects,
                            double timestamp, const TrackerOptions& options,
                            std::vector<ObjectPtr>* tracked_objects) {
   // E. update tracks
  // E.1 update tracks with associated objects
  UpdateAssignedTracks(&tracks_predict, &transformed_objects, assignments, time_diff);
  // E.2 update tracks without associated objects
  UpdateUnassignedTracks(tracks_predict, unassigned_tracks, time_diff);
  DeleteLostTracks();
  // E.3 create new tracks for objects without associated tracks
  CreateNewTracks(transformed_objects, unassigned_objects);
  // F. collect tracked results
  CollectTrackedResults(tracked_objects);
  return true;
}

void HmObjectTracker::UpdateAssignedTracks(
    std::vector<Eigen::VectorXf>* tracks_predict,
    std::vector<TrackedObjectPtr>* new_objects,
    const std::vector<TrackObjectPair>& assignments, const double& time_diff) {
  // Update assigned tracks
  std::vector<ObjectTrackPtr>& tracks = object_tracks_.GetTracks();
  for (size_t i = 0; i < assignments.size(); i++) {
    int track_id = assignments[i].first;
    int obj_id = assignments[i].second;
    tracks[track_id]->UpdateWithObject(&(*new_objects)[obj_id], time_diff);
  }
}
```

从上述的代码可以看到，更新过程有`ObjectTrack::UpdateWithObject`和`ObjectTrack::UpdateWithoutObject`两个函数完成，这两个函数间接调用kalman滤波器完成滤波更新，接下去我们简单地分析`ObjectTrack::UpdateWithObject`函数的流程。

```c++
/// file in apollo/modules/perception/obstacle/lidar/tracker/hm_tracker/object_track.cc 
void ObjectTrack::UpdateWithObject(TrackedObjectPtr* new_object, const double& time_diff) {

  // A. update object track
  // A.1 update filter
  filter_->UpdateWithObject((*new_object), current_object_, time_diff);
  filter_->GetState(&belief_anchor_point_, &belief_velocity_);
  filter_->GetOnlineCovariance(&belief_velocity_uncertainty_);

  (*new_object)->anchor_point = belief_anchor_point_;
  (*new_object)->velocity = belief_velocity_;
  (*new_object)->velocity_uncertainty = belief_velocity_uncertainty_;

  belief_velocity_accelaration_ = ((*new_object)->velocity - current_object_->velocity) / time_diff;
  // A.2 update track info
  ...

  // B. smooth object track
  // B.1 smooth velocity
  SmoothTrackVelocity((*new_object), time_diff);
  // B.2 smooth orientation
  SmoothTrackOrientation();
}
```

从代码中也可以间接看出更新的过程A.1和A.2是更新KalmanFilter和TrackedObject状态信息，B是更新ObjectTrack状态，这里必须按顺序来更新！

1. KalmanFilter滤波器状态更新

主要由`KalmanFilter::UpdateWithObject`函数完成，计算过成分下面几步：

- Step1. 计算更新评分 `ComputeUpdateQuality(new_object, old_object)`

这个过程主要是计算更新力度，因为每个Object和对应的跟踪目标TrackedObject之间有一个关联系数`association_score`，这个系数衡量两个目标之间的相似度，所以这里需要增加对目标的更新力度参数。

计算关联力度: `update_quality_according_association_score = 1 - association_score / s_association_score_maximum_`。默认s_association_score_maximum_= 1，关联越大(相似度越大)，更新力度越大

计算点云变化力度: `update_quality_according_point_num_change  = 1 - |n1 - n2| / max(n1, n2)`。点云变化越小，更新力度越大

最终取两个值的较小值最为最终的更新力度。

- Step2. 计算当前时刻的速度`measured_velocity`和和加速度`measured_acceleration `(这两个变量相当于卡尔曼滤波中的观测变量$Z_t$)

首先计算重心速度: `measured_anchor_point_velocity = [NewObject_barycenter(x,y,z) - OldObject_barycenter(x,y,z)] / timediff`。timediff是两次计算的时间差，很简单地计算方式

其次计算标定框(中心)速度：`measured_bbox_center_velocity  = [NewObject_center(x,y,z) - OldObject_center(x,y,z)] / timediff`。这里的中心区别于上面的重心，重心是所有点云的平均值；重心是MinBox的中心值。还有一个需要注意的是，如果求出来的中心速度方向和重心方向相反，这时候有干扰，中心速度暂时置为0。

然后计算标定框角点速度:

A. 根据NewObject的点云计算bbox(这不是MinBox)，并求出中心center，然后根据反向求出4个点。如果NewObject方向是dir，那么首先对dir进行归一化得到dir_normal=dir/|dir|^2=(nx,ny,0)，然后求他的正交方向dir_ortho=(-ny,nx,0)，如果中心点坐标center，那么左上角的坐标就是: center+dir\*size[0]\*0.5+ortho_dir\*size[1]\*0.5。根据这个公式可以计算出其他三个点的坐标。

B. 计算标定框bbox四个角点的速度: `bbox_corner_velocity = ((new_bbox_corner - old_bbox_corner) / time_diff)`公式与上面的重心、中心计算方式一样。

C. 计算4个角点的在主方向上的速度，去最小的点最为标定框角点速度。只需要将B中的`bbox_corner_velocity`投影到主方向即可。

最后在重心速度、重心速度、bbox角速度中选择速度增益最小的最后最终物体的增益。增益=当前速度-上时刻最优速度

加速度`measured_acceleration `计算比较简单，采用最近3次的速度(v1,t1),(v2,t2),(v3,t3)，然后加速度a=(v3-v1)/(t2+t3)。注意(v2,t2)意思是某一时刻最优估计速度为v2，且距离上次的时间差为t2，所以三次测量的时间差为t2+t3。速冻变化为v3-v1。

- Step3. 估算最优的速度与加速度(卡尔曼滤波Update步骤)

首先，计算卡尔曼增益$H_t = P_{t,t-1}{C_t}^T[C_tP_{t,t-1}{C_t}^T + R]^{-1}$，在apollo代码中计算代码如下：

```c++
// Compute kalman gain
Eigen::Matrix3d mat_c = Eigen::Matrix3d::Identity();                                 // C_t
Eigen::Matrix3d mat_q = s_measurement_noise_ * Eigen::Matrix3d::Identity();          // R_t
Eigen::Matrix3d mat_k = velocity_covariance_ * mat_c.transpose() *                   // H_t
      (mat_c * velocity_covariance_ * mat_c.transpose() + mat_q).inverse();
```

从上面可知，代码和我们给出的结果是一致的。

然后，由当前时刻的估算速度$X_{t,t-1}$、观测变量$Z_t$以及卡尔曼增益$H_t$，得到当前时刻的最优速度估计$X_t = X_{t,t-1} + H_t[Z_t - C_tX_{t,t-1}]$，在apollo代码中计算了速度增益，也就是$X_t-X_{t,t-1}$：

```c++
// Compute posterior belief
Eigen::Vector3d measured_velocity_d = measured_velocity.cast<double>();                      // Zv_t
Eigen::Vector3d priori_velocity = belief_velocity_ + belief_acceleration_gain_ * time_diff;  // Xv_{t,t-1}
Eigen::Vector3d velocity_gain = mat_k * (measured_velocity_d - mat_c * priori_velocity);     // Gain = Xv_t - Xv_{t,t-1}
```

然后对速度增益进行平滑并且保存当前t时刻最优速度以及最优加速度

```c++
// Breakdown
ComputeBreakdownThreshold();
if (velocity_gain.norm() > breakdown_threshold_) {
  velocity_gain.normalize();
  velocity_gain *= breakdown_threshold_;
}

belief_anchor_point_ = measured_anchor_point_d;
belief_velocity_ = priori_velocity + velocity_gain;          // Xv_t = Xv_{t,t-1} + Gain
belief_acceleration_gain_ = velocity_gain / time_diff;       // Acc_t = Xv_t / timediff
```

最后就是速度整流并且修正估计协方差矩阵$P_{t,t-1}$，得到当前时刻最优协方差矩阵$P_t=[I-H_tC_t]P_{t,t-1}$，在这个应用中$C_t\equiv1$

```c++
// Adaptive
if (s_use_adaptive_) {
  belief_velocity_ -= belief_acceleration_gain_ * time_diff;
  belief_acceleration_gain_ *= update_quality_;
  belief_velocity_ += belief_acceleration_gain_ * time_diff;
}

// Compute posterior covariance
velocity_covariance_ = (Eigen::Matrix3d::Identity() - mat_k * mat_c) * velocity_covariance_;   // P_t

```

加速度更新与上述速度更新方法一致。

- Step4. 缓存更新信息

将观测变量`measured_velocity`和时间差`time_diff`缓存，同时使用观测速度`measured_velocity`对实时协方差矩阵`online_velocity_covariance_ `进行更新


2. TrackedObject状态更新

设置TrackedObject的重心，速度(滤波器得到的t时刻最优速度`belief_velocity_`)，加速度([最优速度`belief_velocity_` - OldObject的最优速度]/时间差)

更新跟踪时长(++age)，目标可见次数(++total_visible_count_)，跟踪总时长(period_ += time_diff)，连续不可见时长置0(consecutive_invisible_count_=0)


3. ObjectTrack状态更新(速度与方向平滑滤波)

由原始KalmanFilter中的各个状态信息，加入实际应用中的限制进行滤波得到ObjectTrack的状态信息，这些信息才是真实被使用的。

对跟踪物体的速度整流过程如下(`ObjectTrack::SmoothTrackVelocity`)：

- 如果物体的加速度增益查过一定阈值(s_acceleration_noise_maximum_, 默认为5)，那么当前速度保持上时刻的速度。
- 否则，对小速度物体进行修建。计算物体的速度，默认`s_speed_noise_maximum_ = 0.4`
	- 如果`velocity_is_noise = speed < (s_speed_noise_maximum_ / 2)` ,则判定为噪声
	- 如果`velocity_is_small = speed < (s_speed_noise_maximum_ / 1)`，则判定为小速度
	- 计算物体两个时刻角度的变化，`fabs(velocity_angle_change) > M_PI / 4.0`，如果cos值小于pi/4(45度)，说明物体没有角度变化
	最终判断：if(velocity_is_noise || (velocity_is_small && is_velocity_angle_change)) 如果速度是噪声，或者速度很小方向不变，那么认定车是静止的。
	对于车是静止的，真实速度和加速度都设置为0.

这里需要注意：

>// NEED TO NOTICE: claping small velocity may not reasonable when the true
> velocity of target object is really small. e.g. a moving out vehicle in
> a parking lot. Thus, instead of clapping all the small velocity, we clap
> those whose history trajectory or performance is close to a static one.

按照官方代码提醒，其实这样对小速度物体进行修剪时不太合理，因为某些情况下物体速度确实很小，但是他确实是在运动。E.g. 汽车倒车的时候，速度小，但是不能被忽略。所以最好的方法是根据历史的轨迹(重心，anchor_point)来判断物体在小速度的情况下是否是运动的。

对跟踪物体的方向整流过程如下(`ObjectTrack::SmoothTrackOrientation`)，如果物体运动比较明显`velocity_is_obvious = current_speed > (s_speed_noise_maximum_ * 2)`(大于0.4m/s)，那么当前运动方向为物体速度的方向；否则设定为车道线方向。

就这样，经过三步骤，跟踪配对的物体(Object-TrackedObject存在)完成了状态信息的更新，主要包括当前时刻最优速度、方向、加速等信息。

---------------------------------------------------------------------------

如果跟踪物体中没有找到对应的Object与之匹配，就需要使用`UpdateUnassignedTracks`函数来更新跟踪物体的信息。从上面我们可以看到，匹配成功的可以用Object的属性来计算观测变量，间接估算出t时刻的最优状态。 但是未匹配的TrackedObject无法因为找不到Object，所以无法了解当前时刻真实能测量到的位置、速度与加速度信息，因此只能依赖自身上时刻的最优状态来推算出当前时候的状态信息(注意，这个推算出来的不是最优状态)。

对未找到Object的跟踪物体，更新过程如下：

1. 使用2.4.2节中的估算数据来预测当前时刻的状态

```c++
Eigen::Vector3f predicted_shift = predict_state.tail(3) * time_diff;
new_obj->anchor_point = current_object_->anchor_point + predicted_shift;
new_obj->barycenter = current_object_->barycenter + predicted_shift;
new_obj->center = current_object_->center + predicted_shift;
```

其中`predicted_shift`是利用卡尔曼滤波Predict阶段预测到的当前时刻物体重心位置与速度状态$Xp_{t,t-1}$和$Xv_{t,t-1}$，乘以时间差就可以得到这个时间差内的位移，去更新中心，重心。

2. 上时刻TrackedObject里面的原始点云和多边形坐标也加上这个位移，完成更新。

3. 更新KalmanFilter里面的原始状态信息，`KalmanFilter::UpdateWithoutObject`，KalmanFilter只更新重心坐标，不需要更新速度和加速度(因为无法更新，缺少观测数据Z，不能使用卡尔曼滤波器的Update过程去更新)。

4. 更新TrackedObject状态信息，更新跟踪时长(++age)，跟踪总时长(period_ += time_diff)，更新连续不可见时长置(++consecutive_invisible_count_=0)

5. 更新历史缓存


更新完匹配成功和不成功的跟踪物体以后，下一步就是从跟踪列表中删掉丢失的跟踪物体。遍历整个跟踪列表：

1. 可见次数/跟踪时长小于阈值(s_track_visible_ratio_minimum_，默认0.6)，删除
2. 连续不可见次数大于阈值(s_track_consecutive_invisible_maximum_，默认1)，删除

---------------------------------------------------------------------------------

如果Object没有找到对应的TrackedObject与之匹配，那么就创建新的跟踪目标，并且加入跟踪队列。


最终对HM物体跟踪做一个总结与梳理，物体跟踪主要是对上述CNN分割与MinBox边框构建产生的Object对一个跟踪与匹配，主要流程是：

- Step1，预处理，Object里面的中心，重心，点云，多边形凸包从lidar坐标系转换成局部ENU坐标系。
- Step2. 将坐标转换完成Object封装成TrackedObject，方便后续加入跟踪列表
- Step3. 使用卡尔曼滤波Predict阶段，对正处于跟踪列表中的跟踪物体进行当前时刻重心位置、速度的预测
- Step4. 使用当前检测到的Object(封装成了TrackedObject)，去和跟踪列表中的物体进行匹配
	- 计算Object与TrackedObject的关联矩阵
		- 重心位置坐标距离差异评分
		- 物体方向差异评分
		- 标定框尺寸差异评分
		- 点云数量差异评分
		- 外观特征差异评分
	- 根据关联矩阵，配合阈值，划分子图
	- 对于每个子图使用匈牙利匹配算法(Hungarian Match)进行匹配，得到<Object,TrackedObject>、<Object,None>, <None,TrackedObject>
		- <Object,TrackedObject>成功匹配(有Object计算观测数据)，更新KalmanFilter状态(Update阶段), 更新TrackedObject状态，更新ObjectTrack状态
		- <None,TrackedObject>没有对应的Object(无法得到观测数据，无法使用卡尔曼滤波估算最优速度)，更新部分KalmanFilter状态(仅重心)，跟新TrackedObject状态，更新ObjectTrack状态
		- <Object,None>创建新的TrackedObject，加入跟踪列表
	- 删除丢失的跟踪目标
		- 可见次数/跟踪时长过小
		- 连续不可见次数过大