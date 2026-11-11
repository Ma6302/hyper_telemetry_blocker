# Hyper 遥测与采集拦截器

面向 **HyperOS / 小米澎湃 OS** 的 KernelSU 模块：拦截系统底层的日志、遥测与隐私采集，减少后台资源消耗与存储写入。

> 当前实装范围：**MiSight 性能与隐私采集**（关闭采集进程、关闭开关、冻结 `event.db` 增长）＋**可选日志压制**。
> 定位是「可验证、可回滚、卸载零残留」，不是「一键优化」。

## 它到底做了什么

### 1. MiSight 采集关闭（默认开启）

HyperOS 有一套叫 **MiSight** 的采集服务：`/system_ext/bin/misight` 常驻，把系统内上报的卡顿事件（渲染卡顿、低帧率、操作无响应等）写进 `/data/miuilog/misight/event.db`。阈值很低（动画卡顿 96 ms、应用卡顿 167 ms 起记），一次抖动还会拆成多条，所以库里很容易堆到十万级记录。

模块做的事：

| 动作 | 实现 |
|---|---|
| 停止采集进程 | `setprop ctl.stop misight`（该服务在 `init.misight.rc` 里**没有** `critical` 标记，可安全停止，不会触发重启循环） |
| 关闭开关 | `persist.sys.misight.detectswtich=0`、`persist.sys.perfdebug.monitor.enable=0` |
| 降低日志噪音 | `persist.log.tag.misight=W`（原本是 `I`） |
| 冻结数据库 | 唯一写者就是那个采集进程，进程停掉后 `event.db` 不再增长（保留历史数据，不删库） |

### 2. 日志压制（可选，默认 `safe`）

`config.conf` 里的 `log_suppress` 三档：

- `off`：不碰日志
- `safe`（默认）：10 条经过「读取点核实」的属性 —— `persist.traced.enable=0`（停 perfetto 的 traced / traced_probes、`perfetto.rc` 里 `on property:persist.traced.enable=0` 直接 stop）、7 条 `persist.vendor.ims.*`（`lib-imsdpl.so` / `ims_rtp_daemon` 读取）、`log.tag.stats_log` 与 `log.tag.DeviceStatisticsService` 降为 SILENT
- `full`：另加 `persist.sys.log.lfeh*` / `ro.logd.size.stats` / `persist.sys.dump` 等（`MiBugReportOS4.apk`、`CatchLog.apk`、`miui-services.jar` 读取）

**实测收益要说清楚**：日志侧的 CPU 成本主要是 `logd`，稳态约 **2–3% 单核**；这台机器上 60% 的日志量来自 vendor 的显示/传感器组件（`qdgralloc`、`SDM`、`TRUETONE`、`libsensor-parseRGB`），**没有可写开关能关** —— 那些是模块管不到的。所以这一档是「顺手省一点 + 少写盘」，不是省电主力。

## 安全设计

- **不用只读挂载**：实测本机 `mount -o remount,ro,bind` 直接 `Device or resource busy`，`mount -o bind,ro` 会静默变成 rw，所以「冻结库」不依赖 bind mount，从源头上避免了「挂载失败把你数据锁死」。
- **备份在模块目录之外**：状态保存在 `/data/adb/<id>_state/`。重刷模块时 KernelSU 会整目录替换模块目录，备份若放在里面会被冲掉（v1.0.2 修的就是这个坑）。
- **备份不会被自己污染**：如果刷入时模块已生效（抓到的"原值"等于模块的目标值），会判定为可疑备份并写入内置兜底原值，而不是把"已关闭"当成原值存下。
- **看门狗**：开机后 1–5 分钟内系统有一次把属性写回原值的行为（实测），所以开机后按 `settle_schedule`（30/90/180/300/600/1200 秒）密集复查，之后 30 分钟一次；纠正动作都记进 `logs/tamper.log`。
- **卸载还原**：`uninstall.sh` 逆序还原属性（原本不存在的属性直接删除）、撤掉日志压制、重新拉起采集进程，自检后才退出，最后清掉状态目录。

## 安装

1. 下载最新 Release 的 zip
2. KernelSU / SukiSU Manager → 模块 → 从本地安装 → 选中 zip
3. 重启一次（首次生效必须重启；此后看门狗的纠正是即时的，不需要重启）

**升级旧 id 版本（`hyper_misight_off`）的用户**：先删掉旧模块目录 `/data/adb/modules/hyper_misight_off`，否则每次开机会被重复执行一套脚本。

## 在线更新

`module.prop` 里有：

```properties
updateJson=https://raw.githubusercontent.com/Ma6302/hyper_telemetry_blocker/main/update.json
```

`update.json` 字段：`version` / `versionCode` / `zipUrl` / `changelog`。有新版本时 KernelSU 模块页会出现「更新」按钮，点击自动下载并安装。

## 配置

`/data/adb/modules/hyper_telemetry_blocker/config.conf`：

```properties
enabled=true          # 总开关
stop_service=true     # 停 misight 进程
set_props=true        # 写属性
freeze_db=keep        # keep（不动库）| mask（tmpfs 遮住）| off
watchdog=true         # 看门狗
watchdog_interval=1800
log_suppress=safe     # off | safe | full
```

改完重跑模块或重启生效。只读状态检查：`sh /data/adb/modules/hyper_telemetry_blocker/action.sh`（也可以从管理器里的「操作」按钮触发）。

## 卸载

管理器里移除模块 → 重启即可。会还原所有被改过的属性、重新拉起采集进程、删除状态目录，不留残余。

## 边界

- 只针对本机实测过的 HyperOS 版本（`OS4.0.6.0.XPCCNXM`，Android 17，Xiaomi 17 / `25113PN0EC`）。其他机型的属性名与 init 服务定义可能不同，**没有做过兼容性保证**。
- 不涉及任何联网/上报拦截（MiSight 的上报通道本来就是空表 + 写者已死）。
- 不修改系统分区、不注入 Zygisk、不 hook 进程。
- 日志压制会减少诊断现场（trace/ANR 的辅助信息变少）；如果你正在排查性能问题，先把它设为 `off`。

## 许可

仅供个人学习与自用，按现状提供，不承担任何责任。刷机有风险，请自行备份。
