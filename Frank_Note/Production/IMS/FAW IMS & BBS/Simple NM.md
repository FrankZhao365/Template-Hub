# NM Statue Flow

![[Pasted image 20260115105620.png]]
# switch condition
(1) 定时器 T<sub>Active_Min</sub>超时时，预休眠条件不满足。
(2) 定时器 T<sub>Active_Min</sub>超时时，预休眠条件满足。
(3) 预休眠条件满足。
(4) 预休眠条件不满足。
(5) 定时器 T<sub>NMTimeout</sub>超时。
(6) 本地唤醒且 ECU 有通信需求，或收到 NM 报文。
(7) 定时器 T<sub>WBS</sub>超时。
(8) 本地唤醒且 ECU 有通信需求，或收到任何报文（仅支持 CAN 唤醒的 ECU）。
(9) 关闭 NM（如通过诊断服务 28h 关闭 NM 通信）。对于使用电源模式总线信号的控制器，当通过诊断服务 28h 关闭 NM 通信时，禁止休眠，维持唤醒状态直到诊断服务 28h 失效。
(10)开启 NM（如通过诊断服务 28h 开启 NM 通信）。