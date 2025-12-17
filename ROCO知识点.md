# MQTT

MQTT (Message Queuing Telemetry Transport) 是目前物联网 (IoT) 领域最主流的通信协议。

如果说 HTTP 是互联网世界的“普通话”（适用于网页浏览、文件下载），那么 MQTT 就是物联网世界的“电报”——它极其轻量、省电、带宽占用极低，且专为网络不稳定的环境设计。

在你的 roco-studio 项目中，MQTT 是连接前端界面（Web/Tablet）、后端逻辑（ROS2）和底层硬件（Driver）的核心神经系统。
1. 核心架构：发布/订阅 (Pub/Sub) 模型

MQTT 与 HTTP 最大的不同在于它不是“请求-响应”模式，而是发布/订阅模式。
角色分工：

    Publisher (发布者)：发出消息的人。

        在你的项目中：roco_faio_driver (发布机械臂状态)、camera_server (发布采集状态)。

    Subscriber (订阅者)：接收消息的人。

        在你的项目中：client/web (显示设备状态)、mqtt_server (监听控制指令)。

    Broker (代理/服务器)：邮局或转发中心。

        发布者和订阅者互不认识，也不直接连接。大家只连接 Broker。

        Broker 负责接收消息，并根据“主题”把消息分发给所有订阅了该主题的人。

        常见软件：Mosquitto, EMQX, HiveMQ。

2. 关键概念详解
A. Topic (主题) —— 消息的地址

Topic 是一个用斜杠 / 分隔的字符串，类似于文件路径。

    示例：

        roco/arm/status （机械臂状态）

        roco/camera/capture （拍照指令）

通配符 (Wildcards)：订阅者可以使用通配符一次订阅多个主题。

    + (单层通配符)：roco/+/status 匹配 roco/arm/status 和 roco/camera/status，但不匹配 roco/arm/left/status。

    # (多层通配符)：roco/# 匹配 roco 下的所有层级所有主题。

B. Payload (负载) —— 消息的内容

MQTT 对内容格式没有限制，可以是纯文本、二进制图片、加密数据。

    在你的项目中：主要使用 JSON 字符串（例如 {"angle": 90, "speed": 0.5}），方便前端 JS 和后端 Python 解析。

C. QoS (服务质量) —— 消息可靠性分级

这是 MQTT 适应不稳定网络的杀手锏。

级别	描述	别名	特点	适用场景
QoS 0	最多一次	Fire and Forget	发出去就不管了，丢了就丢了。	传感器高频数据（丢一两个包无所谓）。
QoS 1	至少一次	At least once	保证送达，但可能重复收到。需要 ACK 确认。	最常用。关键指令（如“停止”命令）。
QoS 2	只有一次	Exactly once	保证送达且不重复。握手过程复杂，慢。	计费系统、银行转账。

D. Keep Alive (心跳保活)

客户端和 Broker 之间会定时发送 Ping 包。如果 Broker 超过一定时间（如 60秒）没收到心跳，就会认为客户端断线了。
3. 两个针对 IoT 的“神技”

在你的 roco_faio_driver 中，这两个特性非常重要：
1. Retained Messages (保留消息)

    问题：前端 Web 刚打开时，机械臂可能 10 分钟前就发过状态了，Web 此时是一片空白，必须等下一条消息。

    解决：发布消息时标记 Retain=True。Broker 会把这条消息“贴在墙上”。任何新来的订阅者连接成功后，Broker 会立刻把这条最后已知的好消息发给它。

    场景：设备在线状态、固件版本号。

2. LWT (Last Will and Testament, 遗嘱消息)

    问题：机械臂突然断电或网线被拔，它来不及告诉服务器“我要下线了”。服务器还以为它活着。

    解决：客户端在连接 Broker 时，预先留下一封“遗嘱”（比如 Topic: status, Payload: offline）。

    机制：如果客户端非正常断开（心跳超时），Broker 会自动把这封遗嘱发给所有订阅者。

    场景：你的 Web 界面能瞬间把设备状态变红（显示离线），而不是傻傻地等。

4. 结合你的代码流程 (roco_faio_driver)

让我们回到你刚才提供的 C++ 代码逻辑，看看 MQTT 是如何工作的：

    连接阶段：

        mqtt::connect(...)：驱动程序作为 Client 连接 Broker。此时它可能设置了 LWT（遗嘱），告诉 Broker：“如果我挂了，告诉大家 roco_faio_driver 离线了。”

    指令接收 (Subscribe)：

        订阅 /roco_faio_driver/cmd。

        当 Web 前端点击“移动”按钮 -> Web 发送 JSON 到 Broker -> Broker 转发给 C++ 驱动 -> 触发回调函数 -> 驱动解析 JSON 控制机械臂。

    状态反馈 (Publish)：

        驱动执行完命令，得到结果。

        发布到 /response。

        Web 前端订阅了这个 Topic，界面上显示“执行成功”。

5. 为什么你的项目选择 MQTT 而不是 HTTP？

    双向通信：HTTP 是单向的（客户端请求 -> 服务器回复）。服务器很难主动给客户端发消息（虽然有 WebSocket，但 MQTT 更适合设备）。MQTT 天生就是双向的。

    带宽极小：MQTT 的协议头最小只有 2 字节。对于车联网（4G/5G 流量昂贵且有时信号差）场景，这比 HTTP 的几十个字节的 Header 高效得多。

    解耦：前端 Web 不需要知道底层的 C++ 驱动的 IP 地址，也不需要知道它是不是由 Python 写的。大家只需要约定好 Topic 即可。

# 日志级别

级别	常见标记	含义	你的代码中的例子
Debug	[debug]	调试信息。非常琐碎的细节，只有开发找 Bug 时才看。	logger::set_level(spdlog::level::debug);
Info	[info]	一般信息。程序正常运行的关键节点，类似“里程碑”。	LOG_INFO("Main", "机械臂控制器连接成功");
Warn	[warn]	警告。有点不对劲，但程序还能跑，不会崩。	LOG_WARN("Main", "WebSocket连接失败");
Error	[error]	错误。出大事了，某个功能失效了。	LOG_ERROR("Main", "Failed to connect to MQTT");
Fatal	[fatal]	致命。程序彻底挂了，必须退出了。	(你的代码中暂时没用到)
