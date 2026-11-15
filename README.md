# Samsung 45W 充电协议切换死锁（IBUSUCP 断流）
    发现日期：2026-11-15
    作者：candydads
    摘要S25 Ultra 使用 45W PD/PPS 充电器时，在充电协议协商过程中变更快速充电开关或退出旁路充电，
    会触发 PMIC 输入欠流保护（IBUSUCP），充电反复断开重连，无法自行恢复，需物理拔插线缆。
    Framework 在 `pass_through` 置 0 后 **56 ms** 即下发快充请求，此时 VBUS 电流尚未建立。
#大白话理解：
----------------------------------------------------------------------------------------------------------------------------
    🚗 场景一：为什么会“打碎齿轮”（断流BUG）？
    快速充电 = 挂着 5档（高功率），跑得飞快。
    旁路充电 = 挂着 空档（高功率供电，但低电流不进电池，发动机在空转）。
    触发BUG：你从空档，瞬间强行挂回 5档。变速箱直接被打碎（底层芯片触发欠流保护），
    车疯狂顿挫（间歇性断流），而且熄火后短时间根本打不着（持续触发BUG）。
    🛠️ 场景二：你的“神仙救车法”（卡BUG状态）
    降频动作：切到空档（旁路）后，你强行关了快充。这就相当于踩死刹车，把车速强制降到最慢（低功率旁路）。
    卡出BUG：在这个极慢的速度下，你把快充开关按亮。按理说该加速了，但系统脑子卡住了（显示 USB 2.0，设备管理器里消失）。
    它处于一个“我想挂5档，但变速箱死机了，只能维持1档走”的状态。此时，车绝不顿挫（怎么切都不掉电）。
    💥 场景三：终极点火（模拟断电通电）
    模拟断电：既然变速箱死机了，你直接拔了车钥匙（unplug），让全车断电，强制芯片清空刚才的“死机”记忆。
    通电：重新插钥匙打火（reset）。因为你断电前，快充开关早就按亮了，所以打火的瞬间，汽车直接挂上 5档（45W 2.0），而且因为这一次没有中间切换的顿挫，变速箱完美咬合，一路狂飙！
----------------------------------------------------------------------------------------------------------------------------
## 环境

| 项 | 值 |
|---|---|
| 设备 | Samsung Galaxy S25 Ultra |
| 系统 | One UI（Android 15） |
| 充电器 | 45W PD/PPS 原装 |
| PMIC | sm5443 |
| 触发入口 | `settings put system pass_through` / `settings put system adaptive_fast_charging` |
----------------------------------------------------------------------------------------------------------------------------
## 复现步骤

**场景 A — 开启旁路充电**

1. 45W 充电器充电中，确认显示「超级快充 2.0」
2. 执行 `settings put system pass_through 1`

**结果**：充电反复断开重连，提示音持续，`current_now` 为负值，长时间无法恢复。

**场景 B — 切换快速充电开关**

1. 45W 充电中
2. 设置 → 电池 → 关闭「快速充电」→ 再打开

**结果**：与场景 A 相同。

## 对照组

| 条件 | 结果 |
|---|---|
| 同一充电器接其他品牌手机 | 正常 |
| 卸载所有第三方 App | 仍复现 |
| 快速充电保持关闭 | 不断流 |
| 物理拔插 USB-C 线 | 恢复，但断流一次 |

说明问题与本机充电子系统相关，与第三方 App、线材无关。

## 日志证据

**内核日志**

```
sm5443-PD-DC sm_dc_ibusucp_work: IBUSUCP DETECTED
sm5443-PD-DC check_error_state: error status retry, wait 2sec
sec_direct_chg_set_switching_charge: charger_mode(Charging-Off->Buck-Off)
```

**Framework 日志**

```
01:43:28.960  BatteryService: PassThrough Settings = false
01:43:28.968  BatteryService: success to set PassThrough sysfs as false
01:43:29.016  BatteryService: AdaptiveFastCharging Settings = true    ← +56 ms
01:43:29.019  BatteryService: success to set AFC sysfs as true
01:43:43.283  ACTION_POWER_DISCONNECTED
01:43:45.810  ACTION_POWER_CONNECTED
01:43:47.050  ACTION_POWER_DISCONNECTED
```

退出旁路与请求快充间隔仅 **56 ms**。

## 原因分析

触发条件：**快充开关变更 × 协议协商进行中**。

```
pass_through 0
   ↓ PMIC 从直供切回恒流，VBUS 电压/电流建立中
+56 ms
   ↓ AFC sysfs = 1，系统请求 PPS 高压档
PMIC 立即拉升输出电压
   ↓ 输入电流未跟上
IBUSUCP → Charging-Off（保护态）
   ↓ 2 s 后重试，仍在协商窗口内
循环断流
```

**规律验证**

| 操作时机 | 是否断流 |
|---|---|
| 旁路状态（电池断开、无协商） | 否 |
| 协议协商窗口内 | 是 |
| 协议稳定后 | 视是否触发重新协商而定 |

## 影响

- 45W 充电器下，旁路充电功能不可用
- 手动切换快速充电开关会中断当前充电会话
- 需物理拔插线缆才能恢复

## 期望修复

1. **协商窗口保护**：`pass_through` 从 1 变为 0 后 2–3 秒内，拒绝或排队处理 AFC / 超级快充的 sysfs 写入
2. **一次性协商**：协议复位前已确立的快充状态，复位后应直接按目标功率单次协商，而非先基础功率再升压
3. **降档重试**：IBUSUCP 重试策略改为「降档后渐进升压」，而非固定 2 秒后原功率重试

## 复现命令

```bash
# 触发
settings put system pass_through 1

# 观测
dmesg | grep -iE "ibusucp|sm5443"
dumpsys battery | grep -E "status|plug"
cat /sys/class/power_supply/battery/current_now
```

---

