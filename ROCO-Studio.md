## 构建
### 编译服务器
```
roco@roco:~/backup/roco-studio_12_15/roco-studio$ cd server/
roco@roco:~/backup/roco-studio_12_15/roco-studio/server$ cd mqtt_server/
roco@roco:~/backup/roco-studio_12_15/roco-studio/server/mqtt_server$ ./dev.sh
```
### 编译客户端
```
roco@roco:~/backup/roco-studio_12_15/roco-studio$ cd client/
roco@roco:~/backup/roco-studio_12_15/roco-studio/client$ cd web
// 客户端 调试器 下的代码未更新，是因为客户端的代码有报错
// 把Web下的dist文件夹与node_modules删除，重新install
// roco@roco:~/backup/roco-studio_12_15/roco-studio/client/web$ npm install
roco@roco:~/backup/roco-studio_12_15/roco-studio/client/web$ npm run build
roco@roco:~/backup/roco-studio_12_15/roco-studio/client/web$ npm run dev

```

## 调试
单步调试  启动后端
web      启动前端
在页面中发送MQTT消息

## 业务逻辑

外部控制系统将指令发布至 MQTT 服务器（Broker）的 Command 主题。主程序（Main） 作为网关维持与服务器的连接，并在订阅收到消息后对指令进行解析。随后，程序利用内部的指令映射表（Dispatch Map） 将抽象指令转化为具体操作，最终分发给相机、机械臂或 AGV 小车执行。

“核心逻辑：

    订阅：Main 程序连接 MQTT Broker 并监听 Command 主题指令。

    解析：接收并反序列化 JSON 格式的指令负载。

    分发：基于指令映射表（Mapping Table）查找对应回调。

    执行：将控制信号下发至底层硬件（相机/机械臂/AGV）。”

**前端与后端的关系**    
前端页面发送MQTT消息，通过Topic及Web页面（前端的那个位置--从每个组件的位置查找）来定位这个消息的Topic的方法及指令； 修改Topic
后端中driver_adapter.cpp来查找映射表--查看方法对应的操作。


## 核心定位：异构通信

驱动程序同时管理着两种完全不同的硬件，它们使用的语言（协议）不同：

    机械臂控制：使用 HRIF 协议，通过 TCP 连接。

    AMR 底盘控制：使用 JSON 格式，通过 WebSocket 连接。
1. 初始化与连接管理

    启动加载：程序启动时，会像查电话簿一样从配置文件读取 AMR 的地址（IP 和端口）。

    建立连接：调用 connectWebSocket 建立长连接。

    周期性心跳：一旦连接成功，驱动会开启“自动打卡”模式，定期发送 {"cmd":"getAgvStatus"}。这不仅是为了获取状态，也是为了确认连接依然活着。

    熔断保护：如果 WebSocket 没连上，所有的 AMR 相关操作会立即报错，不会浪费时间去尝试发送。

2. 指令下发路径 (Request Path)

这是一个典型的“翻译+转发”过程：

    MQTT 接收：外部（如 Web 页面）发送一个 JSON-RPC 请求到 /roco_faio_driver/cmd。

    路由分发：DriverAdapter 识别出这是 AMR 指令，分发给对应的 Handler（如远程遥控）。

    协议转换：Handler 将 MQTT 请求重新包装，添加 seq（序列号，用于标记是谁发的）和 cmd 参数。

    同步等待：调用 sendMessageAndWaitResponse。注意： 虽然 WebSocket 本质是异步的，但这里驱动层做了封装，让代码看起来像是在“阻塞”等待 AMR 回复，从而简化了业务逻辑。

3. 响应匹配机制 (The Matcher)

由于 WebSocket 是双向流，消息是乱序回来的。驱动如何知道哪个回复对应哪个请求？

    Worker 线程：有一个专门的后台线程在“轮询”消息。

    关键标识：收到消息后，检查 seq（序列号）。

    唤醒机制：如果 seq 匹配到了之前挂起的请求，就把数据填进去并“拍醒”那个正在等待的线程。

4. 状态推送 (Status Pushing)

除了回复指令，AMR 还会主动“说话”（例如报告当前的坐标、电量）：

    被动接收：如果收到的消息没有匹配的 seq，驱动会判定这是一条“状态广播”。

    转换上报：驱动将其封装为 amr_status 事件，通过 MQTT 发布到 /roco_faio_driver/event。

## 单步调试Roco_fair_driver的main.cpp
这个 `main.cpp` 是一个 **非 ROS 2 依赖的独立驱动程序入口**。它主要扮演“网关”或“中间件”的角色，负责在 MQTT 网络和具体的硬件驱动（如机械臂、相机、AGV小车）之间进行协议转换和指令分发。

以下是该文件的核心作用、详细运行流程以及推荐的调试断点位置。

### 1. 文件的核心作用

该程序的主要功能是将远程发来的 JSON-RPC 指令转换为本地硬件控制指令，并将硬件状态反馈回去。

* **配置管理**：通过环境变量（如 `MQTT_BROKER`, `MQTT_PORT`）配置运行参数。
* **通信桥梁**：
* **北向（上层）**：通过 MQTT 协议与上位机或调度系统通信。
* **南向（底层）**：通过 `DriverAdapter` 与具体的硬件（机械臂、WebSocket 设备等）通信。


* **指令处理**：订阅 MQTT 主题，接收 JSON 格式的指令，解析并执行。
* **状态上报**：将硬件的日志和事件通过 MQTT 推送到指定主题。

---

### 2. 详细运行流程 (Workflow)

程序的执行流程可以分为三个阶段：**初始化阶段**、**业务配置阶段**、**运行与消息循环阶段**。

#### 第一阶段：环境与基础组件初始化 (main 函数开头)

1. **日志启动**：初始化 `logger` 并设置级别为 DEBUG。
2. **获取配置**：程序读取环境变量（`MQTT_BROKER` 等）。如果环境变量不存在，则使用硬编码的默认值（如 `127.0.0.1`）。
3. **实例化核心对象**：
* 创建 `DriverConfig`（配置对象）。
* 创建 `DriverAdapter`（硬件适配器对象，这是业务逻辑的核心）。



#### 第二阶段：建立连接与定义行为

4. **连接 MQTT Broker**：调用 `mqtt::connect` 连接到消息服务器。
5. **定义指令处理逻辑 (关键)**：
* 调用 `mqtt::subscribe_raw` 订阅 `/roco_faio_driver/cmd` 主题。
* **定义 Lambda 回调函数**：这是程序接收到指令后的处理逻辑。
* 收到消息 -> 调用 `driver_adapter.handleRequest` 处理 -> 生成 JSON 响应 -> 发布到 `/response` 主题。

6. **启动 MQTT 线程**：创建一个新线程 `mqtt_thread` 专门跑 `mqtt::start_loop()`，确保消息接收不阻塞主线程。

#### 第三阶段：硬件初始化与阻塞

7. **硬件连接 (`Init` 函数)**：
* 设置日志和事件的回调函数（当 Adapter 产生日志时，自动通过 MQTT 发出去）。
* 建立与机械臂状态流、机械臂控制器、WebSocket 的连接。
* 打印连接状态日志。

8. **主线程阻塞**：调用 `mqtt_thread.join()`，主线程在此挂起，直到 MQTT 线程退出（通常程序会一直运行）。
9. **退出清理**：如果线程结束，断开 MQTT 连接并退出。

---

### 3. 单步调试的推荐节点 (Breakpoints)

为了有效调试，建议在以下关键位置打断点（点击 VS Code 行号左侧的红点）：

#### A. 检查启动配置 (Start-up)

* **位置**：`main` 函数开头，读取环境变量之后。
```cpp
// 约第 120 行
auto driver_config = std::make_shared<roco_faio_driver::DriverConfig>();

```


* **调试目的**：检查 `broker`、`port`、`topic_prefix` 等变量的值是否正确读取了你的 `launch.json` 或系统环境变量。

#### B. 检查 MQTT 连接 (Connection)

* **位置**：MQTT 连接判断处。
```cpp
// 约第 127 行
if (!mqtt::connect(broker, port, client_prefix, topic_prefix))

```

* **调试目的**：如果你发现程序启动就退出，在这里单步执行，看是否因为连不上 MQTT Broker 导致返回 `-1`。

#### C. 核心业务：指令接收与处理 (Critical Logic)

这是最重要的地方。你需要在 Lambda 表达式内部打断点。**只有当你通过 MQTT 发送了指令给程序时，这里的断点才会被触发。**

* **位置**：`subscribe_raw` 的回调函数内部。
```cpp
// 约第 138 行
LOG_INFO("Main", "[main] Received /command: {}", payload);

// 约第 142 行
roco_faio_driver::JsonRpc::JsonRpcResponse resp =
    driver_adapter.handleRequest(payload);

```

* **调试目的**：
1. 查看 `payload`：确认程序收到的 JSON 字符串是否完整、正确。
2. **“步入” (F11)** `handleRequest`：进入 `DriverAdapter` 内部，查看它是如何解析命令并控制硬件的。



#### D. 硬件初始化 (`Init` 函数)

* **位置**：`Init` 函数内部。
```cpp
// 约第 80 行
if (driver_adapter.connectToArmController()) {

```

* **调试目的**：查看程序是否成功连接到了真实的硬件（或仿真器）。如果连接失败，查看错误日志。

### 总结调试步骤

1. 在 **A** 处打断点，按 `F5` 启动。
2. 程序停下后，检查变量，按 `F5` 继续。
3. 程序会完成启动并进入等待状态（终端应该显示 "Mechanical arm controller connection success" 等日志）。
4. 在 **C** 处打断点。
5. 使用 MQTT 工具（如 MQTTX）向 `/roco_faio_driver/cmd` 发送一条 JSON 消息。
6. VS Code 会捕获断点，此时你可以单步调试具体的指令执行逻辑。


## Web文件下的dist与node_modules
这两个文件夹与 src 或 public 不同，它们是由命令自动生成的“非源码”文件夹。
1. node_modules（依赖库文件夹）
它的用处：

这个文件夹是项目的“仓库”，存放了项目运行所需的所有第三方库（Dependencies）。

    当你在代码里写 import { ref } from 'vue' 时，程序就是去这个文件夹里寻找 Vue 的源码。

    它通常非常庞大，包含成千上万个小文件。

它是如何产生的：

    产生命令：执行 pnpm install（或者 npm install）。

    原理：包管理器会根据项目根目录下的 package.json 文件列出的清单，从互联网（NPM 镜像站）下载对应的代码包，解压并放置在这个文件夹中。

    注意：这个文件夹通常不进入 Git 版本管理（会被 .gitignore 忽略），因为只要有 package.json，任何人都可以通过安装命令重新生成它。

2. dist（发布构建文件夹）
它的用处：

dist 是 Distribution（发布/分配） 的缩写。它包含了可以直接部署到服务器上的生产环境代码。

    压缩与优化：它将 src 里的 .vue、.ts、.scss 等源码，经过编译、压缩、混淆，变成了浏览器能直接识别的 .html、.js 和 .css。

    最终产物：当你把项目交给后端或者部署到云服务器时，只需要拷贝这个 dist 文件夹里的内容，而不需要拷贝整个项目源码。

它是如何产生的：

    产生命令：执行 pnpm build（或者 npm run build）。

    原理：构建工具（你项目中使用的是 Vite）会对源码进行“打包”：

        把多个 JS 文件合并。

        将高级语法转换为浏览器兼容性更好的代码。

        移除空格和注释以减小体积。

        将图片等资源进行路径重命名（防止缓存问题）。
