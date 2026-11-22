---
id: "b48dc14c-afaf-40fd-9fe7-b692cd8cb9fc"
title: "重生2我独自逆袭 启动闪退排查：exit code 0 与 SteamSdk 初始化失败（BuildID 26832099，2026-11-22）"
description: "Steam 正版游戏启动 4–7 秒即退出的只读排查记录：日志/事件/Steam 侧证据、补丁 vs 驱动的时间线、回退驱动 A/B 验证步骤与开发商反馈模板。"
status: "active"
created_at: "2026-11-22T09:25:08.569Z"
updated_at: "2026-11-22T09:25:08.569Z"
content_hash: "e1672caff08d4041a10b8ab896fcb3d21373df2816ef3a487754b736b9c25559"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-66fcbc74-880b-48d7-ae11-5abd1d6369fd"
memory_body_ids:
  []
---

# 重生2我独自逆袭：启动后 4–7 秒退出（exit code 0）排查记录

## 背景

- 2026-11-22，本机（RTX 5080，VRAM 15977 MB，NVIDIA 驱动 32.0.16.2124 即 621.24），Steam 正版游戏，AppID **3816120**，Unity 2021.3.45f2c1，IL2CPP。
- 现象：点启动后加载页出现，约 4–7 秒窗口消失。当天 16:21、16:49、16:52、16:59、17:03、17:08 共 6 次启动表现一致。
- 全程只读排查：未运行游戏、未修改任何文件（会话为只读模式）。

## 结论

不是显卡驱动崩溃，也不是程序崩溃，而是**游戏主动正常退出**：Steam SDK 初始化失败 → 连续保存存档 → 正常关闭。最可疑变量是 2026-11-22 16:04 Steam 推送的 **BuildID 26832099** 补丁；驱动为次级候选，需回退驱动做 A/B 对照才能最终定案。

## 关键证据

| 判据 | 实际情况 | 含义 |
|---|---|---|
| 进程退出码 | `gameprocess_log.txt` 当天 6 次启动全部为 `no longer tracking PID …, exit code 0` | 程序自己退出，不是被系统/驱动强制结束 |
| 日志结尾 | `Player.log` 末尾打印完整 `Memory Statistics`（`ALLOC_TEMP_TLS` 等分配器统计） | 只有 Unity 正常关闭流程才会打印，崩溃/TDR 时不会出现 |
| 系统事件 | 无 Application Error 1000、无 WER 报告、无 BugCheck、无 nvlddmkm 4101 (TDR) | 显卡驱动未崩溃 |
| 图形初始化 | `Direct3D 11.0 [level 11.1]`、`Renderer: NVIDIA GeForce RTX 5080`、`VRAM: 15977 MB`、`Driver: 32.0.16.2124` | D3D11 设备创建成功，排除「驱动建不出设备」路径 |

`Player.log` 第 27–44 行的顺序：

```
27 开始加载load界面:17:08:26
28 开始初始化存档:17:08:27
...（InfoTipsMrg / EventContentMrg / GameDataMrg / MouseAutoHideMrg 初始化正常）
35/36 全屏设置为: 2560x1440
37 SteamSdk初始化:17:08:27
38 SteamSdk初始化失败:17:08:27     ← 唯一报错
39-42 保存存档 ×4
43 Curl error 42: Callback aborted  ← 退出时中断网络请求，是结果不是原因
44 Memory Statistics:              ← 正常关闭
```

从 `global-metadata.dat` 反查到 `SteamSdk初始化:` / `SteamSdk初始化成功:` / `SteamSdk初始化失败:` 三种字符串都在程序里，说明开发者把「Steam 初始化失败」当作显式分支处理，而这条分支的结局就是保存并退出。

## 时间线

- **11-21 23:52 启动 → 11-22 00:32 结束**：补丁前最后一次正常运行（Steam 记录 00:32:08 Fully Installed）。
- **16:03:57**：Kernel-Power 41（上一次非正常关机），16:53 手动重启。
- **16:04:40–16:04:48**：Steam 自动更新 AppID 3816120，BuildID **26832099**，`37 updated, 0 moved, 13 deleted`，下载 143,501,552 B、delta 65,726,246 B；`GameAssembly.dll` 时间戳变为 16:04（IL2CPP 全部逻辑，Steam 初始化代码就在其中）。
- **16:11**：新 NVIDIA 驱动进入 DriverStore（`nv_dispi.inf_amd64_81cce6c273d405a6`）；旧驱动 `…cee48bd6d33f95ab`（11-13 17:33）仍保留，可直接回退。
- **16:21**：首次闪退，此后每次同样 6 秒退出。

两个变化都在首次闪退之前，单靠时间排序不能定案；但日志内容明确排除了驱动崩溃路径（若是驱动问题，应出现 TDR、黑屏、`0xC0000005` 或 device removed）。

## Steam 侧排除项（均正常）

- `connection_log.txt` 16:55:18 登录 `'OK'`；客户端非 beta 通道；客户端本体当天未更新（5 月版本）。
- `console_log.txt`：覆盖层正常拉起，启动流程无异常。
- 17:00 文件完整性校验，17:03 完成且无错误。
- `cloud_log`：17:08:30 成功上传 `SaveData.dat`（154,960 B）→ 读档/存档链路通，排除「存档坏死」。
- `steam_api64.dll` 为正版 Valve 10.24.16.75。

## 与本次无关但值得处理的系统问题

1. 16:11:35 `C:\DeepCool\resources\service\x64\deep_service.exe` 在新驱动 `nvml.dll`（8.17.16.2114）内崩溃（c0000005，偏移 0xfd6e0）。散热软件与新驱动 NVML 的兼容问题，不影响游戏；回退驱动后应一并消失。
2. 16:03:57 的 Kernel-Power 41 说明这次驱动安装是在异常关机之后完成的，安装过程不算干净。

## 建议处理顺序

0. **先备份存档**（每次失败启动都会写 4 次存档）：`C:\Users\zhouy\AppData\LocalLow\ProudmateEntertainment\重生2我独自逆袭\SaveData.dat`。
1. **排除 Steam 本地状态**：彻底结束 `steam.exe`、`steamwebhelper.exe` → 以管理员身份运行 Steam → 确认已登录（非离线模式）→ 启动游戏。
2. **A/B 定案（关键一步）**：设备管理器 → 显示适配器 → RTX 5080 → 属性 → 驱动程序 → 回退驱动程序（旧驱动在 DriverStore，回退很快；或 DDU 后装 11-13 版本）。
   - 仍闪退 ⇒ 与驱动无关，确认是补丁问题。
   - 能进 ⇒ 驱动确为元凶，同时 DeepCool 崩溃一起消失。
3. **若确认是补丁问题**：本地基本无解，等开发商热修并反馈；反馈时附上下节要点。低成本可试：Steam 启动选项 `-screen-fullscreen 0`；退出 Steam 后用离线模式启动一次（各约 1 分钟，用于排除全屏切换与联网校验两个变量）。

## 给开发商的反馈要点（可直接复制）

- AppID 3816120，出问题 BuildID **26832099**（2026-11-22 16:04 推送）；此前版本正常。
- 现象：加载页出现后 4–7 秒窗口消失，Steam 记录 **exit code 0**，非崩溃。
- `Player.log` 关键行：`SteamSdk初始化:…` → `SteamSdk初始化失败:…` → `保存存档`×4 → 正常关闭。
- 同一台机器补丁前（11-21 23:52）可正常进入；已做文件完整性校验（通过）、重装 Steam 并重登（无效）。
- 附件：`C:\Users\zhouy\AppData\LocalLow\ProudmateEntertainment\重生2我独自逆袭\Player.log`。

## 证据路径

- 游戏日志：`C:\Users\zhouy\AppData\LocalLow\ProudmateEntertainment\重生2我独自逆袭\Player.log`、`Player-prev.log`
- Steam 日志：`C:\Program Files (x86)\Steam\logs\gameprocess_log.txt`、`content_log.txt`、`console_log.txt`、`connection_log.txt`、`cloud_log.txt`
- 安装目录：`C:\Program Files (x86)\Steam\steamapps\common\重生2我独自逆袭\`
- 应用清单：`C:\Program Files (x86)\Steam\steamapps\appmanifest_3816120.acf`

## 未完成 / 待验证

- 驱动回退 A/B 实验尚未执行（需用户侧操作），因此「补丁 vs 驱动」尚未最终定案；当前排序依据是日志排除法，而非对照实验。
- 若回退驱动后仍闪退，即可把 BuildID 26832099 作为确定结论对外反馈。
