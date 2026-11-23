# wechatauto-replica 安装与实测报告

- 测试日期：2026-11-23
- 测试对象：PyPI 成品 `wechatauto-replica==1.3.0.1`
- 测试环境：Windows / Python 3.13.14 (64bit) / 微信 **4.2.0.52**（已登录，5 个 Weixin.exe 进程）
- 说明：本报告只记录运行状态与元信息，聊天内容、账号 ID、群名均已脱敏；测试过程中未执行任何真实发消息操作。

---

## 一、安装结果：成功

| 项目 | 结果 |
|---|---|
| 安装方式 | `pip install wechatauto-replica==1.3.0.1`（隔离 venv，未污染全局环境） |
| 安装版本 | 1.3.0.1 |
| 依赖完整性 | `pip check` → No broken requirements found |
| CLI 入口 | `wechatauto.exe` 可用 |
| 与 GitHub 源码一致性 | 前面已核实：82 文件逐字节一致（仅换行符差异） |

## 二、体检（doctor）：全部通过

```
wechatauto 1.2.10 | Python 3.13.14 (64 位) | win32
账号：wxid_***（已脱敏）
数据库密钥：20/20 个库可解密
图片密钥：已拿到（已缓存，之后离线可用）
控件树（发消息/下载原图要用）：已物化
```

结论：**读取链路在微信 4.2.0.52 上真实可用**。密钥提取、SQLCipher 解密、图片密钥、UIA 热激活全部正常。

注意首行版本号显示 `1.2.10`，而 pip 安装的是 `1.3.0.1` —— 版本号不一致 bug 在成品中复现。

## 三、功能实测

| 功能 | 命令 | 结果 |
|---|---|---|
| 列出会话 | `wechatauto sessions` | ✅ 成功列出 20 个会话（群+联系人） |
| 读取群消息 | `wechatauto messages <群名> --limit 10 --json` | ✅ 返回 10 条，字段 `local_id/type/sender_id/sender_username/create_time/content/sort_seq/shard` 完整 |
| 监听单群 | `wechatauto listen <群名>` 15 秒 | ✅ 进程稳定运行，无 Traceback、无错误；15 秒内无新消息，回调投递未触发 |
| 监听全部 | `wechatauto listen --all` 20 秒 | ✅ 进程稳定运行，无报错；同样无新消息到达 |
| 导出消息 | `wechatauto export <群名> --limit 5` | ✅ 生成 5 行 UTF-8 文本文件 |
| 转发（核心需求） | `wechatauto forward ...` | ❌ **ImportError 崩溃，功能完全不可用** |
| 版本查询 | `wechatauto --version` | ❌ 报 1.2.10（应为 1.3.0.1） |

### 转发崩溃的完整调用链

```
wechatauto/wx.py:1048        from wechatauto.forward import ForwardOps
wechatauto/forward.py:39     from wechatauto.labels import (CHECKBOX_INSET_PX, ...)
ImportError: cannot import name 'CHECKBOX_INSET_PX' from 'wechatauto.labels'
```

根因：v1.3.0 重构 `labels.py` 时移除了 `CHECKBOX_INSET_PX`，但 `forward.py` 仍在导入。v1.2.10 正常，v1.3.0 / v1.3.0.1 均受影响。受影响的入口：`ForwardMessage` / `ForwardFavorite` / `SendToLabel` / `ForwardToLabel` 及 CLI `forward`。

测试未能拦截的原因：118 个单测不导入 forward 模块；专门的 `tools/test_forward.py`（59 项）发布前未运行——跑一次必然失败。

## 四、修复与验证（在 fork 上）

在你的 fork 创建了分支 `fix/forward-import-regression`，做最小兼容修复：

- 文件：`wechatauto/labels.py`
- 改动：恢复 `CHECKBOX_INSET_PX = 70`（v1.2.10 实测值），附回归原因注释
- 未改动 `main` 分支

三层验证：

| 验证项 | 结果 |
|---|---|
| `from wechatauto.forward import ForwardOps` | ✅ 成功，`CHECKBOX_INSET_PX = 70` |
| `tools/test_forward.py` | ✅ **通过 59，失败 0**（修复前必然 ImportError） |
| CLI `wechatauto forward --chat <不存在会话>` | ✅ 不再崩溃，正常走到业务逻辑并报「打不开会话」（预期行为，未驱动真实发送） |

分支已推送：`https://github.com/chen113323/wechatauto-replica/tree/fix/forward-import-regression`

## 五、重要观察与限制

1. **微信窗口当前在屏幕外**：CLI 日志显示 `WeChatGUI 初始化成功：hwnd=..., rect=(-31992, -32000, -30922, -31243)`，这是窗口最小化/隐藏时的坐标。项目依赖窗口可见做 UIA/OCR 定位，**真实转发前需把微信窗口恢复到正常可见状态**。
2. **未执行真实发送**：转发会产生对外可见动作，本次未指定测试群、未获发送确认，因此只做到 dry_run 级别的逻辑验证。想要真机验证，请用小号 + 测试群。
3. **监听回调未被触发**：两次监听期间没有新消息到达，因此「收到消息 → 回调投递」这一段只验证了线程能稳定启动、无异常，未验证实际投递。
4. **临时测试数据已清理**：`_test_msgs.json` / `_test_msgs.err` / `_test_listen.log` / `_test_listen_all.log` / `_test_export.txt` 已删除，不保留聊天内容。

## 六、结论

- **只读场景（查看群内容、监听、导出、体检）在 4.2.0.52 上实测可用，稳定可靠。**
- **转发功能在 PyPI 成品 1.3.0.1 上完全不可用**，一行导入错误导致整个模块崩溃；fork 分支已修复并通过 59 项测试，但真机发送仍需窗口可见 + 小号测试群验证。
- 建议：在作者修复上游前，需要转发就用 `fix/forward-import-regression` 分支的源码安装，或 `pip install wechatauto-replica==1.2.10`（旧版转发正常，但可能不支持 4.2 最新 UI）。

## 七、后续可选项

- 把修复分支合并进 fork 的 main，并向原作者提 issue/PR
- 用小号 + 测试群做一次真实 dry_run（开窗勾选、核对人数、取消）验证 +70px 在新版布局是否仍点得准
- 修复其余确凿 bug（监听 stop 后不可重启、回调失败丢消息、异常类默认消息、版本号不一致等）
