# 麦麦转发（MaiForward）

- 插件 ID：`github.cateye.mai-forward`
- 类型：`tool`（LLM 工具插件）
- 作者：cateye
- 宿主：MaiBot 1.0.0 ~ 1.4.99
- SDK：`maibot_sdk` 2.5.0 ~ 2.99.99
- 许可证：MIT

麦麦转发让机器人把聊天消息搬到另一个聊天：合并或逐条转发、复读单条消息，以及把若干消息打包收藏后再以合并转发分享。消息正文由插件从 NapCat 或宿主消息库读取，LLM 只负责挑选消息、指定目标和写概括，不能用自己转述的文本构造转发内容。

直接对机器人提出转发、复读或收藏要求即可。群聊记录补录启用时，插件会注册用于入库的接收网关 `maiforward_recorder`。

## 工具总览

| 工具 | 作用 | 是否真的发送消息 |
| --- | --- | --- |
| `forward_messages` | 把消息转发到指定聊天 | 是 |
| `repeat_message` | 把一条消息复读到当前聊天 | 是 |
| `pack_chat_messages` | 把消息打包收藏到本地 | 否 |
| `list_message_packs` | 列出本地尚未销毁的包 | 否 |
| `share_message_pack` | 把包以合并转发分享到当前聊天 | 是 |

启用「群聊记录补录」时，转发、复读或分享到**群聊**成功后会尝试补录一条以工具名标注的记录。**私聊**发送记入本地审计。

## 安装与依赖

- 将插件目录放入 MaiBot 的 `plugins/` 下，由宿主加载入口 `create_plugin()`。
- manifest 未声明依赖；代码只导入 `maibot_sdk` 与 Python 标准库，没有额外的 Python 包需要安装。
- 宿主需提供 manifest 声明的能力：`send.text`、`send.emoji`、`send.hybrid`、`send.forward`、`message.get_by_id`、`chat.get_all_streams`、`chat.get_stream_by_group_id`、`chat.get_stream_by_user_id`、`chat.open_session`、`api.call`。
- NapCat 适配器是可选的外部插件：插件通过 `adapter.napcat.action.call` 读取原始消息和直连发送，并在需要时通过 `adapter.napcat.system.get_login_info` 获取机器人账号。未安装适配器时，内容改从宿主消息记录读取、发送改走宿主接口，功能仍可用，但保真度下降（语音、视频、文件等只能以占位文本表示）。
- 本地数据写入宿主提供的 `ctx.paths.data_dir`：包在 `packs/`，审计在 `audit.jsonl`。数据目录不可用时，打包返回 `pack_store_failed`，列包返回空列表，分享按 `pack_not_found` 处理；转发与复读仍可发送，只是审计无法写入。

补录网关注册和注销成功后，日志分别显示「已就绪」和「已注销」。

## 使用示例

对机器人说：

- “把刚才这三条消息合并转发到群 123456。”
- “复读一下上一条。”
- “把这段对话收藏起来，回头分享到别的群。”
- “列出收藏，把最近那个分享到本群。”

## LLM 工具

以下工具的参数都没有默认值：标为必填的必须由 LLM 传入，缺少或为空会取消本次调用；`list_message_packs` 没有参数。

### `forward_messages`

把若干条消息转发到指定聊天：可以合并成一条合并转发，也可以逐条发送。仅在用户明确要求转发聊天记录时调用。

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `message_ids` | string[] | 是 | 要转发的消息 ID 列表 |
| `target_stream_id` | string | 是 | 目标聊天流 ID；也接受群号或对方 QQ 号 |
| `merge` | bool | 是 | `true` 合并转发，`false` 逐条转发 |

行为：

- **目标解析**：先按聊天流 ID 在 `chat.get_all_streams(platform="qq")` 的结果中匹配 `session_id` / `stream_id`；未命中且传入的是纯数字时，依次按群号（`chat.get_stream_by_group_id`）、对方 QQ 号（`chat.get_stream_by_user_id`）查找已有聊天流；仍未命中时按群号调用 `chat.open_session(chat_type="group")` 尝试打开或创建会话。**私聊没有既有聊天流时，不能用 QQ 号指定目标**，需要传该私聊的聊天流 ID。
- **参数容错**：`message_ids` 也接受逗号或空白分隔的字符串，重复 ID 会去重并保留首次出现的顺序，未传 `message_ids` 时 `message_id` 可作为单条消息的别名；目标参数接受 `target_chat_id`、`to_stream_id`、`chat_stream_id`、`target_group_id` 别名；`merge` 接受 `true/false`、`1/0`、`yes/no`、`on/off`、`是/否` 等写法，也接受 `is_merge`、`merged`、`as_merge`。
- **条数上限**：超过 `forward.max_items_per_call`（默认 20）直接拒绝，未发送任何消息。
- **整体成功或整体取消**：逐条读取消息，任意一条读不到就取消本次转发，`failed` 列出失败的消息及原因，不部分发送。
- **强制合并**：`merge=false` 时，如果 `forward.force_merge` 开启且本次最早一条消息距现在已达到 `forward.force_merge_age_days`，实际按合并转发发送，并在返回中标记 `forced_merge=true`。原本就传 `merge=true` 时不额外标注。
- **发送路径**：见「内容读取与发送」。NapCat 直连逐条发送时如果已经发出部分消息后失败，返回 `partial_send_failed`，不会回退或重发，避免重复。
- **记录**：目标为群聊时，发送成功后会尝试注入 `[转发工具 源→目标]` 记录；「源」是发起调用的当前聊天，「目标」是目标聊天。目标为私聊时只真发，不注入记录。发送成功但没有宿主记录时写审计日志。

返回：

- 成功：`success=true`、`content`、`forwarded`（条数）、`mode`（`merged` / `single`）、`forced_merge`、`target_stream_id`、`record_injected`。
- 失败：`success=false`、`error`（见错误码表）、`content`；按场景附带 `missing`、`failed` 或 `sent`。

### `repeat_message`

把一条消息在当前聊天重新发送一次。仅在用户明确要求复读某条消息时调用。

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `message_id` | string | 是 | 要复读的消息 ID |

行为：

- 当前聊天由工具调用上下文中的 `stream_id` 或触发消息的会话信息解析；`stream_id`、群号和对方 QQ 号都取不到时返回 `unknown_current_stream`。
- 复读目标固定为当前聊天，没有目标参数。
- 同聊天判定按消息的 `chat_kind` 与 `chat_id`：群聊比群号；私聊比**消息发送者 QQ**（`sender_id`）与当前聊天解析出的私聊 ID（通常是对方 QQ）。两者一致时不做名单检查；不一致时视为跨聊天，检查消息来源聊天与当前聊天。机器人自己发送的私聊消息会被当作跨聊天复读。
- 发送方式按内容形态自动选择：纯文本用 `send.text`，单个表情包用 `send.emoji`，含图片/表情包用 `send.hybrid`，含图片 URL 且没有 base64 图片/表情包时用单节点 `send.forward`，其他情况按文本发送。主路径失败且可读摘要与正文不同时，回退发送纯文本摘要。
- 语音、视频、文件、卡片、合并转发等无法直接复现的内容，以 `[语音]`、`[视频]`、`[文件]`、`[卡片/链接]`、`[合并转发消息]` 等占位文本代替。
- 群聊发送成功后会尝试注入以 `[复读工具]` 开头的记录，正文是纯文本（图片变 `[图片]`、表情包变 `[表情包]`）；私聊只真发、不入库。没有宿主记录时写审计。

返回：成功为 `success=true`、`content`、`repeated`（消息 ID）、`record_injected`；失败为 `success=false`、`error`、`content`。

### `pack_chat_messages`

把若干条消息打包收藏到本地，返回 `pack_id`，不发送到任何聊天。当用户要求收藏聊天记录，或需要把一段聊天留待之后分享时调用。

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `message_ids` | string[] | 是 | 要打包的消息 ID 列表 |
| `summary` | string | 是 | 一句话概括 |
| `description` | string | 是 | 详细描述，供之后判断包的内容 |

行为：

- 只按消息 ID 读取：先经宿主库确认消息存在，内容可能来自 NapCat；不校验消息是否属于当前聊天，也**不做群聊/私聊名单检查**。在参数合法的前提下，能否打包只取决于消息 ID 是否可读。
- 超过 `forward.max_items_per_call` 直接拒绝，未保存任何数据。
- 任意一条消息读不到就整体取消，`failed` 列出失败项。
- 包的来源聊天取**第一条消息**的聊天信息：群聊记群号/群名；私聊记**消息发送者 QQ**（`sender_id`），而不是会话对方。机器人自己发送的私聊消息会因此把来源记成机器人 QQ，分享时也按这个 ID 过私聊名单。
- 生成 `pack_id`（形如 `maiforward-pack-<12 位十六进制>`），把消息快照写入 `data_dir/packs/<pack_id>.json`，`share_count` 初始为 0。
- 快照保留转发节点段（含图片 base64 或图片 URL），之后分享不再按消息 ID 读取原消息。

返回：成功为 `success=true`、`content`、`pack_id`、`packed`（条数）、`summary`；失败为 `success=false`、`error`、`content`。

### `list_message_packs`

无参数。列出 `data_dir/packs/` 下尚未销毁的包，按 `created_at` 倒序。

本工具不按当前聊天或转发名单过滤：`data_dir/packs/` 下所有包（包括其他群聊或私聊来源的 `source`、`summary`、`description`）都会返回给 LLM。

返回：

- `success=true`、`content`（文本列表）、`count`、`packs`。
- 文本列表包含 `pack_id`、来源群/私聊、消息条数、概括与描述；`packs` 数组每项包含 `pack_id`、`summary`、`description`、`source`、`message_count`、`share_count`。
- 数据目录不存在、读取异常或单个 JSON 损坏时跳过对应条目，返回空列表或减少条目，不报错。

分享前先调用本工具取得真实 `pack_id`。

### `share_message_pack`

把指定包以一条合并转发发送到当前聊天。发送目标固定为当前聊天，不能指定其他聊天。当用户要求把已打包的聊天记录分享到当前聊天时调用；`pack_id` 需来自 `list_message_packs`。

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `pack_id` | string | 是 | 来自 `list_message_packs` 的包 ID |

行为：

- `pack_id` 必须对应 `data_dir/packs/` 中存在的包；不存在、已销毁或文件损坏时返回 `pack_not_found`。
- 名单检查：源为包的来源聊天（私聊时是包内第一条消息的发送者 QQ），目标为当前聊天，两侧都按各自的群聊/私聊名单判定；允许把包发回它原本所在的群。
- 发送固定走宿主 `send.forward`（`storage_message=False`），不使用 NapCat 直连，`forward.prefer_napcat_direct` 对它不生效。
- 第一次发送失败且节点中含 `imageurl` / `emoji` 段时，把这类段替换为 `[图片]` / `[表情包]` 文本后重试一次；仍失败则返回 `send_failed`。
- 发送成功后按配置处理包：`destroy_after_forward=true` 时立即删除；否则 `share_count` 加一，达到 `max_forward_count` 后删除。发送失败、名单拦截、包不存在、包为空都不计数。
- 群聊目标发送成功后会尝试注入 `[分享工具 源→目标]` 记录（正文是包的概括）；私聊目标不注入，返回内容会要求 LLM 告知用户已发送成功。
- 包的计数/删除写入失败不影响已完成的发送，结果中会提示状态更新失败。

返回：成功为 `success=true`、`content`、`pack_id`、`share_count`（本次分享后的计数）、`record_injected`；失败为 `success=false`、`error`、`content`。

### 工具错误码

| `error` | 出现的工具 | 含义 |
| --- | --- | --- |
| `missing_or_invalid_parameters` | 全部带参数的工具 | 必填参数缺失或为空；`missing`（若有）列出字段 |
| `too_many_items` | `forward_messages`、`pack_chat_messages` | 超过 `forward.max_items_per_call`，未发送或未保存 |
| `target_stream_not_found` | `forward_messages` | 目标无法解析为聊天流 |
| `messages_not_found` | `forward_messages`、`pack_chat_messages` | 至少一条消息读不到；`failed` 列出；整体取消 |
| `message_not_found` | `repeat_message` | 消息读不到 |
| `unknown_current_stream` | `repeat_message`、`share_message_pack` | 无法确定当前聊天 |
| `permission_denied` | `forward_messages`、`repeat_message`（跨聊天）、`share_message_pack` | 名单未通过，`content` 说明哪一侧被拒 |
| `pack_store_failed` | `pack_chat_messages` | 写 `data_dir/packs` 失败 |
| `pack_not_found` | `share_message_pack` | 包不存在、已销毁或文件损坏 |
| `pack_empty` | `share_message_pack` | 包内没有可发送的消息 |
| `send_failed` | `forward_messages`、`repeat_message`、`share_message_pack` | 发送接口返回失败；`content` 含原因 |
| `partial_send_failed` | `forward_messages` | NapCat 直连已发出部分消息后失败；`sent` 为已发条数，不重发 |

工具内部异常时返回 `success=false` 与 `content`（含「插件内部异常」），不带 `error` 字段。

## 内容读取与发送

### 消息来源

工具参数不包含消息正文，转发正文一律由插件读取：

1. 先调用 `message.get_by_id(message_id, include_binary_data=True)` 从宿主消息库取消息。取不到即判定该消息读取失败，后续 NapCat 不再尝试。
2. 消息平台为 `qq` 时，再尝试用 NapCat 的 `get_msg` 读取原始消息段。候选 ID 包括传入 ID、宿主记录里的 `message_id` / `platform_message_id` / `napcat_message_id` 等字段，以及传入 ID 末尾的数字串；命中后还要与宿主记录核对发送者、群号和消息时间（容差 300 秒），不一致就放弃该候选，继续尝试其他候选或改用宿主记录。
3. NapCat 不可用或校验不通过时，用宿主记录的 `raw_message` 消息段；没有消息段时退回 `processed_plain_text`。
4. 文本、@、回复引用、图片（base64 或 URL）、表情包尽量保留；语音、视频、文件、卡片/链接、合并转发等转为占位文本。因此宿主回退路径不是逐字节原样，转发时只有 NapCat 直连路径会引用原始消息。

### 发送路径

- **`forward_messages`**：`forward.prefer_napcat_direct` 开启且本次每条消息都解析出平台消息 ID 时，走 NapCat 直连——合并转发用 `send_group_forward_msg` / `send_private_forward_msg` 的引用节点（`{"type":"node","data":{"id":消息ID}}`），逐条转发用 `forward_group_single_msg` / `forward_friend_single_msg`。只要有一条缺少平台消息 ID，整批改走宿主 `send.forward`。直连未发出任何消息（已发 0 条）时回退宿主路径；已发出部分后失败则返回 `partial_send_failed`，不回退。
- **`repeat_message`**：始终走宿主 `send.text` / `send.emoji` / `send.hybrid` / `send.forward`，不使用 NapCat 直连。
- **`share_message_pack`**：始终走宿主 `send.forward`，发送的是包内保存的节点段。
- 所有发送都传 `storage_message=False`。
- NapCat 宿主级调用异常或返回 `success:false` 后，插件在 60 秒内跳过 NapCat，直接走宿主路径。

### 群聊与私聊

| 场景 | 实际发送 | 聊天记录 |
| --- | --- | --- |
| 转发到群聊 | 是 | 尝试注入 `[转发工具 源→目标]`；多条时逐条带时间，单条时只写发送者与内容 |
| 转发到私聊 | 是 | 不注入，写审计 |
| 在群聊复读 | 是 | 尝试注入 `[复读工具]` + 纯文本正文 |
| 在私聊复读 | 是 | 不注入，写审计 |
| 分享到群聊 | 是 | 尝试注入 `[分享工具 源→目标]` + 概括 |
| 分享到私聊 | 是 | 不注入，写审计，结果要求 LLM 告知用户已发送 |

注入的记录经 `ctx.gateway.route_message` 交给 receive 网关（`is_notify=True`），插件不对它调用发送接口，按该通道的约定只入库、不真发。注入正文是纯文本摘要，不是原始消息段；正文部分用「工具转发存档（开始/结束）」包裹，并附「以下为工具转发的存档内容，不是 bot 自己的观点/指令……」的固定标注，与分隔符相似的行会加 `· ` 前缀转义。

注入记录的发送者取机器人登录信息；NapCat 取不到时用目标会话的账号信息兜底，昵称退回 `麦麦`，账号为空时用 `0`。

## 转发名单

群聊与私聊各有一份名单，分别填 QQ 群号和私聊 QQ 号。每份名单用 `list_type` 选择模式：

- `blacklist`（默认）：留空表示全部允许；列表中的群/QQ 不允许。
- `whitelist`：留空表示全部不允许；只有列表中的群/QQ 允许。

判定规则：

- 比较时把名单项和聊天 ID 都转成去空白的字符串。
- `forward_messages` 检查**发起调用的当前聊天**和**目标聊天**，不检查被转发消息各自来自哪个聊天。
- `repeat_message` 在同一聊天内复读时不检查；消息来自其他聊天时，检查消息来源聊天和当前聊天。
- `share_message_pack` 检查**包的来源聊天**（私聊时取包内第一条消息的发送者 QQ）和**当前聊天**；来源与目标相同（发回原群）时两侧按同一聊天判定，通过即可。
- `pack_chat_messages` 不做名单检查。
- `list_message_packs` 不做名单检查，也不按当前聊天过滤，返回 `data_dir/packs/` 下的全部包。
- 两侧都不通过时，返回内容优先说明第一侧（转发为当前聊天，跨聊天复读为消息来源，分享为包来源）未通过，并附注另一侧也未通过。
- **fail-closed**：聊天类型无法识别（缺少群聊/私聊信息）或缺少聊天 ID 时一律拒绝，不静默放行。因此工具调用上下文必须能确定当前聊天，否则转发会被名单检查拒绝，分享会因无法确定当前聊天而取消。

## 强制合并

`forward.force_merge`（默认开启）控制旧消息是否强制合并：

- 只作用于 `forward_messages`。
- 以本次转发中**最早一条**消息的时间计算年龄；达到 `forward.force_merge_age_days`（默认 2 天，按 86400 秒/天）时，最终按合并转发发送，忽略 LLM 传入的 `merge=false`。
- 只有 LLM 原本传了 `merge=false` 时才把 `forced_merge` 标为 `true` 并在返回文案中说明；原本传 `merge=true` 不额外标注。
- 消息时间缺失时按当前时间参与计算。

## 打包记录

- `pack_chat_messages` 生成 `pack_id`（`maiforward-pack-<12 位十六进制>`），写入 `data_dir/packs/<pack_id>.json`。文件包含 `pack_id`、`summary`、`description`、`created_at`、`source`、`items`、`share_count`。
- `source` 取第一条消息的聊天信息（私聊为消息发送者 QQ）；`items` 是消息快照（发送者、时间、可读摘要、转发节点段等），分享时直接使用快照，不再读取原消息。
- 没有基于时间的过期或定时清理：包文件写入 `data_dir/packs/`，在宿主清理数据目录前会一直保留，直到分享触发销毁条件；也可以直接删除对应的 JSON 文件。
- `forward.destroy_after_forward=true`（默认）：每次分享成功后立即删除该包，`max_forward_count` 不生效。
- `forward.destroy_after_forward=false`：每次分享成功后 `share_count` 加一；达到 `max_forward_count`（默认 3，最小 1）时删除，否则写回文件。
- 发送失败、名单拦截、包不存在、包为空都不消耗次数。
- 分享已成功但删除/写回失败时，发送结果仍为成功，返回文案会提示包状态更新失败。
- 包内可能包含图片 base64，文件体积会随消息数量和图片大小增长；包内容来自聊天记录，注意 `data_dir` 的访问权限。

## 配置

| 配置项 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `plugin.enabled` | bool | `true` | 配置模型中的启用字段；插件代码不读取该值，停用请使用宿主的插件管理 |
| `plugin.config_version` | str | `"0.1.4"` | 配置版本元数据，WebUI 隐藏且禁用，代码不读取 |
| `group_permission.list_type` | `"blacklist"` / `"whitelist"` | `"blacklist"` | 群聊名单模式 |
| `group_permission.id_list` | string[] | `[]` | QQ 群号列表 |
| `private_permission.list_type` | `"blacklist"` / `"whitelist"` | `"blacklist"` | 私聊名单模式 |
| `private_permission.id_list` | string[] | `[]` | 私聊 QQ 号列表（通常为对方 QQ） |
| `forward.record_group_history` | bool | `true` | 在 WebUI 聊天记录中补录群聊转发、复读和分享 |
| `forward.force_merge` | bool | `true` | 旧消息强制合并（仅 `forward_messages`） |
| `forward.force_merge_age_days` | float，≥ 0 | `2.0` | 强制合并阈值（天） |
| `forward.prefer_napcat_direct` | bool | `true` | 优先 NapCat 直连（仅 `forward_messages`） |
| `forward.destroy_after_forward` | bool | `true` | 分享成功后立即删除包 |
| `forward.max_forward_count` | int，≥ 1 | `3` | 关闭分享后销毁时的最大分享次数 |
| `forward.max_items_per_call` | int，≥ 1 | `20` | `forward_messages` / `pack_chat_messages` 单次条数上限 |

在插件配置页的「转发行为」中切换「群聊记录补录」，保存后热生效。关闭时，MaiBot 注销该插件的 QQ 补录网关。QQ 发送照常进行，发送成功后写入本地审计。重新开启时会重新注册网关。

## 审计日志

本地审计文件为 `data_dir/audit.jsonl`，每次发送成功但没有留下宿主聊天记录时追加一行 JSON：

- 转发、复读或分享到私聊时会写审计。群聊补录关闭或失败时同样记录。
- 字段：`time`（本地时间）、`action`（`forward` / `repeat` / `share`）、`trigger_user`（触发消息的发送者 QQ，缺失为 `未知`）、`source`、`target`、`message_count`。
- 不记录消息内容。
- 写入失败只记 warning，不影响已经完成的发送。

## 使用注意

- 三个工具会真的发消息：`forward_messages`、`repeat_message`、`share_message_pack`。`pack_chat_messages` 只写本地文件，`list_message_packs` 只读本地文件。
- 群聊中发送成功后会尝试额外注入一条标注记录，但注入正文是纯文本摘要：图片变 `[图片]`、表情包变 `[表情包]`、语音/视频/文件/卡片变占位文本。分隔符、免责标注和转义只能降低、不能消除转发内容对宿主模型的二次提示注入风险。
- `forward_messages` 的 `[转发工具 源→目标]` 中「源」是发起调用的当前聊天，不是每条消息各自的来源聊天；插件也不校验被转发的消息是否属于当前聊天。名单检查同样只覆盖当前聊天和目标聊天。
- `pack_chat_messages` 不做名单检查，也不校验消息归属，只按消息 ID 读取；能拿到消息 ID 的调用方，只要该消息在宿主库中可读，就能把它打包。包文件保存消息内容快照和图片数据，且没有自动过期，请按数据敏感度管理 `data_dir`。
- `share_message_pack` 只能发到当前聊天；分享到私聊不会留下聊天记录，结果会要求 LLM 明确告知用户发送成功。
- 私聊消息的 `chat_id` 取**消息发送者**，不是会话对方：机器人自己发送的私聊消息会把来源记成机器人 QQ，从而影响打包来源、分享来源名单和同聊天复读判断。
- `list_message_packs` 返回 `data_dir/packs/` 下的全部包，不按当前聊天或名单过滤；多会话部署时，其他群/私聊的来源、概括与描述也会进入 LLM 上下文。
- 名单判定依赖工具调用上下文。上下文缺失、聊天类型或 ID 无法识别时会拒绝转发/分享；同聊天复读不受名单限制。
- NapCat 直连通过适配器通用 action 调用，不携带账号参数；多账号部署若需要严格按会话账号发送，可关闭 `forward.prefer_napcat_direct`，统一走宿主路径。
- 插件没有斜杠命令和 Hook，所有操作都由 LLM 工具触发。

## 许可证与仓库

- 许可证：MIT，见 [LICENSE](LICENSE)。
- 版本历史：[CHANGELOG.md](CHANGELOG.md)。
- 仓库：<https://github.com/cateyemizuki/cateye_mai_forward>
- 问题反馈：<https://github.com/cateyemizuki/cateye_mai_forward/issues>
