# 更新日志

## v1.2.1（2026-11-11）

- 接入**在线更新**：`module.prop` 增加 `updateJson`，指向本仓库的 `main/update.json`。KernelSU 模块页检测到更高 `versionCode` 时出现「更新」按钮，点击即从 GitHub Release 下载 zip 并自动安装。
- 脚本与属性清单相对 v1.2.0 **一行未改**，不需要为升级重启。

## v1.2.0（2026-11-11）

- **更名**：`hyper隐私采集关闭` → **Hyper 遥测与采集拦截器**；id `hyper_misight_off` → `hyper_telemetry_blocker`；作者 Ma6302。
- 状态目录随之改名 `/data/adb/hyper_telemetry_blocker_state/`，`lib.sh` 发现旧目录时自动迁移（否则会误判"从未备份"，把已生效的值当原值存下）。
- 加固 `watchdog_alive()`：除 pid + boot_id 外，还核对 `/proc/<pid>/cmdline` 必须指向**本模块目录**的 `watchdog.sh`。
- 旧 id 的模块目录需手动删除，否则开机会被重复执行；新版本会检测到并写 `WARN` 日志。

## v1.1.0（2026-11-11）

- 新增可选日志压制 `log_suppress=off|safe|full`（默认 safe，10 条属性；full 另加 5 条，含 30 余条在本机无读取点的占位项）。
- 属性清单泛化为「可挂多份」，备份 / 还原 / 看门狗全部覆盖新清单。
- 备份改为「已有备份只补录新条目、绝不覆盖」，避免事后改档位导致新条目漏备份、卸载时留在系统里。
- `ro.*` 属性不再写入持久区。

## v1.0.3（2026-11-11）

- 修掉「重启后看门狗没起来」的致命路径：旧逻辑只用 `kill -0` 判断存活，重启后 pidfile 里的 pid 号可能被别的进程复用 → 误判已在运行而跳过启动（实机踩中：重启后属性被系统改回原值且无人纠正）。
- 改为三重校验（pid 存活 + cmdline 是 `watchdog.sh` + boot_id 与本次开机一致）；`uninstall.sh` 不再盲杀 pid。

## v1.0.2（2026-11-11）

- 备份与日志从模块目录搬到 `/data/adb/hyper_telemetry_blocker_state/`——重刷模块时模块目录会被整目录替换，旧备份随之丢失，导致卸载时把「已生效的关闭状态」当成原值还原。
- 新增 `backup_suspect` 自检与内置原值兜底。

## v1.0.1（2026-11-11）

- 看门狗改两段式：开机后按 `settle_schedule` 密集复查（实测系统会在开机后 1–5 分钟把开关写回原值），之后进入 30 分钟周期。

## v1.0.0

- 首版：停止 MiSight 采集进程、关闭 `persist.sys.misight.detectswtich` / `persist.sys.perfdebug.monitor.enable`、把 `persist.log.tag.misight` 降级到 WARN、冻结 `event.db` 增长；带状态备份、看门狗与卸载还原。
