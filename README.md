# 麦麦转发（MaiForward）

- 插件 ID：`github.cateye.mai-forward`
- 作者：cateye
- 类型：tool（LLM 工具插件）

把指定的聊天消息**按原样**打包，进行**合并转发**或**逐条转发**，并支持**复读**单条消息。消息内容一律从 NapCat / MaiBot 宿主读取，不由 LLM 转述构造，避免转发内容失真；发送成功后向目标聊天注入一条带来源标注的聊天记录（`[转发工具 源→目标]` / `[复读工具]`）。

## 注册的 LLM 工具

### `forward_messages` — 打包转发

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `message_ids` | string[] | ✔ | 要打包的消息 ID 列表（可多个） |
| `target_stream_id` | string | ✔ | 目标聊天流 ID（也接受目标群号 / 对方 QQ 号） |
| `merge` | bool | ✔ | 是否合并为一条合并转发消息 |

**缺少任意一个参数都会直接报错**：打印 debug 日志，并把缺少的参数、收到的原始参数、正确用法一并返回给 LLM。任一消息 ID 读不到时同样中止并逐条列出（不部分发送）。

### `repeat_message` — 复读

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `message_id` | string | ✔ | 要复读的消息 ID |

复读在当前聊天进行；若被复读的消息来自其他聊天，视为跨聊天转发，同样受转发名单限制。发送成功后构造一条相同的信息入库，开头显示 `[复读工具]` 字样。

## 工作原理

1. **取原内容（防失真）**：优先通过 NapCat 适配器 `get_msg` 拉取原始消息段（内容、发送者、时间均来自协议端）；适配器不可用时回退 MaiBot 宿主消息记录（`message.get_by_id`，含二进制数据）。
2. **发送**：
   - **优先 NapCat 直连**（`prefer_napcat_direct = true`，默认开）：合并转发用 `send_group_forward_msg` / `send_private_forward_msg` 的**引用节点**（`{"type":"node","data":{"id":消息ID}}`），逐条转发用 `forward_group_single_msg` / `forward_friend_single_msg`——原始消息由 QQ 服务器直接取用，内容零改动；
   - **回退宿主路径**：把读取到的原始内容段构造为转发节点，经 `ctx.send.forward` 发送（图片用 base64 或 URL 引用，语音/视频/文件等用占位文本摘要）。
3. **注入聊天记录**：发送成功后，通过插件自带的 receive 网关以 `is_notify=True` 合成通知消息走完整入站链写入数据库（官方"只入库不真发"通道，WebUI 聊天记录可见，不触发回复循环）：
   - 转发：开头标注 `[转发工具 源群号→目标群号]`（私聊为对应 QQ 号），逐条列出每条消息的时间、发送者与内容摘要；
   - 复读：`[复读工具] 原消息内容`。
4. 发送时统一关闭宿主自动入库（`storage_message=False`），保证每个动作在库中只有一条带标注的记录。

## 配置项

| 配置节 | 字段 | 默认 | 说明 |
|---|---|---|---|
| `[plugin]` | `enabled` | `true` | 插件总开关 |
| `[group_permission]` | `list_type` | `"blacklist"` | 群聊名单类型：`blacklist`（黑名单）/ `whitelist`（白名单） |
| | `id_list` | `[]` | QQ **群号**列表 |
| `[private_permission]` | `list_type` | `"blacklist"` | 私聊名单类型 |
| | `id_list` | `[]` | 对方 **QQ 号**列表 |
| `[forward]` | `force_merge` | `true` | 强制合并（见下） |
| | `force_merge_age_days` | `2.0` | 触发强制合并的消息年龄阈值（天） |
| | `prefer_napcat_direct` | `true` | 优先 NapCat 原样直连转发 |

### 名单判定规则（群聊/私聊分开配置、双向检查）

- **黑名单模式**：留空 = 全部允许；填入的群号/QQ号**不允许**参与转发。
- **白名单模式**：留空 = 全部不启用（谁都不能转发）；只有填入的群号/QQ号**允许**参与转发。
- 转发时**源聊天与目标聊天都要通过检查**（各自按群聊/私聊名单判定）：
  - 源未通过 → 告知 LLM「当前群/私聊不允许转发聊天记录」；
  - 目标未通过 → 告知 LLM「不允许将信息转发到目标群/私聊」；
  - 两者都未通过 → **优先显示当前聊天不允许转发的提示**（并附注目标同样未通过）。
- 无法识别的聊天来源（如本地控制台触发）不拦截。

### 强制合并

`force_merge = true`（默认）时，若要转发的消息中存在早于 `force_merge_age_days`（默认 2 天）的旧消息，将**忽略 LLM 传入的 `merge` 参数**，强制合并为一条合并转发消息，并在工具返回中告知 LLM 已强制合并。

## 安装

1. 将本目录放入 MaiBot 安装目录的 `plugins/` 下；
2. 重启 MaiBot，确认日志出现 `[麦麦转发]` 加载输出（本插件带 manifest 能力声明，**必须完整重启**才生效）；
3. 运行时 `config.toml` 由 Runner 自动生成，可在 WebUI 插件管理中修改并热重载。

NapCat 适配器（`maibot-team.napcat-adapter`）为**可选依赖**：未安装时本插件自动走宿主回退路径（仅影响转发保真度，不影响功能可用性）。

## 注意事项

- 工具默认进入 deferred 池，由 Planner 通过 `tool_search` 发现；如需常驻可在工具元数据上加 `core_tool`。
- 私聊目标的注入记录：私聊会话按对方 QQ 号定位，合成记录的 user_info 记为对方（机器人账号写在 `additional_config.self_id`），来源说明已包含在记录正文前缀中。
- 多账号部署时 NapCat 直连路径走适配器自身的连接；如需严格按会话账号路由，请设置 `prefer_napcat_direct = false`。
- 若发现目标聊天中出现"无标注 + 有标注"两条记录，说明宿主未 honored `storage_message=False`，请反馈并检查主进程日志。
