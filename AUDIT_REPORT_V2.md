# wechatauto-replica v1.3.0.1 审计报告 V2（全盘复核版）

- 复核日期：2026-11-23
- 对象：github.com/fanyuantaier/wechatauto-replica @ 3fff263
- 方法：三路独立复核，**每条必须追调用链或实测**，只看函数定义的一律判"证据不足"
- 与 V1 的关系：V1 有 20+ 条误报或过度定级，本版以本版为准，V1 存档仅作对照

## 零、V1 错在哪：根本方法论问题

V1 把"看起来可疑的代码形式"当成了 bug，没有追调用链和作者的意图注释。三类典型错误：

1. **忽略语义上下文**：`aligned_aes_block_size` 的 `+16` 在 PKCS7 里是正确填充，被误判为 bug。
2. **只看一半代码**：`_current_chat` 跳过 open_chat 的条件还有第二项 `_chat_is_open(who)` 实时校验，V1 只读了第一项。
3. **忽略兜底/降级路径**：RVA 版本表只有 4 项，但主路径是字节扫描+物化校验，版本表只是最后兜底；SEARCH_BOX_RATIO 有 UIA 精确矩形优先。

## 一、V1 结论推翻清单（这些不是 bug，不要改）

| V1 编号 | 条目 | 复核判定 | 决定性证据 |
|---|---|---|---|
| P0-1 | 图片解密块对齐 bug | **误报** | PKCS7 明文16倍数补整块；实测15/16/20/32B 全部 MATCH |
| P1-4 | _current_chat 不同步发错页面 | **误报** | `guia.py:2193` 跳过条件是 `who==_current_chat AND _chat_is_open(who)`，有实时二次校验；切朋友圈还有 `back_to_chat_tab()` 补救（2158） |
| P1-5/6 | 假成功门禁 | **设计权衡** | `_paste_landed` 读不到值放行是注释明示的意图（避免控件改版锁死发送）；verify=True 有 `_wait_landed` 兜底 |
| P1-12 | ForwardToLabel dry_run 默认值 | **设计权衡** | `wx.py:1201` docstring 明写"默认只预演"，标签批量默认安全 |
| P2-1 | 布局缓存只校验宽度 | **误报** | 缓存的是比例；`guia.py:723` 运行时 `当前render_h × 比例` 重算，高度变化不产生陈旧 y |
| P2-3 | SEARCH_BOX_RATIO 太窄 | **误报** | `guia.py:1756` UIA 精确矩形优先，比例仅 UIA 不可用时兜底 |
| P2-4 | 名字列 0.55 硬编码 | **设计权衡** | 是相对 sidebar_right 的比例，随侧栏缩放；有实测注释 |
| P2-6 | RVA 表与字节码扫描失效 | **误报** | `accessibility.py:454` 候选顺序=已验证缓存>特征扫描>版本兜底表；每个候选有 `_mmui_present` 物化校验 |
| P2-11 | 时钟回拨清空 stamps | **设计权衡** | `rhythm.py:258` 注释明示"时钟回拨数据一律丢弃" |
| P2-12 | 跨进程最后写入者赢 | **设计权衡** | `rhythm.py:25` 作者明示"故意没做互斥锁" |
| P2-16 | DPAPI 失败降级明文 | **设计权衡** | `db.py:194` 注释明示兼容旧版；且密文解不开返回空而非误当明文 |
| P2-22 | 轮询同步串行查库 | **设计权衡** | 单轮询线程设计；作者在 `db.py:3045`、`recall.py:351` 两处知情记录 |
| P3-1 | 输入框探针绝对像素 | **设计权衡** | 7 个偏移兜底 + UIA 优先 + 失败自动重校准 |
| P3-11 | --dry-run 是死标志 | **误报** | `args.dry_run` 在 `cli.py:469,534` 确实被读取（V1 看错了子命令） |
| — | Qt 类名写死 | **误报** | `guia.py:580` 评分制定位，类名只是软条件；注释明说 Qt 升级会改名 |
| — | silkdec 环境变量无校验 | **设计权衡** | 环境变量是信任输入，subprocess 列表形式无 shell 注入 |
| — | 硬编码窗口标题 | **非 bug** | 微信中文客户端真实标题，且有类名双锚点 |

## 二、需要降级/修正描述的（V1 定级过重）

| V1 编号 | 条目 | 修正 |
|---|---|---|
| P1-3 | 多显示器 SendInput 坐标 | 条件性，且**只影响右键引用**；主发送路径用 SetCursorPos 物理坐标，不受影响 |
| P1-10 | 图片密钥地标硬编码 | 条件性，删掉"导致坏图"断言；下游 `media.py:159 _validate_key` 用真实密文兜底 |
| P1-11 | 转发回读错库 | 条件性，**单账号无影响**（你的场景） |
| P2-5 | 类名硬编码 | Qt 部分误报，仅 mmui 类名降级为条件性 |
| P2-7 | WS_EX_TRANSPARENT 并发 | 删"永久损坏"结论——finally 收敛，最终样式正确；仅瞬时穿透风险 |
| P2-9 | SPI 前台锁不还原 | 条件性，ctypes 调用很难抛异常，触发苛刻 |
| P2-10 | rhythm burst 失效 | 条件性，仅自定义 `cooloff=(0,0)` 触发；三个预设档均 >0 |
| P2-13 | WAL applied=-1 | 确凿但危害需说明：缺消息状态被 stamp 固化（`db.py:1535` build 判定不看 applied） |
| P2-14 | 内存扫描越界 | 条件性，实测可抛 struct.error，但需尾部恰好=13（约 2^-32） |
| P2-17 | 连接"泄漏" | 条件性，GC 可回收；`_open` 少抛 DatabaseError，路径窄 |
| P2-19 | 16TB 稀疏文件 | **实测不成立**——Windows `seek(0xFFFFFFFF*4096)` 抛 OSError，未建文件；降为条件性 |
| P2-23 | WeChat(account=) 被吞 | kwargs 吞参数属实，但删"README 宣称支持 WeChat(account=)"（README 实指 WeChatDB） |
| P2-24 | 转发弱校验 | 条件性，删"重试必然重复"（代码无自动重试）；准确说法是"缺幂等，调用方重试会重复" |
| P2-26 | selftest 562/0 | 数字是运行时统计非硬编码；"混入真机组+50%软断言"保留 |
| P2-28 | dry_run 多块 | "不重开窗"属实；删"块数虚报"（chunks 是数学公式，数值正确） |
| P3-4 | 路径穿越 | 修正归因：username 前缀有保护；真正薄弱点是合并转发 title（`media.py:2358`） |

## 三、确凿 bug 清单（复核后仍成立）

### 高优先级 —— 消息静默丢失（三条同源）

**B-1 防撤回镜像主键缺 shard**（V1 P1-7）
- `recall.py:103` `PRIMARY KEY (chat, local_id)`；`recall.py:120` 写入取原始 `local_id`，忽略随消息下发的 shard
- `db.py:1874` 作者自述实测：单账号 20008 个 `(会话, local_id)` 跨分片重复
- 后果：镜像互相覆盖，撤回时取错原文或原文已丢

**B-2 Listener stop() 后不可重启 + 消息永久丢失**（V1 P1-8）
- `db.py:3193 stop()` 未清空 `_worker_queues`/`_worker_threads`；`db.py:3240 _dispatch` 复用旧 queue
- 更严重：`_poll_once` dispatch 前已把 `_inflight[user]` 推到本批 last_seq；这批消息重启后 `since <= inflight` 不再拉取 → **永久丢失**
- 触发：进程内 stop→start，或热重载

**B-3 回调失败仍推进 watermark**（V1 P1-9）
- `db.py:3264-3272`：重试耗尽仅写 stderr，随后无条件 `_commit_watermark`
- 全仓无死信表/未确认记录/补偿机制；重启后该消息永久不再拉取

### 中优先级

**B-4 WAL 合并失败被 stamp 固化**（V1 P2-13）：`db.py:1535` build 判定不含 applied；`applied=-1` 的降级快照在 WAL 稳定后永久沿用。

**B-5 _decrypt_v2 长度字段无校验**（V1 P2-20，与 P0-1 无关）：`media.py:506`；实测 `xor_size > len(data)` 时切片反转产静默坏图。

**B-6 图片 AES 密钥明文非原子覆盖**（V1 P2-21）：`media.py:223-235` 无 DPAPI/无锁/无原子写，与项目内 `db.py:1362` 的成熟写法不一致。

**B-7 异常类默认消息失效**：`exceptions.py:18` 基类 dataclass 字段覆盖子类类属性；实测 `str(NetWorkError())` 返回类名。
- 精确修复：`default_message: ClassVar[str] = ""`

**B-8 版本不一致**：`__init__.py:56` = 1.2.10 vs `pyproject.toml:7` = 1.3.0.1

**B-9 leave_message 死参数**：`forward.py:276` 声明并透传，函数体从不读取；全仓仅好友申请有独立留言实现（不同功能）

**B-10 import 包破坏宿主日志/stdout**：`logger.py:56 root_logger.handlers.clear()` + `:13` 替换 sys.stdout，在 import 阶段执行

**B-11 剪贴板静默覆盖不恢复**：`guia.py:1939`、`uia_driver.py:1704`；`_clip_get` 定义了但全仓无调用；摧毁用户复制的密码/证件号

**B-12 CLI --fav 丢弃 --text**：`cli.py:364-375`

**B-13 auto_download 未知类型 KeyError**：`auto_download.py:371` 绕过构造器 ValueError

**B-14 numpy 未声明依赖**：`accessibility.py:45` 顶层 import，pyproject 无 numpy

**B-15 Chat.__str__ AttributeError**：`wx.py:356` 空 Chat 访问不存在的 nickname

### 低优先级 / 条件性

- `WeChat(account=...)` 被 kwargs 吞（多账号直觉传参无效）
- 转发"送达"判定弱（只看窗口关闭 + 任意新行），无幂等
- except-pass 170 处（多数是注释明示的降级，需逐处评估）
- dry_run 多块不重开窗（预演与真实路径不一致）
- CLI 交互输入 `-` 开头被 argparse 误解析
- 重复收件人不去重（部分被 `already` 短路缓解）
- sort_seq 为 NULL 时 TypeError（口径与其他路径不一）
- 合并转发 title 路径未净化（真正的穿越薄弱点）
- XOR 密钥末级兜底 0x88（前两级命中率高）

## 四、对初版审计的自我批评

初版报告最大的问题不是"找到了假 bug"，而是**验证方法错了**：我用 `python -c` 打印函数返回值，却拿错误的预期去对照，等于验证了自己的假设。正确做法是：

1. 先读注释和文档，理解设计意图（PKCS7、UIA 优先、评分制定位）
2. 追完整调用链，看条件的所有合取项、看兜底分支
3. 实测时用真实语义的输入（真实 PKCS7 加密，而非猜想的 size 值）
4. 对"看起来可疑"的代码，默认假设作者是对的，直到证明相反

V1 的"60+ 问题"里，复核后确凿 bug 约 15 条、条件性约 15 条、误报/设计权衡约 30 条。

## 五、你的场景（4.2.0.52 / 单账号 / 实时转发）净结论

**可以放心用的**：
- 图片解密（V1 的"图片损坏"是误报）
- 发送内容校验（有 verify 兜底）
- 布局定位、RVA 热激活、类名适配（作者做了多路径兜底）

**需要规避的**：
- 不要用 `ForwardToLabel`（默认只预演）
- 转发前备份剪贴板
- 单账号使用（你的场景已满足）
- 不要 stop/start 监听器（B-2 会永久丢消息）
- 不依赖防撤回（B-1）

**测试验证状态**：`python -m unittest discover -s tests` → 118 tests OK（复核期间重跑）

## 六、修复优先级建议

1. B-2 / B-3 / B-1 / B-4（四条消息静默丢失，危害最大）
2. B-7（异常消息，一行 ClassVar 修复）、B-8（版本号，一行修复）
3. B-10 / B-11（库副作用与用户数据）
4. B-5 / B-6 / B-9 / B-12
5. 其余条件性问题按需
