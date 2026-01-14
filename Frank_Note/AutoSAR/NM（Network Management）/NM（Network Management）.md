
## CAN NM Frame
#### 报文本质：广播式状态通告
CAN NM使用标准CAN数据帧进行 **广播传输** ，所有参与节点均可接收。它的ID由系统设计阶段分配，通常具有较高优先级（即较低的CAN ID），以保证在网络拥塞时仍能及时送达。

| 字节  | 含义                                        |
| --- | ----------------------------------------- |
| 0   | Network Management Data (NMD) —— 应用层自定义数据 |
| 1   | Control Bit Vector (CBV) —— 核心控制位         |
| 2   | Source Node ID —— 发送方逻辑地址                 |
| 3   | Destination/Group ID —— 目标或组播地址           |
| 4-7 | Reserved / User Extension —— 可扩展区域        |

#### CBV(Control Bit Vector)
真正决定行为的是 **Byte 1：控制位向量（CBV）** 。它在一个字节内编码了多个关键标志：

| Bit | 名称                             | 功能说明          |
| --- | ------------------------------ | ------------- |
| 7   | RMR (Repeat Message Request)   | 是否请求重复发送NM报文  |
| 6   | -(Reserved)                    | 必须置0          |
| 5   | PNI (Partial Network Info)     | 是否属于局部网络通信    |
| 4   | CSyn (Coordinator Sleep Ready) | NM协调器已准备睡眠    |
| 3   | Awu (Active Wakeup)            | 主动唤醒标志（非被动监听） |
| 2-0 | SI[2:0] (State Information)    | 当前NM状态编码      |
- **RMR == 1**:这是“求生信号”。刚唤醒的节点会设置此位，告诉全网：“我刚起来，请大家别睡！”其他节点即使本地无请求，也应转发NM报文，形成链式传播。
- **PNI**:支持“局部网络”（Partial Network）。例如，仅开启空调时，只需唤醒空调相关的ECU，而不必点亮整个动力域。这大大提升了能效灵活性。
- **SI[2:0]** :状态编码，对应状态机中的 `Repeat Message` 、 `Normal Operation` 等状态，用于远程监控。
*==在调试过程中，若发现某节点无法唤醒其他模块，请优先检查其发出的NM报文中 **RMR是否正确置位** ，以及 **Source Node ID是否配置正确**

## 状态机跳转
CAN NM的状态机并非凭空设计，而是围绕“ **检测活动 → 维持唤醒 → 安全关闭** ”这条主线构建的。每个节点都在本地运行这套FSM，并根据接收到的NM报文和自身需求做出决策。
![[Pasted image 20260114140245.png]]
#### 1.<mark style="background: #BBFABBA6;">Bus-Sleep Mode</mark> （彻底休眠）
- [ ] **行为特征**：CAN控制器进入 **只听模式（Listen Only Mode）** ，不发送任何报文，CPU可进入深度睡眠。
- [ ] **何时进入**：成功完成 `Prepare Bus-Sleep` 阶段且未被唤醒。
- [ ] **如何退出**：
	-  检测到有效的NM报文
	-  本地有网络请求（如IO中断，定时任务触发）
**进入条件**

| current status               | condition                                                |
| ---------------------------- | -------------------------------------------------------- |
| ```Prepare Bus SLeep Mode``` | `1.T_WAITBUSSLEEP TimeOut && Get_Ready_Sleep() == FALSE` |
==*在此状态下，节点只能被动监听唤醒信号，不能主动发起通讯

___
#### 2.<mark style="background: #D2B3FFA6;">Prepare Bus-Sleep Mode</mark> （准备入睡 最后确认）
- [ ] **持续时间**：由参数 `NmWaitBusSleepTime` 控制（典型值2–5秒）深度睡眠。
- [ ] **关键动作**：
	-  停止发送NM报文
	-  监听是否有新的NM帧出现
- [ ] **结果分支**：
	-  若期间收到任何有效NM报文 → 立即返回 `Network Mode`
	-  若全程安静 → 进入 `Bus-Sleep`
**进入条件**

| current status      | condition        |
| ------------------- | ---------------- |
| `Ready Sleep State` | `1.T_NM TimeOut` |
==*提供一个“安全窗口”，防止因微小延迟导致误判休眠，提升系统鲁棒性* 
___

#### 3.Network Mode （网络活跃期，细分为三个子状态）
- 3.1<mark style="background: #FF5582A6;">Repeat Message State</mark> ：紧急广播期
	- [ ] **触发条件**：上电初始化、收到外部唤醒、本地有高优先级请求
	- [ ] **行为**：
	     -  以 `NmMessageCycleTime` （如200ms）周期发送NM报文
	     -  设置 **RMR=1**
	- [ ] **目的**：强力宣告“我醒了！”，推动全网同步激活
**进入条件**: 

| current status               | condition                                                                                                 |
| ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `Normal Operation State`     | ```1.Get_ReqRepeatbit() == TRUE```<br>```2.Get_Is_Received_Flag() == TRUE && Get_RxRepeatbit() == TRUE``` |
| `Ready Sleep State`          | ```1.Get_ReqRepeatbit() == TRUE```                                                                        |
| ```Bus Sleep Mode```         | `1.NM_Wakeup_flag == NM_RX_MESSAGE`<br>`2.NM_Wakeup_flag == NM_APPL_LOCAL_WAKEUP`                         |
| ```Prepare Bus SLeep Mode``` | `1.NM_Wakeup_flag == NM_RX_MESSAGE` <br>`2.NM_Wakeup_flag == NM_APPL_LOCAL_WAKEUP`                        |
==*在OTA升级或UDS诊断刷写场景中，常手动触发此状态，确保所有依赖模块均已上线。*

---

- 3.2<mark style="background: #ABF7F7A6;">Normal Operation State</mark> ：稳定运行期
	- [ ] **进入条件**: RMR超时（ `NmRepeatMessageTime` ，通常3秒）或被清除
	- [ ] **行为**：
	    -   继续发送NM报文，但 **RMR=0**
	    -  仅通报自身状态（Node ID、PNI等）
	- [ ] **作用**: 维持“我还活着”的存在感，供其他节点感知
**进入条件**

| current status         | condition                                             |
| ---------------------- | ----------------------------------------------------- |
| `Repeat Message State` | `1.T_REPEATMSG TimeOut && Get_Ready_Sleep() == FALSE` |
| `Ready Sleep State`    | ```1.Get_Ready_Sleep() == FALSE```                    |

---
- 3.3<mark style="background: #FFB86CA6;">Ready Sleep State</mark> ：准备退场
	- [ ] **进入条件**： 本地无网络请求 + 所有远程节点也无活动迹象
	- [ ] **行为** :
		- 停止发送NM报文
		- 启动 `NmWaitBusSleepTime` 计时器
	- [ ] **最终去向**：倒计时结束 → 进入 `Prepare Bus-Sleep`
**进入条件**

| current status           | condition                                            |
| ------------------------ | ---------------------------------------------------- |
| `Repeat Message State`   | `1.T_REPEATMSG TimeOut && Get_Ready_Sleep() == TRUE` |
| `Normal Operation State` | ```1.Get_Ready_Sleep() == TRUE```                    |
==*任一时刻收到NM或本地请求 -> 回归Network Mode*

---

## 传输策略
#### 1.周期发送 + 主函数轮询

#### 2.RMR中继机制：唤醒的“雪崩效应”

#### 3.发送抑制机制

#### 4.随机启动偏移：错峰发送防止冲突
