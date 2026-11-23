# 部署与使用说明（本机常驻）

部署日期：2026-11-23
微信版本：4.2.0.52
Python：3.13.14 (64-bit)

## 一、已完成的部署

| 项目 | 位置 / 状态 |
|---|---|
| 长期虚拟环境 | `C:/Users/windows/WorkBuddy/2026-11-23-07-59-21/wxauto_env/` |
| 安装方式 | 可编辑模式（`pip install -e`）装的是你的 fork `fix/forward-import-regression` 分支源码 |
| 版本 | 1.3.0.1（**含转发修复**，PyPI 原版此版本转发是坏的） |
| 依赖完整性 | `pip check` → No broken requirements found |
| 转发模块 | ✅ 可导入，`CHECKBOX_INSET_PX = 70` 修复已生效 |
| 数据库密钥 | ✅ 20/20 个库可解密，图片密钥已缓存 |

## 二、关键前提：微信窗口必须在前台可见

**这是本项目最容易踩的坑，也是唯一需要人工维持的状态。**

- 微信进程在运行 ≠ 能用。窗口最小化到托盘时，主窗口句柄为 0，脚本会报
  `未找到微信主窗口，请确认微信已登录并运行`，控件树热激活也会失败。
- 使用时必须把微信窗口**从托盘点开、保持可见**（不需要一直是最前台，但窗口不能被最小化）。
- 检查方法：
  ```bash
  cd C:/Users/windows/WorkBuddy/2026-11-23-07-59-21
  wxauto_env/Scripts/python -c "import logging; logging.disable(logging.CRITICAL); from wechatauto.guia import WeChatGUI; g=WeChatGUI(); print(g.render_rect)"
  ```
  返回四个正数（如 `(48, 0, 1118, 751)`）就是可见；报错或负值就是窗口没打开。

## 三、日常运行命令

先进入环境目录：
```bash
cd C:/Users/windows/WorkBuddy/2026-11-23-07-59-21
```

```bash
# 体检（看账号/密钥/控件树状态）
wxauto_env/Scripts/wechatauto.exe doctor

# 列出会话 / 读某群消息 / 导出
wxauto_env/Scripts/wechatauto.exe sessions
wxauto_env/Scripts/wechatauto.exe messages "群名" --limit 20 --json
wxauto_env/Scripts/wechatauto.exe export "群名" --out "导出.txt"

# 发消息 / 转发（会驱动微信窗口，务必先确认窗口可见）
wxauto_env/Scripts/wechatauto.exe send "内容" --to "文件传输助手" --verify
wxauto_env/Scripts/wechatauto.exe forward --chat "源群" --to "目标群" --dry-run   # 先预演
wxauto_env/Scripts/wechatauto.exe forward --chat "源群" --to "目标群" --go        # 真发

# 监听某群新消息（Ctrl+C 退出）
wxauto_env/Scripts/wechatauto.exe listen "群名"
```

## 四、拟人节流（防风控）

环境变量 `WECHATAUTO_RHYTHM=calm` 可切换到最保守档（间隔 6~14 秒、300 秒内最多 3 次写动作）。
连续操作时项目会**主动冷却 60~90 秒**，这是防封号的正常保护，不是卡死。

 bash 下临时启用：
```bash
WECHATAUTO_RHYTHM=calm wxauto_env/Scripts/wechatauto.exe forward ...
```

## 五、注意事项

- **保持微信登录**：退出登录后密钥失效，需重新扫码并重取密钥（`doctor` 会提示）。
- **权限一致**：Python 和微信都用普通权限，或都用管理员权限，不要混用。
- **单账号使用**：多账号同时登录时转发回读可能用错数据库（审计中的条件性 bug）。
- **封号风险**：个人号自动化非官方接口，用小号 + 低频 + calm 档；不要群发营销内容。
- **源码更新**：当前装的是 fork 的修复分支。上游有新版本时：
  ```bash
  cd wechatauto-replica
  git fetch upstream && git merge upstream/main
  # 注意：合并后要确认 CHECKBOX_INSET_PX 修复没被覆盖掉
  ```

## 六、尚未做的（按你的选择留到以后）

- 监听源群 → 自动转发到目标群的常驻脚本（需要你提供源群名和目标群名）
- 开机自启（Windows 任务计划程序 / 启动文件夹快捷方式）
