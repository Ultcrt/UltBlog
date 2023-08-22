---
title: Open3D 中 RayCast 在循环调用和使用张量情况下的性能对比
date: 2023-08-22T07:43:53.071Z
last_modified_at: 2023-08-22T07:43:53.077Z
excerpt: 使用Open3D的C++接口（默认编译选项），对 RayCast 在循环调用和使用张量情况下的性能对比
categories:
  - 工作随笔
tags:
  - Open3D
  - 测试
header:
  overlay_image: https://picsum.photos/1920/640
  caption: "来源: [**Lorem Picsum**](https://picsum.photos/)"
  teaser: https://raw.githubusercontent.com/isl-org/Open3D/master/docs/_static/open3d_logo_horizontal.png
---
开发过程中发现，项目在射线求交的计算耗时太大，不使用射线求交和使用射线求交的运行时间相差了3倍左右。

之前为了方便开发，求交时使用的是一个 $$1 * 6$$ 的张量分多次计算求交，示例代码如下所示：

```c++
    const auto start = (double)clock();

    open3d::t::geometry::RaycastingScene scene;
    const auto& tensor = open3d::t::geometry::TriangleMesh::FromLegacy(part, open3d::core::Float32, open3d::core::Int64);
    scene.AddTriangles(tensor);

    const auto numRays = (int)part.vertices_.size() * 10;
    auto rays = open3d::core::Tensor::Zeros({1, 6}, open3d::core::Float32);

    Eigen::Vector3d position(0.05,0.5, -0.25);

    for (size_t idx = 0; idx < numRays; idx++) {
        const auto& vertex = part.vertices_[idx % part.vertices_.size()];
        Eigen::Vector3d rayDir = (vertex - position).normalized();

        rays.SetItem({open3d::core::TensorKey::Index(0), open3d::core::TensorKey::Slice(0, 6, 1)},
                     open3d::core::Tensor::Init<float>({float(position.x()), float(position.y()), float(position.z()),
                                                        float(rayDir.x()), float(rayDir.y()), float(rayDir.z())}));
        // 循环进行多次求交
        auto castResults = scene.CastRays(rays);
    }
    cout << ((double)clock() - start) / CLOCKS_PER_SEC << endl;
```

而Open3D的RayCast实际上会在底层利用多线程来处理张量，理论上速度更快，因此使用下述代码进行测试：

```c++
    const auto start = (double)clock();
    open3d::t::geometry::RaycastingScene scene;
    const auto& tensor = open3d::t::geometry::TriangleMesh::FromLegacy(part, open3d::core::Float32, open3d::core::Int64);
    scene.AddTriangles(tensor);

    const auto numRays = (int)part.vertices_.size() * 10;
    auto rays = open3d::core::Tensor::Zeros({numRays, 6}, open3d::core::Float32);

    Eigen::Vector3d position(0.05,0.5, -0.25);

    for (size_t idx = 0; idx < numRays; idx++) {
        const auto& vertex = part.vertices_[idx % part.vertices_.size()];
        Eigen::Vector3d rayDir = (vertex - position).normalized();

        rays.SetItem({open3d::core::TensorKey::Index((int)idx), open3d::core::TensorKey::Slice(0, 6, 1)},
                     open3d::core::Tensor::Init<float>({float(position.x()), float(position.y()), float(position.z()),
                                                        float(rayDir.x()), float(rayDir.y()), float(rayDir.z())}));
    }
    auto castResults = scene.CastRays(rays);

    cout << ((double)clock() - start) / CLOCKS_PER_SEC << endl;
```

最终的测试结果（光线数量约为7万）如下图所示：

![](/UltBlog/assets/images/uploads/截图-2023-08-22-16-51-41.png)

可见，两种方式在性能上确实存在一定差异。

不过需要注意，张量在初始化时需要耗费一定时间，因此在光线数量较少的情况下，两者差距并不大。