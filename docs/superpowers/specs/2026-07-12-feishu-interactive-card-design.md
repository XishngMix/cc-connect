# 飞书 interactive 卡片消息支持 — 设计文档

日期:2026-07-12
状态:已确认

## 背景

飞书平台当前不处理 `message_type: "interactive"` 的卡片消息:主分发 switch(`platform/feishu/feishu.go` `dispatchMessage`)没有对应 case,卡片落入 default 分支被忽略(`ignoring unsupported message type`)。用户把告警卡片(PagerDuty/Grafana 等)转发到群里,或告警侧通过自定义 webhook 机器人直发卡片并 @ 机器人时,agent 均无响应;目前唯一的使用方式是"引用卡片 + @bot + 文字说明"。

卡片文本提取函数 `extractInteractiveCardText` 已存在,但只挂在引用消息(`fetchSingleMessage`)和合并转发子消息路径上。

## 目标

1. 主分发路径支持 interactive 卡片消息,提取为可读文本后分发给 agent。
2. 增强 legacy 格式提取器,减少告警卡片的信息丢失(链接、备注、@)。

## 非目标

- 不改动群聊 @ 门槛及 `thread_isolation` 豁免列表(群内免 @ 触发由既有 `group_reply_all` 承担)。
- 不新增配置项。
- 不解析 `user_dsl` 字段(非标准字段,legacy 提取已覆盖所需信息)。
- 不把卡片原始 JSON 附带给 agent(体积大、双重转义、与提取文本冗余)。
- 不新增 webhook 直连触发入口(独立 feature)。

## 设计

### 1. 主分发路径新增 `case "interactive"`

在 `dispatchMessage` 的消息类型 switch(default 分支之前)新增:

```go
case "interactive":
    text := extractInteractiveCardText(content)
    p.dispatchCoreMessage(&core.Message{
        SessionKey: sessionKey, Platform: p.platformName,
        MessageID: messageID,
        UserID:    userID, UserName: userName, ChatName: chatName,
        Content: text, ExtraContent: quoted.text, Images: quoted.images,
        ReplyCtx:          rctx,
        UserMessageTimeMs: createTimeMs,
    })
```

- 结构与 `text` 分支一致;不调用 `stripMentions`(卡片 JSON 不含 mention 占位实体,mentions 在事件顶层)。
- `extractInteractiveCardText` 提取失败时返回 `"[interactive card]"` 占位符,仍然分发——agent 至少知道收到了一张卡片。
- 引用上下文(`quoted`)照常生效:卡片消息若本身引用了别的消息,被引内容进 `ExtraContent`。

### 2. legacy 提取器增强(`extractInteractiveCardText`)

legacy elements 循环(当前只提取 `tag == "text"`)扩展支持:

| 元素 | 处理方式 |
|---|---|
| `a` | 输出 `[text](href)` markdown 链接;href 为空时退化为纯文本 |
| `note` | 嵌套 `elements` 结构,递归提取其中的文本元素 |
| `at` | 输出 `@user_name`;`user_name` 为空则跳过 |

schema 2.0 路径(`extractCardElements`)不改动。引用卡片、合并转发路径复用同一提取器,自动受益。

### 3. 行为矩阵

| 场景 | 行为 |
|---|---|
| 私聊直发/转发卡片 | 直接处理(私聊无 @ 门槛) |
| 群聊转发卡片,`group_reply_all = true` | 直接处理 |
| 群聊转发卡片,未开 `group_reply_all`、无 @bot | 维持现状:忽略(需引用 + @bot) |
| 群聊 webhook 机器人直发卡片并 @ 本机器人 | 直接处理(@ 门槛按 mention open_id 匹配通过,无需 `group_reply_all`) |
| 引用卡片 + @bot + 说明 | 维持现状,提取质量随提取器增强提升 |

### 4. 机器人直发卡片的使用前提(文档性说明,无代码)

- 飞书应用需开通 `im:message.group_at_msg.include_bot:readonly` 权限(接收群内用户及其他机器人 @ 本机器人的消息)。`im:message.group_msg` 明确**不含**机器人消息,其他机器人不 @ 本机器人的消息飞书不会推送,平台机制无法绕过。
- 自定义 webhook 机器人的 sender open_id 非空(实测 `ou_` 开头),`allow_from` 配置了名单时需将该 open_id 加入名单(或使用 `"*"`)。

### 5. 已知限制

- legacy 卡片的行内片段(如 `"◾ "`、`"告警事件"`、`": "`)逐个 append 后用 `\n` join,原本同一行的内容会拆行。既有行为,本次不改(纯外观,不影响 agent 理解)。
- 卡片的纯视觉信息(header 颜色模板、字段布局)不提取,对分析无价值。

## 测试

均为单元测试(`platform/feishu`),沿用现有 stub/fixture 模式:

1. **回归**:群聊、未开 `group_reply_all`、无 @bot 的 interactive 消息仍被忽略。
2. 群聊 + `group_reply_all = true` 的 interactive 消息被分发,`Content` 含卡片标题与正文。
3. p2p 私聊 interactive 消息被分发。
4. 群聊 interactive 消息带 @bot mention(模拟 webhook 机器人直发场景)、未开 `group_reply_all`,被分发。
5. `extractInteractiveCardText`:含 `a` 标签的 legacy 卡片输出 markdown 链接;含 `note` 嵌套元素的内容被提取;含 `at` 元素输出 `@名称`(fixture 取自真实告警卡片结构)。

## 影响范围

- 仅 `platform/feishu/feishu.go` 及其测试文件;`core/` 无改动,无 i18n 新文案,无配置变更。
