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
roco@roco:~/backup/roco-studio_12_15/roco-studio/client/web$ npm run build
roco@roco:~/backup/roco-studio_12_15/roco-studio/client/web$ npm run dev

```

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
