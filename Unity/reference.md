## 01-环境搭建与核心特性

### Unity 2022 LTS 核心技术背景
- [cite_start]LTS 版本的定义与稳定性优势 [cite: 6]
- [cite_start]面向数据的技术栈 (DOTS) 的生产环境集成 [cite: 7, 8]
- [cite_start]Netcode for GameObjects (NGO) 联机方案重构 [cite: 9]
- [cite_start]渲染管线演进 (URP Forward+ 与 HDRP 水体/云层) [cite: 10]

### 标准化开发环境构建
- [cite_start]Unity Hub 版本管理与模块安装 [cite: 12]
- [cite_start]目标平台 Build Support 的选择 (Android/iOS/WebGL) [cite: 12]
- [cite_start]IDE 选择与配置 (Visual Studio 2022/VS Code) [cite: 13]
- [cite_start]智能提示 (IntelliSense) 与 API 文档关联 [cite: 13]
- [cite_start]Git 环境配置与 .gitignore 文件设置 [cite: 13]

## 02-基础入门与工作流

### Unity 编辑器核心模块解构
- [cite_start]Project 窗口与 .meta 文件机制 [cite: 20]
- [cite_start]资源 GUID 引用保护原则 [cite: 20]
- [cite_start]Hierarchy 与 Scene 窗口的层级关系 [cite: 22]
- [cite_start]坐标系统：世界坐标 (Position) 与本地坐标 (LocalPosition) [cite: 23]
- [cite_start]Inspector 窗口的数据驱动设计 [cite: 24]
- [cite_start]Game 窗口与 Stats 性能面板 [cite: 25]

### 核心概念：GameObject 与 Component
- [cite_start]组合优于继承 (Composition over Inheritance) 架构思想 [cite: 27]
- [cite_start]Transform 组件的核心作用 (位置/旋转/缩放) [cite: 28]
- [cite_start]Mesh Filter 与 Mesh Renderer 的区别 [cite: 29]
- [cite_start]MonoBehaviour 脚本基类与组件自定义 [cite: 30]
- [cite_start]预制体 (Prefab) 系统与模版实例机制 [cite: 31]
- [cite_start]嵌套预制体 (Nested Prefab) 的模块化应用 [cite: 31]

### 游戏开发 C# 编程基础
- [cite_start]浮点数 (float) 在游戏开发中的应用 [cite: 34]
- [cite_start]Public 变量与 Inspector 暴露机制 [cite: 34]
- [cite_start]核心生命周期：Awake() 的初始化逻辑 [cite: 35]
- [cite_start]核心生命周期：Start() 的首帧前执行 [cite: 35]
- [cite_start]核心生命周期：Update() 的每帧逻辑处理 [cite: 35]
- [cite_start]核心生命周期：FixedUpdate() 的物理计算处理 [cite: 35]
- [cite_start]组件获取方法 GetComponent<Type>() [cite: 36]
- [cite_start]对象查找方法 GameObject.Find() 及其性能开销 [cite: 36, 37]

## 03-核心系统原理

### 物理引擎深度解析
- [cite_start]刚体 (Rigidbody) 与重力模拟 [cite: 41, 42]
- [cite_start]IsKinematic 属性与代码控制权 [cite: 42]
- [cite_start]碰撞体 (Collider) 与实体碰撞检测 [cite: 43]
- [cite_start]触发器 (Trigger) 与区域事件检测 [cite: 43]
- [cite_start]OnCollisionEnter 与 OnTriggerEnter 的调用区分 [cite: 43]
- [cite_start]物理材质 (Physics Material) 的摩擦力与弹力设置 [cite: 44]
- [cite_start]层级碰撞矩阵 (Layer Collision Matrix) 与性能优化 [cite: 45]
- [cite_start]2D 物理与 3D 物理系统的独立性 [cite: 46]

### 输入系统演进
- [cite_start]旧版 Input Manager 的轮询机制 [cite: 49, 50]
- [cite_start]New Input System 的事件驱动架构 [cite: 50]
- [cite_start]Input Actions 资产创建与动作定义 [cite: 51]
- [cite_start]逻辑代码与硬件设备的解耦 [cite: 51]
- [cite_start]多设备支持与运行时改键功能 [cite: 50]

### UI 系统架构
- [cite_start]UGUI (Unity UI) 的可视化工作流 [cite: 55]
- [cite_start]RectTransform 的锚点 (Anchors) 与轴心点 (Pivot) [cite: 62]
- [cite_start]UI Toolkit 的 Web 技术栈思想 (UXML/USS) [cite: 59]
- [cite_start]UI Toolkit 的样式逻辑分离优势 [cite: 60]
- [cite_start]运行时游戏 UI 的支持现状 [cite: 61]

### 场景管理 (Scene Management)
- [cite_start]同步加载 SceneManager.LoadScene() [cite: 65]
- [cite_start]异步加载 SceneManager.LoadSceneAsync() [cite: 66]
- [cite_start]叠加加载 (Additive Loading) 与模块化管理 [cite: 67]

## 04-进阶架构与表现力

### 架构设计模式
- [cite_start]ScriptableObject (SO) 的数据驱动应用 [cite: 71, 72]
- [cite_start]基于 SO 的事件架构与代码解耦 [cite: 73, 74]
- [cite_start]有限状态机 (FSM) 的设计与实现 [cite: 75]
- [cite_start]抽象 BaseState 类与具体状态逻辑 [cite: 76]

### 动画系统 (Mecanim)
- [cite_start]Animator 可视化逻辑状态机 [cite: 78]
- [cite_start]Blend Trees (混合树) 与运动过渡 [cite: 79]
- [cite_start]Animation Layers (动画分层) 与 Avatar Masks [cite: 82]
- [cite_start]Inverse Kinematics (IK) 反向动力学应用 [cite: 83]

### 特效与视觉反馈
- [cite_start]Particle System (粒子系统) 核心模块 [cite: 87]
- [cite_start]Shader Graph 可视化着色器编辑 [cite: 88]
- [cite_start]Cinemachine 智能相机套件与运镜 [cite: 89, 90]
- [cite_start]震动反馈 (Impulse Listener) 实现 [cite: 90]

### 现代资源管理
- [cite_start]Addressables 系统与 AssetBundle 封装 [cite: 92, 93]
- [cite_start]基于地址字符串 (Address) 的统一加载 [cite: 94]
- [cite_start]引用计数与内存自动管理 [cite: 95]

## 05-多领域专项技术

### 2D 游戏开发专项
- [cite_start]Sprite Atlas v2 图集打包与 Draw Call 优化 [cite: 99]
- [cite_start]URP 2D Lighting 实时光照系统 [cite: 100]
- [cite_start]Tilemap 瓦片地图绘制与 Rule Tile [cite: 102]

### 3D 游戏开发专项
- [cite_start]Character Controller 与 Rigidbody 的适用场景区分 [cite: 105]
- [cite_start]NavMesh 导航网格烘焙 [cite: 106]
- [cite_start]AI Navigation 包与动态烘焙 [cite: 107]

### VR/AR 基础开发
- [cite_start]XR Interaction Toolkit (XRI) 框架 [cite: 109]
- [cite_start]XR Origin 与空间追踪配置 [cite: 110]
- [cite_start]Locomotion System (瞬移/平滑移动) [cite: 111]
- [cite_start]Interactors 交互逻辑 (射线抓取/直接接触) [cite: 112]
- [cite_start]XR Plugin Management 与 OpenXR 后端 [cite: 113]

### 多人联机基础 (Netcode)
- [cite_start]Netcode for GameObjects (NGO) 架构 [cite: 116]
- [cite_start]Network Transform 位置同步 [cite: 118]
- [cite_start]NetworkVariable 数值同步机制 [cite: 118]
- [cite_start]RPC (Remote Procedure Call) 远程过程调用 [cite: 119]
- [cite_start]ServerRpc 与 ClientRpc 的区别 [cite: 119, 120]
- [cite_start]UGS Relay (内网穿透) 与 Lobby (大厅) 服务集成 [cite: 121]

## 06-工程化与性能优化

### 性能分析与调优
- [cite_start]Profiler 性能热点分析 (CPU/GPU) [cite: 126]
- [cite_start]Memory Profiler 内存快照与泄漏排查 [cite: 127]
- [cite_start]对象池 (Object Pooling) 技术与 GC 优化 [cite: 129, 130]
- [cite_start]静态合批 (Static Batching) 与 GPU Instancing [cite: 131]

### 调试与问题排查
- [cite_start]Frame Debugger 渲染过程分析 [cite: 133]
- [cite_start]Debug.DrawRay 可视化调试技巧 [cite: 134]
- [cite_start]Visual Studio 断点 (Breakpoints) 与监视窗口 [cite: 135]
- [cite_start]NullReferenceException (空引用) 排查 [cite: 182, 184]
- [cite_start]物理穿透 (穿模) 原因与连续碰撞检测 [cite: 186, 189]
- [cite_start]脚本不执行的常见原因检查 [cite: 191]

## 07-实战流程与资源库

### 项目实战规范
- [cite_start]白模原型 (Prototyping) 验证阶段 [cite: 172]
- [cite_start]核心系统数据流向设计 [cite: 173]
- [cite_start]标准目录结构 (Scripts/Art/Prefabs) [cite: 175]
- [cite_start]版本控制提交规范 [cite: 177]
- [cite_start]目标设备真机测试流程 [cite: 178]
- [cite_start]发布配置 (Keystore/Xcode) [cite: 179]

### 团队协作技巧
- [cite_start]场景冲突 (Scene Conflict) 避免策略 [cite: 193]
- [cite_start]Multi-Scene Editing (多场景编辑) 工作流 [cite: 194]
- [cite_start]基于 Prefab 的协作修改模式 [cite: 194]

### 精选权威学习资源
- [cite_start]Unity Learn 官方互动课程 [cite: 139]
- [cite_start]官方手册与 API 文档 [cite: 144]
- [cite_start]官方示例项目 (Boss Room/Gem Hunter) [cite: 147, 149]
- [cite_start]经典书籍 (Unity in Action 3rd Ed) [cite: 153]
- [cite_start]优质开源项目 (San Andreas Unity) [cite: 167]