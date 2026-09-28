# RV1126 YOLOv5 摄像头目标检测

面向正点原子 ATK / Rockchip RV1126 Linux 平台的 C/C++ 摄像头目标检测工程。通过 RKMedia 获取图像，使用 RGA 进行图像格式转换与缩放，通过 RKNN 运行 YOLOv5 模型，并把检测框绘制到显示画面。

GitHub 仓库名为 `rv1126-yolov5-object-detection`。原始项目目录名为 `12_atk_yolov5_object_recognize`；保留源文件名与构建目标名，以保持现有 CMake 引用一致。

该工程依赖特定板卡 SDK、模型和摄像头环境，不是普通桌面 OpenCV 程序。此次整理未执行交叉编译或板端运行。

## 功能与数据流

```text
摄像头 / ISP
  → RKMedia VI 通道
  → RGA 图像处理
  → RKNN YOLOv5 推理
  → 量化输出解码、置信度过滤与 NMS
  → OpenCV 绘制检测框
  → RKMedia VO / DRM 显示
```

源码包含显示和推理线程，以及传递检测结果的链表。后处理按 80 类、3 个输出分支与固定 YOLOv5 anchors 设计；更换模型时应同时核对类别、量化参数、输出布局和 anchors。

## 目录

| 文件或目录 | 用途 |
| --- | --- |
| `atk_yolov5_object_recognize.cpp` | 摄像头、显示、线程、RKNN 推理和结果绘制 |
| `atk_yolov5_object_recognize.h` | 应用依赖、尺寸配置和接口声明 |
| `postprocess.cc`、`include/postprocess.h` | YOLOv5 量化输出解码、标签及 NMS |
| `drm_func.c`、`rga_func.c` | DRM 缓冲区和 RGA 图像处理辅助函数 |
| `include/rknn_api.h` | RKNN API 头文件 |
| `include/3rdparty/` | 随项目提供的 DRM、RGA、stb、CImg 等头文件 |
| `lib/` | 随项目提供的 `libeasymedia.so` 及版本化名称 |
| `CMakeLists.txt` | ATK RV1126 交叉编译配置 |

`.gitignore` 排除了旧 `build/`、本机编辑器配置和编译中间文件。`lib/` 是原项目依赖的一部分，保留在仓库中。

## 环境依赖

- RV1126 目标板、兼容的摄像头、显示设备及配套系统镜像。
- Linux 构建主机、CMake 3.8 或更高版本、Make。
- ATK RV1126 交叉工具链，工程默认根目录为 `/opt/atk-dlrv1126-toolchain`。
- `arm-linux-gnueabihf-gcc` / `g++`、对应 Buildroot sysroot，C++11。
- 配套 RKMedia / EasyMedia、RGA、RKAIQ、RKNN 和 `sample_common_isp` 库及头文件。
- OpenCV：工程链接 `core`、`imgcodecs`、`imgproc`、`features2d`、`flann`、`highgui`、`freetype`。
- Linux pthread 与 dl。

仓库内的头文件和 `libeasymedia` 并不能替代完整板卡 SDK。`rkmedia_api.h`、`sample_common.h` 等头文件及其他库需要由配套工具链/sysroot 提供。默认编译启用了 `RKAIQ`。

## 构建

先检查 `CMakeLists.txt` 中的 `TOOLCHAIN_DIR`、编译器、sysroot 和库搜索路径。它们是原开发环境的配置，迁移到其他目录时需要修改。

在 Linux 构建主机的仓库根目录执行：

```bash
cmake -S . -B build
cmake --build build -j2
```

输出为 `build/atk_yolov5_object_recognize`，是目标板 ARM 程序。当前配置采用 `-O0` 和调试信息，不是已经调优的性能构建。

## 模型及板端文件

下列资源在源码中使用固定路径，但**没有随本项目提供**：

| 板端路径 | 用途 |
| --- | --- |
| `/userdata/bin/yolov5s_relu_rv1109_rv1126_out_opt.rknn` | 与当前 RKNN Runtime、80 类后处理兼容的模型 |
| `/userdata/bin/coco_80_labels_list.txt` | 与模型输出顺序一致的 80 类标签 |
| `/etc/iqfiles` | 摄像头 ISP IQ 配置 |

需从配套 SDK、原项目部署资料或自己的模型转换流程中准备这些文件。本仓库没有模型训练或 RKNN 转换脚本，也不提供不存在的模型下载链接。

## 运行

将编译程序部署到配置完成的板端，并确保运行库能够被动态链接器找到。例如把程序放入 `/userdata/bin/` 后执行：

```bash
cd /userdata/bin
./atk_yolov5_object_recognize
```

程序直接初始化摄像头、ISP、RGA、RKNN 和显示通道。当前没有实现可用的命令行配置接口，运行参数主要在源码中修改：

| 参数 | 当前配置 |
| --- | --- |
| 采集图像尺寸 | `1920 × 1080` |
| 显示尺寸 | `1280 × 720` |
| 摄像头索引 | `0` |
| VI 节点 | `rkispp_scale0`、`rkispp_scale1` |
| DRM 设备 | `/dev/dri/card0` |
| ISP 帧率设置 | `25`，不代表实际检测帧率 |
| 类别数 | `80` |

尺寸定义在应用头文件中；模型路径、设备节点与推理阈值在应用 `.cpp` 中；标签文件路径和 anchors 位于 `postprocess.cc`。

## 已知限制

- 原有检测结果链表的 `rknn_list_pop()` 在仅有一个节点的分支中没有正常赋值局部指针，函数末尾仍调用 `free(t)`，存在释放未初始化指针的风险，运行前应修复。
- 显示与推理线程共享链表，未看到对应的互斥保护，线程同步和退出清理需要进一步整理。
- 主循环依赖 `quit` 标志，当前代码没有完整的正常退出控制流程。
- 多处平台路径、图像尺寸和后处理参数硬编码，更换硬件或模型需要同步调整。
- 未提供自动化测试和板端性能记录。本次仅做文件与源码检查，未声称构建成功或运行稳定。

## 来源与版权

源文件保留 Rockchip 和原工程作者的声明。部分文件声明 Apache-2.0，部分文件引用 BSD 风格授权，第三方头文件也有各自声明；本次整理未擅自为整个仓库添加统一开源许可证。使用或再分发应遵循相应文件和 SDK 的授权。
