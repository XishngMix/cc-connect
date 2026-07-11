# 飞书 interactive 卡片消息支持 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 飞书平台主分发路径支持 `interactive` 卡片消息(提取为可读文本喂给 agent),并增强 legacy 卡片提取器(链接/备注/@)。

**Architecture:** 在 `platform/feishu/feishu.go` 的 `dispatchMessage` 消息类型 switch 中新增 `case "interactive"`,复用已有的 `extractInteractiveCardText`;将该函数的 legacy elements 提取循环抽为可递归的辅助函数 `extractLegacyCardElements`,补 `a`/`note`/`at` 三类元素。不改 @ 门槛、不加配置、不动 core/。

**Tech Stack:** Go,标准库 `encoding/json`,测试用 `go test`(现有 stub 模式,无外部依赖)。

**Spec:** `docs/superpowers/specs/2026-07-12-feishu-interactive-card-design.md`

## Global Constraints

- 每个 shell 会话先初始化 goenv,否则找不到 `go`:`export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"`
- 所有 `go build` / `go test` 必须带 `-tags no_web`(本机无 web/dist 前端产物)
- 仅改 `platform/feishu/` 下文件;`core/`、配置、i18n 均不改
- 遵循 CLAUDE.md:错误用 `fmt.Errorf(... %w)` 包装,运行时日志用 `slog`
- 本机已知与本改动无关的失败:`daemon` 包 `TestLaunchdStatusUsesUserDomainWhenGUIDomainUnavailable`;`agent/codex` 全量并行时偶发超时。全量测试时忽略这两项

---

### Task 1: legacy 卡片提取器增强(a / note / at)

**Files:**
- Modify: `platform/feishu/feishu.go:2262-2270`(`extractInteractiveCardText` 内的 legacy elements 循环)
- Test: `platform/feishu/feishu_test.go`(追加到文件末尾)

**Interfaces:**
- Consumes: `extractInteractiveCardText(content string) string`(已存在,feishu.go:2196)
- Produces: `extractLegacyCardElements(elements []json.RawMessage, parts *[]string)` — 新辅助函数,Task 2 不直接使用,但 Task 2 的分发测试断言依赖本任务的提取输出格式(markdown 链接 `[text](href)`)

- [ ] **Step 1: Write the failing test**

在 `platform/feishu/feishu_test.go` 文件末尾追加:

```go
func TestExtractInteractiveCardText_LegacyElements(t *testing.T) {
	tests := []struct {
		name    string
		content string
		want    string
	}{
		{
			name:    "text and title",
			content: `{"title":"Alert Title","elements":[[{"tag":"text","text":"line one"}]]}`,
			want:    "Alert Title\nline one",
		},
		{
			name:    "a tag becomes markdown link",
			content: `{"title":"T","elements":[[{"tag":"a","href":"https://grafana.example/d/1","text":"Grafana"}]]}`,
			want:    "T\n[Grafana](https://grafana.example/d/1)",
		},
		{
			name:    "a tag without href falls back to text",
			content: `{"title":"T","elements":[[{"tag":"a","href":"","text":"plain"}]]}`,
			want:    "T\nplain",
		},
		{
			name:    "note nested elements extracted",
			content: `{"title":"T","elements":[[{"tag":"note","elements":[{"tag":"text","text":"📌 remark"}]}]]}`,
			want:    "T\n📌 remark",
		},
		{
			name:    "at with user_name",
			content: `{"title":"T","elements":[[{"tag":"at","user_id":"@_user_1","user_name":"张三"}]]}`,
			want:    "T\n@张三",
		},
		{
			name:    "at with empty user_name skipped",
			content: `{"title":"T","elements":[[{"tag":"at","user_id":"@_user_1","user_name":""},{"tag":"text","text":"tail"}]]}`,
			want:    "T\ntail",
		},
		{
			name:    "real alert card shape with hr ignored",
			content: `{"title":"[Trigger]Pod CPU Throttling","elements":[[{"tag":"text","text":"告警事件"},{"tag":"a","href":"https://pd.example/i/Q1","text":"Q1"}],[{"tag":"hr"}],[{"tag":"note","elements":[{"tag":"text","text":"📌 过去 5 分钟内 pod CPU 限制率增长超过 100%"}]}]]}`,
			want:    "[Trigger]Pod CPU Throttling\n告警事件\n[Q1](https://pd.example/i/Q1)\n📌 过去 5 分钟内 pod CPU 限制率增长超过 100%",
		},
	}
	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			if got := extractInteractiveCardText(tt.content); got != tt.want {
				t.Errorf("extractInteractiveCardText() = %q, want %q", got, tt.want)
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"
go test -tags no_web ./platform/feishu/ -run TestExtractInteractiveCardText_LegacyElements -v
```

Expected: FAIL。`text and title` 子用例应 PASS(既有行为),`a tag becomes markdown link`、`note nested elements extracted`、`at with user_name` 等子用例 FAIL(当前循环只提取 `tag=="text"`)。

- [ ] **Step 3: Write minimal implementation**

在 `platform/feishu/feishu.go` 中,将 `extractInteractiveCardText` 内的 legacy 提取循环(当前 2262-2270 行):

```go
		for _, raw := range elements {
			var elem struct {
				Tag  string `json:"tag"`
				Text string `json:"text"`
			}
			if json.Unmarshal(raw, &elem) == nil && elem.Tag == "text" && strings.TrimSpace(elem.Text) != "" {
				parts = append(parts, elem.Text)
			}
		}
```

替换为:

```go
		extractLegacyCardElements(elements, &parts)
```

并在 `extractInteractiveCardText` 函数之后新增辅助函数:

```go
// extractLegacyCardElements extracts readable text from legacy-format card
// elements: plain text, hyperlinks (as markdown), @mentions, and note
// containers (recursed into).
func extractLegacyCardElements(elements []json.RawMessage, parts *[]string) {
	for _, raw := range elements {
		var elem struct {
			Tag      string            `json:"tag"`
			Text     string            `json:"text"`
			Href     string            `json:"href"`
			UserName string            `json:"user_name"`
			Elements []json.RawMessage `json:"elements"`
		}
		if json.Unmarshal(raw, &elem) != nil {
			continue
		}
		switch elem.Tag {
		case "text":
			if strings.TrimSpace(elem.Text) != "" {
				*parts = append(*parts, elem.Text)
			}
		case "a":
			if strings.TrimSpace(elem.Text) == "" {
				continue
			}
			if elem.Href != "" {
				*parts = append(*parts, fmt.Sprintf("[%s](%s)", elem.Text, elem.Href))
			} else {
				*parts = append(*parts, elem.Text)
			}
		case "at":
			if strings.TrimSpace(elem.UserName) != "" {
				*parts = append(*parts, "@"+elem.UserName)
			}
		case "note":
			extractLegacyCardElements(elem.Elements, parts)
		}
	}
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"
go test -tags no_web ./platform/feishu/ -run TestExtractInteractiveCardText_LegacyElements -v
```

Expected: PASS(全部 7 个子用例)。

- [ ] **Step 5: Run the whole feishu package to check for regressions**

```bash
export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"
go test -tags no_web ./platform/feishu/
```

Expected: `ok`(无失败)。

- [ ] **Step 6: Commit**

```bash
git add platform/feishu/feishu.go platform/feishu/feishu_test.go
git commit -m "feat(feishu): extract links, notes and mentions from legacy card elements"
```

---

### Task 2: 主分发路径新增 `case "interactive"`

**Files:**
- Modify: `platform/feishu/feishu.go:1672-1674`(`dispatchMessage` 的 switch,`case "media"` 结束之后、`default` 之前插入)
- Test: `platform/feishu/platform_test.go`(追加到文件末尾)

**Interfaces:**
- Consumes: `extractInteractiveCardText(content string) string`(Task 1 增强后的版本;输出含 markdown 链接);`p.dispatchCoreMessage(*core.Message)`;switch 作用域内已有变量 `content`, `quoted`, `sessionKey`, `messageID`, `userID`, `userName`, `chatName`, `rctx`, `createTimeMs`
- Produces: 无新导出接口;行为变更 — `interactive` 消息将以提取文本作为 `core.Message.Content` 分发

- [ ] **Step 1: Write the failing test**

在 `platform/feishu/platform_test.go` 文件末尾追加(`stringPtr` 辅助函数已存在于本文件;`lark`/`larkim`/`core` 等 import 已存在):

```go
func TestInteractiveCard_Dispatch(t *testing.T) {
	const cardContent = `{"title":"[Trigger]Pod CPU Throttling","elements":[[{"tag":"text","text":"告警事件"},{"tag":"a","href":"https://pd.example/i/Q1","text":"Q1"}]]}`

	tests := []struct {
		name          string
		chatType      string
		groupReplyAll bool
		mentionBot    bool
		senderType    string
		wantPass      bool
	}{
		{"p2p card dispatched", "p2p", false, false, "user", true},
		{"group card with group_reply_all dispatched", "group", true, false, "user", true},
		{"group card without mention ignored", "group", false, false, "user", false},
		{"group card from webhook bot with at-bot dispatched", "group", false, true, "bot", true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			p, err := newPlatform("feishu", lark.FeishuBaseUrl, map[string]any{
				"app_id": "cli_xxx", "app_secret": "secret",
				"enable_feishu_card": true,
				"group_reply_all":    tt.groupReplyAll,
			})
			if err != nil {
				t.Fatalf("newPlatform() error = %v", err)
			}
			ip := p.(*interactivePlatform)
			ip.botOpenID = "ou_bot"

			msgCh := make(chan *core.Message, 1)
			ip.handler = func(_ core.Platform, msg *core.Message) {
				msgCh <- msg
			}

			var mentions []*larkim.MentionEvent
			if tt.mentionBot {
				mentions = []*larkim.MentionEvent{{Id: &larkim.UserId{OpenId: stringPtr("ou_bot")}}}
			}

			messageID := "om_card_" + tt.name
			chatID := "oc_card_test"
			msgType := "interactive"
			content := cardContent
			senderOpenID := "ou_card_sender"
			senderType := tt.senderType
			chatType := tt.chatType
			createTime := strconv.FormatInt(time.Now().UnixMilli(), 10)

			if err := ip.onMessage(context.Background(), &larkim.P2MessageReceiveV1{
				Event: &larkim.P2MessageReceiveV1Data{
					Sender: &larkim.EventSender{
						SenderId:   &larkim.UserId{OpenId: &senderOpenID},
						SenderType: &senderType,
					},
					Message: &larkim.EventMessage{
						MessageId:   &messageID,
						ChatId:      &chatID,
						ChatType:    &chatType,
						MessageType: &msgType,
						Content:     &content,
						CreateTime:  &createTime,
						Mentions:    mentions,
					},
				},
			}); err != nil {
				t.Fatalf("onMessage() error = %v", err)
			}

			select {
			case msg := <-msgCh:
				if !tt.wantPass {
					t.Fatal("expected interactive card to be ignored, but it was dispatched")
				}
				if !strings.Contains(msg.Content, "[Trigger]Pod CPU Throttling") {
					t.Errorf("Content missing card title, got %q", msg.Content)
				}
				if !strings.Contains(msg.Content, "[Q1](https://pd.example/i/Q1)") {
					t.Errorf("Content missing markdown link, got %q", msg.Content)
				}
			case <-time.After(2 * time.Second):
				if tt.wantPass {
					t.Fatal("expected interactive card to be dispatched, but it was ignored")
				}
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

```bash
export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"
go test -tags no_web ./platform/feishu/ -run TestInteractiveCard_Dispatch -v
```

Expected: FAIL。三个 `wantPass=true` 子用例超时失败(interactive 落入 default 被忽略);`group card without mention ignored` 子用例 PASS。

- [ ] **Step 3: Write minimal implementation**

在 `platform/feishu/feishu.go` 的 `dispatchMessage` switch 中,`case "media"` 的 `p.dispatchCoreMessage(...)` 结束之后(当前 1672 行)、`default:`(当前 1674 行)之前插入:

```go
	case "interactive":
		text := extractInteractiveCardText(content)
		p.dispatchCoreMessage(&core.Message{
			SessionKey: sessionKey, Platform: p.platformName,
			MessageID: messageID,
			UserID:    userID, UserName: userName, ChatName: chatName,
			Content: text, ExtraContent: quoted.text, Images: quoted.images, ReplyCtx: rctx,
			UserMessageTimeMs: createTimeMs,
		})
```

- [ ] **Step 4: Run test to verify it passes**

```bash
export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"
go test -tags no_web ./platform/feishu/ -run TestInteractiveCard_Dispatch -v
```

Expected: PASS(全部 4 个子用例)。

- [ ] **Step 5: Full build and feishu package regression**

```bash
export GOENV_ROOT="$HOME/.goenv"; export PATH="$GOENV_ROOT/bin:$PATH"; eval "$(goenv init -)"
go build -tags no_web ./... && go vet -tags no_web ./platform/feishu/ && go test -tags no_web ./platform/feishu/
```

Expected: build 无输出,vet 无输出,test `ok`。

- [ ] **Step 6: Commit**

```bash
git add platform/feishu/feishu.go platform/feishu/platform_test.go
git commit -m "feat(feishu): dispatch interactive card messages to agent"
```
