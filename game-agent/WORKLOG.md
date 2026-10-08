# Game-Agent 工作日志

> 生效日期: 2026-09-29
> 仓库: Game-Agent
> 当前目标: 将金铲铲自动对局能力与自动化采集项目对接

## 维护规则

1. 从 2026-09-29 起, 所有代码、配置和文档修改都在本文件登记.
2. 每次修改必须记录日期、改动文件、修改内容、验证结果和剩余问题.
3. 固定区分三类内容:
   - 预设: 已规划但尚未实现.
   - 已实现: 已进入本地代码并通过对应验证.
   - 当前问题: 已知缺陷、未验证风险和外部依赖.
4. 计划变化时更新"预设", 不删除历史变更记录.

## 当前基线

- 本地仓库提交: `ed34c03`
- 金铲铲运行环境: Windows + LDPlayer 14
- 模拟器路径: `D:\leidian\LDPlayer14`
- ADB: `D:\leidian\LDPlayer14\adb.exe`, `localhost:5555`
- 运行分辨率: `1920x1080`, DPI `280`, 横屏
- 游戏包名: `com.tencent.jkchess`
- 前台 Activity: `com.tencent.jkchess.ApolloZGame`
- 当前主入口: `jinchanchan/auto_loop.py`
- 当前主控制通道: ADB 截图和 ADB 输入
- Python 实机验证版本: `3.12`
- 当前本机虚拟环境版本: `3.14`

## 预设

- [x] 增加 `standalone` 和 `external` 两种运行模式.
- [x] 增加 `--event-file`, `--stop-file`, `--timeout`, `--device-id` 等生命周期参数.
- [x] 输出带 `schema_version`, `run_id`, `seq`, `ts` 的 JSONL 事件.
- [x] 实现严格的 `MATCH_READY`, 连续两次稳定识别后才放行业务计时.
- [x] 实现自然 `MATCH_FINISHED`, 且在结束事件后停止游戏内输入.
- 区分自然结算和 `ABORT_SURRENDER`.
- [x] 增加外部停止文件、全局超时和明确退出码.
- 修复主菜单与匹配房间的阶段误判.
- 将主循环异常日志迁移到统一日志器, 避免异常后无限继续.

## 已实现

### 2026-09-29 LDPlayer 14 环境二次确认

确认命令:

```powershell
adb devices -l
adb -s emulator-5554 shell wm size
adb -s emulator-5554 shell wm density
adb -s emulator-5554 shell getprop ro.product.cpu.abi
adb -s emulator-5554 shell getprop ro.build.version.release
adb -s emulator-5554 shell cmd package resolve-activity --brief com.tencent.jkchess
```

实测结果:

- ADB 设备序列号: `emulator-5554`
- 设备型号: `ASUS_AI2501_A`
- 画面分辨率: `1920x1080`
- DPI: `280`
- CPU 架构: `x86_64`
- Android 版本: `14`
- 游戏包名: `com.tencent.jkchess`
- 包内启动入口 Activity: `com.tencent.gcloud.msdk.core.policy.ZGamePolicyActivity`
- 游戏运行后前台 Activity: `com.tencent.jkchess.ApolloZGame`

说明:

- `resolve-activity` 返回的是 Android 包级启动入口.
- 游戏启动完成后, 实际前台窗口会切换到 `ApolloZGame`, 采集和前台判断应以后者为准.
- 本次确认时前台为 `com.android.launcher3/.Launcher`, 表示游戏本体当时未在前台运行.

### 2026-09-29 抓包能力确认

LDPlayer 版本:

- 本机没有安装 LDPlayer 9.
- 已安装版本为 LDPlayer 14, 版本 `14.0.28.0`.
- 安装目录: `D:\leidian\LDPlayer14`.
- 自带 ADB: `D:\leidian\LDPlayer14\adb.exe`.
- `C:\Program Files\ldplayer9box` 仅包含 LDPlayer 9 的虚拟化支持组件, 不是完整模拟器安装目录,
  其中没有 `adb.exe`, 不能作为需要确认的 LDPlayer 9 ADB 路径.
- 如果需要 LDPlayer 9, 需单独安装; 其 ADB 路径应取实际安装目录下的 `adb.exe`.

PCAPdroid VPN 测试:

- 测试版本: `PCAPdroid v2.0.2`.
- APK: `D:\apk\PCAPdroid_v2.0.2.apk`.
- Android 包名: `com.emanuelef.remote_capture`.
- 已在 LDPlayer 14 中安装并完成首次启动.
- 系统成功弹出 VPN 授权, 接受后创建 `tun0`.
- VPN 会话标识: `PCAPdroid VPN`.
- PCAPdroid 状态页显示已接收流量, 首次检查时约 `3.6 KB`.
- 测试结束后已停止 VPN; `tun0` 已消失.
- 结论: LDPlayer 14 内可通过 PCAPdroid 建立 VPN 抓包.

root + tcpdump 测试:

- root 检查: `su -c id` 返回 `uid=0(root)`.
- 系统自带工具: `/system/bin/tcpdump`.
- 版本: `tcpdump 4.99.3`, `libpcap 1.10.3`.
- 测试接口: `wlan0`.
- 测试命令成功捕获 `10 packets`, 内核丢包 `0`.
- 测试文件: `/data/local/tmp/jcc_tcpdump_test.pcap`, 大小 `1478` 字节.
- 结论: root + tcpdump 可作为 PCAPdroid 不可用时的备选链路.

### 2026-09-29 接受对局快速点击

问题:

- 接受对局窗口存在时间较短.
- 原流程先做一次全屏 OCR 识别 `ACCEPT`, 再调用 `click_text("接受")` 做第二次全屏 OCR.
- CPU 版 EasyOCR 每次约需 5-8 秒, 两次识别叠加后点击过晚, 匹配已取消并返回房间.

修改:

- `jinchanchan/auto_loop.py`
  - `handle_accept()` 改为识别到 `ACCEPT` 后直接点击固定接受按钮.
  - 移除接受流程中的第二次全屏 OCR.
  - dry-run 模式只记录日志, 不发送点击.
- `jinchanchan/agent/sdk/regions.py`
  - 将 `SpecificButton.ACCEPT_COMBAT` 按 `1920x1080` 实测坐标重新校准.
  - 实测按钮中心: `(959,830)`.
  - 基准坐标中心: `(511.5,553.5)`, 缩放后点击中心 `(959,830)`.

验证:

- 已完成: `jinchanchan` Python 编译检查通过.
- 已完成: 坐标缩放检查通过, `1920x1080` 下点击中心为 `(959,830)`.
- 暂停: LDPlayer 实机匹配接受测试. 2026-09-29 金铲铲停服, 计划 12:00 后继续.

### 2026-09-29 M2 external 对局执行器

新增文件:

- `jinchanchan/agent/lifecycle.py`
- `jinchanchan/tests/test_lifecycle.py`

修改文件:

- `jinchanchan/auto_loop.py`
- `jinchanchan/agent/sdk/adb.py`

已实现:

- external 命令示例:

```powershell
python auto_loop.py `
  --mode external `
  --device-id 127.0.0.1:5555 `
  --event-file <run_dir>\events.jsonl `
  --stop-file <run_dir>\stop.game `
  --timeout 900 `
  --run-id <run_id>
```

- `standalone` / `external` 模式拆分, standalone 保留原全流程行为.
- external 模式只等待对局开始并执行对局内阶段, 不点击匹配、接受对局或下一局.
- `--device-id <host>:<port>` 映射为 ADB TCP 端点.
- `--device-id` 与 `--host/--port` 冲突时退出码为 `2`.
- `--event-file` JSONL 事件, 采用临时文件加 `os.replace` 原子替换.
- 事件包含 `schema_version`, `run_id`, `seq`, `ts`, 且支持一次性事件幂等.
- `--stop-file` 每 `500ms` 检查一次, 触发后退出码为 `3`.
- `--timeout` 全局超时, 触发后退出码为 `4`.
- 正常运行退出码为 `0`, 业务失败退出码为 `1`, 参数/环境错误退出码为 `2`.
- 严格 `MATCH_READY`: 合法回合号加局内阶段, 连续两次且间隔 `1.5s`.
- external 模式检测到 `Stage.RESULT` 后输出 `MATCH_FINISHED` 并立即返回.
- 结算结果解析支持 `win` / `loss` / `unknown`, 同时输出名次.
- external 模式把全局对局操作交给 watchdog, 超时或停止文件可取消主任务.
- ADB 本机扫描同时兼容 `localhost`, `127.0.0.1`, `::1`.
- 当前事件集合: `SCRIPT_STARTED`, `DEVICE_READY`, `WAITING_MATCH_READY`,
  `MATCH_READY`, `ROUND_PHASE`, `MATCH_FINISHED`, `ERROR`, `SCRIPT_STOPPED`.

验证:

- 已完成: `compileall` 通过.
- 已完成: `unittest discover -s tests -v`, 共 7 项全部通过.
- 已完成: CLI 参数冲突、external 缺少必填参数、非法 `--device-id` 均返回退出码 `2`.
- 已完成: 无模拟器环境下返回退出码 `2`, 并生成 `SCRIPT_STARTED`,
  `ERROR(stage=environment)`, `SCRIPT_STOPPED` 三条合法 JSONL 事件.
- 已完成: 模拟器启动后, stop-file 进程级测试返回退出码 `3`; 事件顺序为
  `SCRIPT_STARTED`, `DEVICE_READY`, `WAITING_MATCH_READY`, `SCRIPT_STOPPED(stop_file)`.
- 已完成: 模拟器启动后, timeout 进程级测试返回退出码 `4`; 事件顺序为
  `SCRIPT_STARTED`, `DEVICE_READY`, `WAITING_MATCH_READY`, `ERROR(stage=timeout)`,
  `SCRIPT_STOPPED(timeout)`.
- 已完成: 对局中的严格 `MATCH_READY` 实机验证; 首次运行从 `1-2` 识别成功.
- 已完成: 自然 `MATCH_FINISHED` 实机验证; 续跑从 `4-2` 接续到 `6-1` 后识别结算.
- 已完成: 结算后进程立即以退出码 `0` 结束, 未点击“下一局”或结算按钮.
- 已完成: JSONL 契约程序化校验, 共 `20` 条事件, `seq` 连续,
  `MATCH_READY` 和 `MATCH_FINISHED` 各出现一次, 最后一条为 `SCRIPT_STOPPED`.
- 实机事件文件: `jinchanchan/logs/m2-live-resume-20260929-210909/events.jsonl`.
- 已观察: 从 `4-2` 到 `6-1`, ROUND_PHASE 覆盖 battle/prepare/carousel,
  对局内购买、经验、上阵和选秀动作持续执行.
- 已观察: 第一次完整运行在 1200 秒到达全局 timeout, 输出 `ERROR(stage=timeout)`
  和退出码 `4`; 第二次续跑在自然结算时正常结束.

## 当前问题

- `Ctrl+C` 已增加顶层处理和退出码 `3`, 但尚未在长时间 ADB/OCR 任务中实机复验.
- `StageDetector` 会把主菜单中的"魔典任务"误判为 `queue`.
- `1600x910 / DPI 240` 下金币和等级 OCR 区域失配, 当前不支持该分辨率.
- external 的 `MATCH_FINISHED` 当前依赖全屏 OCR, 尚未验证是否满足结算出现后 5 秒内输出.
- 本次自然结算事件输出为 `result=unknown`, 结算页名次解析仍需补充稳定的识别特征.
- 金币 OCR 存在阶段性读取为 `None`, 但回合号可用时对局动作仍能继续.
- stdout 尚未做到纯 JSONL, OCR 和普通日志仍会混入 stdout; 当前外部总控应只读取事件文件.
- `ABORT_SURRENDER` 和主动异常清理尚未实现.
- 问号模板在部分画面存在疑似假阳性点击.
- 本机使用 Python 3.14, 与作者实机验证的 Python 3.12 不一致.
