#ISO14229 #ISO11898 #ISO5765 
# 报文类型
- [ ] 单帧 SingleFrame
- [ ] 首帧 FirstFrame
- [ ] 连续帧 ConsecutiveFrame
- [ ] 流控帧 FlowControl
**每个诊断帧的第一个字节的高4bit用于描述该帧的类型，即该帧属于上述四种中的哪一种**
![[Pasted image 20260108165038.png]]

# SingleFrame：
- 报文结构：
	-  第一个字节的bit7:4: **0 0 0 0*
	-  第一个字节的bit3:0: **SF_DL**
	
- 用于长度不超过7个字节的诊断命令或响应。SingleFrame用于下面这种简单的场景：当诊断报文长度小于等于7时，再加上一个字节的PCI控制信息就是小于等于8，可以在一帧CAN报文上传输，所以不需要进行分包。此时数据域的第一个字节高4bit值为0000，标识这是一个帧SingleFrame，低4bit是SF_DL，即DataLength，描述后面有几个字节。如果有没有使用的字节，通常要用0x55或0xAA来填充，因为这两个值的二进制表述其实就是01010101和10101010，这样在CAN总线上可以让信号跳变得更频繁一些，不会出现长时间电平不变的情况

# FirstFrame/ConsecutiveFrame/FlowControl
FirstFrame，ConsecutiveFrame，FlowControl用于传输长度大于7个字节的诊断命令或响应。
## FirstFrame
- 报文结构：
	-  第一个字节的bit7:4: **0 0 0 1**
	-  第一个字节的bit3:0 + 第二个字节: **FF_DL** （DataLength，描述后面字节数,包括在FirstFrame中和在ConsecutiveFrame中的所有报文内容字节数）
	
## ConsecutiveFrame
- 报文结构：
	-  第一个字节的bit7:4: **0 0 1 0**
	-  第一个字节的bit3:0: **SN** （sequence number）

## FlowControl
- 报文结构：
	-  第一个字节的bit7:4: **0 0 1 1**
	-  第一个字节的bit3:0: **FS** (FlowStatus) 用于指示数据传输的控制状态
		- 0: CTS（ContinueToSend）- 允许发送端继续发送ConsecutiveFrame
		- 1: WT (Wait) - 要求发送端等一会再发送ConsecutiveFrame,再下次允许sender发送        ConsecutiveFrame时，receiver再发一个FlowStatus=0的FlowControl
		- 2: Overflow (OVFLW) - receiver因为资源问题无法接收sender发送的数据，则发送一个FlowStatus=2的FlowControl
	-  第二个字节：**BS** (Block Size) 允许连续发送的数据块大小（帧数）
		- **BS=0**：允许连续发送无限帧，直到数据传输完成，无需等待新的流控帧
	-  第三个字节：**Stmin** (Separation Time minimum) 帧与帧之间的最小间隔时间
		- **STmin=0**：帧与帧之间不需要等待，可以以最快速度连续发送



# 流程示例
## 不分段传输的诊断服务举例：

request **0**2 10 03 55 55 55 55 55

response **0**6 50 03 00 32 01 F4 AA
## 分段传输的诊断服务举例：

这是一个读取DTC的命令和应答。

**03** 19 02 08 55 55 55 55 （诊断仪发送的<font color="#ff0000">SingleFrame</font>的request）

**10 33** 59 02 19 01 00 07 （ECU以<font color="#f79646">FirstFrame</font>开始传输的response）

**30** **00 00** 55 55 55 55 55 （诊断仪发送的<font color="#ff0000">FlowControl</font>）

**21** 09 03 05 02 09 05 04 （ECU发送的<font color="#f79646">ConsecutiveFrame</font>）

**22** 07 09 05 06 06 09 05 （ECU发送的ConsecutiveFrame）

**23** 08 03 08 07 01 05 08 （ECU发送的ConsecutiveFrame）

**24** 07 01 06 08 07 01 0C （ECU发送的ConsecutiveFrame）

**25** 08 07 01 0D 08 07 03 （ECU发送的ConsecutiveFrame）

**26** 07 09 08 01 01 09 09 （ECU发送的ConsecutiveFrame）

**27** 01 07 09 AA AA AA AA （ECU发送的ConsecutiveFrame，此时传输结束）