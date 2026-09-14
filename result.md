# adapter-qq 新版 mention 实测简报（Issue #220 / PR #223 / #225）

**实测方式**：自建测试机器人 + 抓包转发代理，抓取真正上网的请求体与服务端返回码。
场景覆盖：群聊 / 单聊 / 文字子频道 × 7 个探针。基线：本地仓库与上游 master `c7dca0b` 完全一致（#223 未合入）。

## 结论速览

| # | 结论 | 性质 |
| --- | --- | --- |
| 1 | 适配器把用户手写的**新格式降级回旧格式**再发出，新旧格式上网字节一字不差 | 关键 |
| 2 | 群聊 `@全体成员` 的结构化信息**被丢弃**（`GroupMentionEveryone` 全仓无消费点） | 确定性 bug |
| 3 | `@everyone` 字面量被**静默清洗**，服务端 200 不报错 | 确定性 bug |
| 4 | `@全体成员` 实测**群聊/单聊成功、文字子频道失败**，与官方文档完全相反 | 文档错误 |
| 5 | 频道不支持原生 markdown（400 `50056`），适配器无场景校验 | 缺校验 |
| 6 | text + markdown 混排产出 `{"msg_type":2,"content":"...","markdown":{...}}` | 印证 #225 |

## 证据

### 1. 新格式被降级回旧格式（直接影响 #223 的发送侧方案）

`_construct` 认得新格式（`message.py:600`），会解析成 `MentionUser`；但 `MentionUser.__str__`
（`message.py:302-303`）仍然吐旧格式：

```python
def __str__(self) -> str:
    return f"<@{self.data['user_id']}>"
```

抓包实录（群聊）：

| 探针 | 输入 | 实际上网 body |
| --- | --- | --- |
| p3 | `<qqbot-at-user id="U" /> 你好` | `{"content": "<@U> 你好", "msg_type": 0}` |
| p4 | `<@U> 你好` | `{"content": "<@U> 你好", "msg_type": 0}` |

**两者字节完全相同。** 含义：只改解析、不改发送 → 上网字节零变化；只改发送、不改解析 → 用户手写新格式
会被当普通文本。**两侧必须一起改** —— 这反过来支持把 #223 的接收侧与发送侧放在同一个 PR。

### 2. 群聊 `@全体成员` 信息丢失（确定性 bug）

`GroupMentionEveryone`（`models/qq.py:34`）已定义、已导出（`:363`）、也已纳入 `mentions` 联合类型，
但 `from_qq_message` 只消费 `GroupMentionUser`（`message.py:656`）：

```python
mentions = {m.id: m for m in message.mentions if isinstance(m, GroupMentionUser)}
```

→ "某人 @了全体成员"这一事实在 `Message` 层彻底消失，业务代码无法感知。
**这是与格式新旧之争无关的确定性缺陷，接收侧修它没有争议。**

### 3. `@everyone` 字面量被静默清洗

`_construct` 开头两行无条件删除（`message.py:579-580`）：

```python
msg = msg.replace("@everyone", "")
msg = re.sub(r"\<qqbot-at-everyone\s/\>", "", msg)
```

实测 `Message("@everyone 公告")` → 上网 `{"content": " 公告"}`，服务端 200 不报错，客户端无任何提示。

### 4. `@全体成员` 的行为与官方文档相反

文档（text-chain 页）称"`@全部成员` 仅在文字子频道可用"。实测：

| 场景 | 请求体 | 结果 |
| --- | --- | --- |
| 群聊 | `{"content": "@everyone 公告", "msg_type": 0}` | **200 成功** |
| 单聊 | 同上 | **200 成功** |
| 文字子频道 | `{"content": "@everyone 公告"}` | **400 `40034117 消息发送失败`** |

**恰好相反。** 这一点是 PR 长期卡在 open 的直接原因：维护者倾向"用文档说明"，
而文档本身与线上行为矛盾，无从说明。

### 5. 频道不支持原生 markdown

`POST /channels/{channel_id}/messages` 带 markdown 段 → 400 `50056 不允许发送原生 markdown`。
适配器不做场景校验，把群聊代码原样搬到频道会直接失败。

### 6. 印证 #225

显式混排 text 段与 markdown 段时产出：

```json
{"content": "hi ", "msg_type": 2, "markdown": {"content": "..."}}
```

`msg_type=2` 却同时带非空 `content`，与文档"二者互斥"不符。
（单独只发 markdown 时 `content` 正确留空，合规。）

## 需要说明的边界（避免过度解读）

- **服务端对所有 `<@openid>` 内容一律返回 200、不回执**，因此"text 消息下 at 是否真的不生效"
  在接口层**无法证实或证伪**，只能靠客户端肉眼确认。本文不主张该点，只主张上面 6 条有抓包支撑的事实。
- 由第 1 条可推：既然新旧格式的上网字节完全相同，"新格式能不能用"就不是问题所在，
  真正的影响因子只可能是**消息类型（markdown / text）** —— 这与 #220 中作者本人"闹乌龙"的自述一致。

## 建议

1. **拆分 #223**：接收侧（`_construct` 补 `qqbot-at-everyone` 分支 + `from_qq_message` 消费
   `GroupMentionEveryone`）无争议，建议独立提交先合；发送侧（`__str__` 换新格式）与 #225 耦合，
   可等 #225 定稿后再动。
2. `@everyone` 的静默清洗至少应加 warning，或仅在特定上下文才清洗。
3. text-chain 文档"文本消息支持 at"与"`@全部成员` 仅在文字子频道可用"两处与实测冲突，
   建议向官方反馈，并在适配器文档中写明实际边界。
