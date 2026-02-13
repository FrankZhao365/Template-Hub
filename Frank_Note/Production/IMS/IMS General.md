
Signal：

| Sender | Received | Signal             | name        | ID    |     |
| ------ | -------- | ------------------ | ----------- | ----- | --- |
| CGW    | ATAC     | IHU_24_ATACArmSW   | 中控屏防盗报警设置开关 | 0x4DC |     |
| BDM    | ATAC     | MstVehMod          | 整车模式        | 0x2EA |     |
| BDM    | ATAC     | KeySts             | 钥匙状态        | 0x391 |     |
| BDM    | ATAC     | ArmingSts          | 整车设防信号      |       |     |
| ATAC   | BDM      | ATAC_1_SetFeedback | 反馈状态信号      | 0x404 |     |

IHU_24_ATACArmSW
# NM
## NM Variable


| name           | explain |
| -------------- | ------- |
| NM_Wakeup_flag | 唤醒的原因   |
|                |         |

## NM State

| NM State            | CN name  |
| ------------------- | -------- |
| NM_SHUTDOWN         |          |
| NM_BUSSLEEP         | 总线睡眠状态   |
| NM_REPEAT           | 重复消息状态   |
| NM_NORMAL           | 正常操作状态   |
| NM_READY_SLEEP      | 准备睡眠状态   |
| NM_PREPARE_BUSSLEEP | 准备总线睡眠状态 |

