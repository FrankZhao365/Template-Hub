
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
|     |                                |               |
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
## 状态机切换流程详解
1）开机—>BusSleepMode，
当KL30上电或当前节点处于休眠情况下被按照唤醒上其他节点唤醒时会执行初始化的操作，CanNM_Init()初始化完成后，CanNM状态机变为第一个模式BusSleepMode，即恢复睡眠模式。

在BusSleep模式下会判断唤醒条件是否满足，如果不满足则停留在睡眠模式下，直到满足休眠或被唤醒条件。

2）BusSleepMode—>RepeatMessage状态
如唤醒条件满足，会根据当前是主动唤醒还是被动唤醒分别调用接口CanNM_NetworkRequest()或CanNM_PassiveStartUp()，将状态切换至进入RepeatMessage状态，这里需要注意，RepeatMessage状态后会根据唤醒的原因是主动唤醒还是被动唤醒决定是否进入快发状态：

①主动唤醒情况：以T_ImmediateNmCycleTime的周期快速发送N_ImmediateNm_Times次NM，目的是快速唤醒唤醒上其他节点，CBV中的主动唤醒位置为1；

②唤醒唤醒情况：说明唤醒上已经有其他节点苏醒，不需要自己去唤醒唤醒上其他节点，只需要以T_NM_MessageCycle 周期发送NM报文即可。

在RepeatMessage状态下停留的时间为T_RepeatMessage，超时后将切换到后续状态。

3）PrepareBusSleepMode—>RepeatMessage状态
同样在当前节点上PrepareBusSleepMode准备休眠模式时同时处于该状态停留T_WaitBusSleep时间，如在此期间检测到主动唤醒或激活唤醒需求同样会调用CanNM_NetworkRequest()或CanNM_PassiveStartUp()，将状态切换至RepeatMessage State，后续在RepeatMessage状态中执行的操作与“2）BusSleepMode—>RepeatMessage State”中一致，此处不再赘述。

4）RepeatMessage State—>NormalOperation State
当状态机在RepeatMessage State停留时间达到T_RepeatMessage后，如果ECU需要保持唤醒状态，会调用接口CanNM_NetworkRequest()进入NormalOperation State即常规运行状态。

5）NormalOperation State—>RepeatMessage State
在NormalOperation State状态下有两种情况会切换回RepeatMessage State：
①如ECU收到其他节点NM报文中的重复消息位位置1了；
②自身节点有重复消息需要，通过调用接口CanNM_RepeatMessageRequest()来实现，在这种情况下，本节点发送的NM报文中重复消息位也置1。

6）NormalOperation State—>ReadySleep State
处于经常运行状态下，如本节点没有与其他节点交易的需求，会通过接口CanNM_NetworkRelease()释放网络管理请求，将状态切换到ReadySleep State切换休眠状态。

7）ReadySleep State—>NormalOperation State
在准备休眠情况下，如存在本地唤醒源或与其他节点的通信需求，会调用接口CanNM_NetworkRequest()将状态重新切换回NormalOperation State。

8）RepeatMessage State—>ReadySleep State
在RepeatMessage State状态下，停留时间达到T_RepeatMessage后，如果ECU不需要保持唤醒状态，会调用接口CanNM_NetworkRelease()进入ReadySleepState即准备休眠状态。

9）ReadySleep State—>RepeatMessage State
与NormalOperation State切换回RepeatMessage State情况一致，同样有两种可能原因：
①如ECU收到其他节点NM报文中的重复消息位位置1了；
②自身节点有重复消息需要，通过调用接口CanNM_RepeatMessageRequest()来实现，在这种情况下，本节点发送的NM报文中重复消息位也置1。

10）ReadySleep状态—>PrepareBusSleepMode
在ReadySleep状态下，如果没有本地或网络唤醒请求，且T_NmTimeout超时后，NM状态机会切换到PrepareBusSleepMode，同时启动T_WaitBusSleep。

11）PrepareBusSleepMode—>BusSleepMode
在T_WaitBusSleep达到设定阈值前，若仍没有本地或网络请求，则进入休眠休眠模式，若有本地或网络请求则通过CanNM_NetworkRequest()或CanNM_PassiveStartUp()切换回RMS状态。

___
## 传输策略
#### 1.周期发送 + 主函数轮询
AUTOSAR采用经典的“主函数调度”模型。NM模块通过定期调用 `CanNm_MainFunctionTx()` 判断是否满足发送条件。
#### 2.RMR中继机制：唤醒的“雪崩效应”
当一个节点发送带RMR=1的NM报文时，其他节点即使本地无请求，也应 **立即加入发送行列** ，形成“接力式”唤醒。
这种机制极大增强了弱信号或远距离节点的唤醒成功率，尤其适用于大型车身网络。
#### 3.发送抑制机制

#### 4.随机启动偏移：错峰发送防止冲突

___
[超详细版AUTOSAR CAN NM报文格式与传输策略-CSDN博客](https://blog.csdn.net/weixin_42298254/article/details/156477606?ops_request_misc=elastic_search_misc&request_id=7a1b237ad4ca38f3707a805619a7587c&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-1-156477606-null-null.142^v102^pc_search_result_base9&utm_term=CAN%20NM%E6%8A%A5%E6%96%87)
[Autosar通信入门系列07-CanNM状态机切换详解-CSDN博客](https://blog.csdn.net/initiallizer/article/details/134894615)

