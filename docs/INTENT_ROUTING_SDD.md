# CC Switch 意图路由功能设计说明（SDD）

## 1. 背景与目标

### 1.1 问题背景

在典型使用场景中，用户往往会为 Claude CLI 同时配置多个供应商和模型，例如：

- 不同云厂商（官方 / 代理）各自提供 Claude 接口；
- 同一供应商下存在多种模型能力组合（通用对话、长上下文、代码增强、数学推理等）；
- 某些供应商专门用于特定任务（如复杂数学、超长文本分析）。

如果完全依赖用户手动在 CC Switch 主界面切换“当前供应商”，会出现几个问题：

1. 同一任务中频繁切换供应商的操作成本高；
2. 用户难以记住“当前这个问题适合哪个供应商”；
3. 对 CLI 用户而言，频繁建议“请切回某某供应商”体验不佳。

### 1.2 目标

意图路由功能旨在：

- 在多个供应商之间，根据“本次请求的意图”和各供应商的描述，自动选择最合适的供应商；
- 仅在**会话的首轮请求**执行跨供应商选择，避免同一任务中途切换供应商；
- 在供应商内部，根据请求长度和模型能力，在多个模型之间做细粒度选择；
- 保持与现有代理 / 故障转移机制兼容，不破坏已有行为。

当前范围仅覆盖 **Claude CLI `/v1/messages`** 请求。

---

## 2. 功能概览

### 2.1 用户视角

从用户角度看，意图路由提供：

1. 设置页开关：
   - 「设置 → 高级 → 代理服务 → 启用 Claude CLI 意图路由」
   - 可选「路由专用供应商」下拉框，用于指定负责决策的供应商 / 模型。
2. 主界面快捷开关：
   - 当代理和 Claude 接管开启时，在主界面顶部显示一个 ✨ 图标 + 开关。
3. 供应商级配置：
   - 在「编辑供应商」中为每个 Claude 供应商配置“意图路由描述”（`intent_description`）。

启用后，用户在 Claude CLI 中发起新会话的首轮提问时，CC Switch 会自动为该会话选择供应商和模型；后续对话轮次保持在同一供应商上继续。

### 2.2 系统视角

系统内部将意图路由拆分为两层：

- **供应商级（跨 API Key）路由**：
  - 在多个 Claude 供应商之间选择最合适的一个；
  - 基于供应商描述（`intent_description` / `notes`）和当前请求摘要；
  - 仅在“首轮请求”时触发。

- **模型级（单供应商内部）路由**：
  - 在同一供应商的多个模型（默认、Haiku/Sonnet/Opus/Reasoning 等）之间选择；
  - 基于请求长度和模型类型；
  - 可作为跨供应商路由失败时的回退策略。

---

## 3. 流程设计

### 3.1 Claude `/v1/messages` 请求处理总览

相关入口：`src-tauri/src/proxy/handlers.rs::handle_messages`

```text
handle_messages
  ├─ 构建 RequestContext（包含 session_id、当前 provider 链等）
  ├─ enable_intent_routing? （设备级设置开关）
  │
  ├─ 若启用意图路由：
  │   ├─ 1) ClaudeIntentRouter::route_across_providers(...)
  │   │     ├─ 仅在“首轮请求”执行跨供应商路由（会话级绑定）
  │   │     └─ 返回按优先级排序的 providers + patched_body
  │   │
  │   └─ 2) 若 1) 返回 None 或出错：
  │         └─ ClaudeIntentRouter::route(...)（单供应商内部模型路由）
  │
  ├─ forward_with_retry(... providers_for_request, allow_switch_current ...)
  └─ process_response / handle_claude_transform
```

### 3.2 会话 ID 提取与会话级绑定

模块：`src-tauri/src/proxy/session.rs`、`src-tauri/src/proxy/handler_context.rs`

1. `extract_session_id(headers, body, "claude")`：
   - 优先从 `metadata.user_id` 按约定格式（`user_xxx_session_yyy`）提取 `yyy`；
   - 否则使用 `metadata.session_id`；
   - 若均不存在，生成新的 UUID。
2. `RequestContext::new(...)` 中将 `session_id` 存入 `RequestContext`：
   - 用于日志关联（请求日志表中的 `session_id` 字段）；
   - 现在也用于意图路由的调试日志。

**会话级绑定策略**实现在 `ClaudeIntentRouter::route_across_providers` 内部：

```rust
if !Self::is_first_turn(body) {
    log::debug!(
        "[IntentRouter] Skip cross-provider routing for non-first turn (session={})",
        session_id
    );
    return Ok(None);
}
```

其中 `is_first_turn` 规则：

- 遍历 `messages` 数组：
  - 统计 `role == "user"` 的数量；
  - 检查是否存在 `role == "assistant"`。
- 当且仅当：
  - `user_count == 1` 且 `!has_assistant` 时，判定为“首轮请求”。

因此：

- 首轮请求：允许做跨供应商意图路由；
- 后续轮次：跳过跨供应商路由，沿用当前 ProviderRouter 提供的供应商链（只允许内部模型级路由）。

### 3.3 供应商级（跨供应商）意图路由

模块：`src-tauri/src/proxy/intent_router.rs::ClaudeIntentRouter::route_across_providers`

流程：

1. 从数据库读取所有 Claude 供应商：

   ```rust
   let providers_map = db.get_all_providers("claude")?;
   ```

2. 对每个供应商：
   - 通过 `get_main_model(provider)` 推断主模型：
     - 依次尝试 `ANTHROPIC_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL`、`ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_REASONING_MODEL`。
   - 组装候选项 `(Provider, main_model_id, description)`：
     - 描述优先使用 `provider.meta.intent_description`；
     - 如果为空，回退到 `provider.notes`；
     - 若仍为空则使用空字符串。

3. 将候选列表转换为 `CandidateModel`：

   ```rust
   struct CandidateModel {
       label: String,   // 例如 "Univibe-Claude: 适合复杂数学推理…"
       model_id: String // 实际 model id
   }
   ```

   - 描述会根据长度截断（>80 字符时添加 `…`）。

4. 选择“路由模型”（router_model）：
   - 从全局设置中读取 `claude_router_provider_id`；
   - 如设置了路由专用供应商，则使用其主模型；
   - 否则回退到当前 Claude 供应商；
   - 如果仍然失败，则使用候选列表中的第一个供应商。

5. 构造路由提示词：
   - 使用 `estimate_token_length` 粗略估算 token 数量并映射到 `short / medium / long`；
   - 使用 `build_summary` 提取最近一条 user 消息文本，并限制最大字符数（例如 2000）；
   - 生成类似以下 prompt：

     ```text
     You are an expert router that chooses which LLM model to use.
     Estimated user request token length: ~{approx_tokens} tokens (category: {size_hint}).

     Available models:
     1: Univibe-Claude: 适合复杂数学推理… (glm-4.7)
     2: Zhipu-GLM: 适合通用对话… (glm-4.7)
     ...

     Given the last user request below, choose the MOST appropriate model index.
     You must respond with ONLY an integer between 1 and N (no explanation):

     --- USER REQUEST START ---
     {summary}
     --- USER REQUEST END ---
     ```

6. 调用“路由模型”：
   - 使用当前 Claude provider 的 adapter：

     ```rust
     let adapter = get_adapter(&AppType::Claude);
     let base_url = adapter.extract_base_url(provider)?;
     let url = adapter.build_url(&base_url, "/v1/messages");
     ```

   - 构造非流式请求，`max_tokens` 很小（例如 16）；
   - 添加认证头、`anthropic-version` 等；
   - 发送请求并解析 `content[0].text` 中的整数索引。

7. 结果处理与回退：
   - 若解析成功且在范围内（1..=N）：
     - 根据索引取出供应商和模型；
     - 将供应商放在返回列表的第一个位置，其余供应商保持原顺序；
     - 将真实请求体中的 `model` 字段替换为选中的 `model_id`。
   - 若调用失败或返回非法值：
     - 记录警告日志；
     - 回退到基于长度的 `fallback_by_length`，在候选中选择一个模型索引；
     - 不改变供应商链，仅改变模型字段。

### 3.4 模型级（单供应商内部）意图路由

模块：`ClaudeIntentRouter::route`

主要逻辑：

1. 从单个 Provider 的 env 中收集模型列表（`collect_candidates`）：
   - 主模型：`ANTHROPIC_MODEL` → `label = "Claude Default"`；
   - 其它模型：`ANTHROPIC_DEFAULT_HAIKU_MODEL` → `"Claude Haiku"` 等；
   - 使用 `seen` 集合去重。
2. 若候选模型数量 ≤ 1，则不做路由。
3. 估算 token 长度，选取路由模型（优先默认模型、其次 Sonnet）。
4. 调用 `call_router_model`（与跨供应商版本相同，但候选列表不同）。
5. 若成功：将 `body["model"]` 修改为选中的 `model_id`；
6. 若失败：使用 `fallback_by_length` 按长度类别选择合适模型。

---

## 4. 数据结构与配置

### 4.1 AppSettings

文件：`src-tauri/src/settings.rs`

新增 / 使用字段：

- `enable_claude_intent_routing: bool`
  - 默认 `false`；
  - 控制是否启用 Claude CLI 意图路由；
  - 前端在设置页和主界面中共享该开关。
- `claude_router_provider_id: Option<String>`
  - 路由专用供应商 ID；
  - 若存在，则跨供应商路由优先使用该供应商的主模型作为“路由模型”。

### 4.2 Provider 元信息

结构：`crate::provider::Provider`

与意图路由相关的字段：

- `settings_config`（JSON）：包含 env 等；
- `meta.intent_description: Option<String>`：
  - 存放“意图路由描述”，由前端「编辑供应商」界面配置；
  - 路由时优先使用；
- `notes: Option<String>`：
  - 通用说明文本，意图描述为空时作为回退。

### 4.3 日志与统计

意图路由会在日志中记录：

- 供应商级路由：

  ```text
  [Claude] IntentRouter(供应商级) 选择模型: claude-sonnet-4-5-... -> glm-4.7 (provider=...)
  ```

- 路由调用失败：

  ```text
  [IntentRouter] 跨供应商路由模型调用失败，回退到长度启发式: ...
  ```

请求日志表 `proxy_request_logs` 中的 `session_id` 字段可用于按会话聚合分析意图路由效果。

---

## 5. 扩展与演进

### 5.1 扩展到其它应用（Codex / Gemini / OpenCode）

当前实现针对 `AppType::Claude`，后续可以：

- 在 `ClaudeIntentRouter` 思路基础上，为 Codex/Gemini/OpenCode 引入对应的 `*IntentRouter`；
- 在设置中增加跨应用的意图路由开关和路由提示词配置；
- 在代理入口处根据 `AppType` 路由到不同的意图路由器。

关键点：

- 不同应用的请求体格式不同（`/v1/responses`、`/v1beta/models/...` 等），需要各自的摘要和 token 估算逻辑；
- UI 上需提供统一入口和跨应用路由策略说明。

### 5.2 更细粒度的会话绑定

目前会话绑定仅基于 `messages` 结构推断“首轮请求”，未来可以：

- 更严格地依赖 `session_id`，并在本地维护 `session_id -> provider_id` 的缓存；
- 支持“锁定当前对话到某供应商”（前端按钮），直接覆写意图路由结果；
- 支持对话级别的“路由重试”（例如允许用户手动重新路由当前会话）。

### 5.3 更丰富的路由策略

现有路由提示词主要考虑：

- 请求长度；
- 供应商描述文本。

未来可以引入：

- 用户偏好（例如偏好某些供应商的隐私策略、地域）；
- 成本信息（结合 `model_pricing` 表，控制成本上限）；
- 历史效果反馈（根据过去会话的成功率或用户评分调整路由权重）。

---

## 6. 风险与限制

### 6.1 兼容性风险

- 某些代理厂商的 Claude 兼容接口可能不支持附加的路由提示词模式；
- 路由模型本身返回非法输出（非整数或超出范围）会触发回退逻辑。

缓解方式：

- 严格限制路由调用的 `max_tokens`；
- 增强错误日志，便于定位具体供应商问题；
- 为无法正常路由的供应商提供禁用路由选项（未来可考虑）。

### 6.2 体验风险

- 意图描述填写不当（所有供应商都写成“全能型”）会降低路由效果；
- 用户中途修改意图描述时，旧会话不会立即反映新配置，可能产生“预期不一致”。

缓解方式：

- 文档中强调：修改意图描述后，新配置只影响新会话；
- 在 UI 中提示“意图路由描述编写建议”；
- 提供调试面板展示路由决策过程（未来增强）。

---

## 7. 参考文件

- `src-tauri/src/proxy/handlers.rs` — Claude `/v1/messages` 处理入口
- `src-tauri/src/proxy/intent_router.rs` — 意图路由核心实现
- `src-tauri/src/proxy/session.rs` — 会话 ID 提取逻辑
- `src-tauri/src/settings.rs` — AppSettings 结构与持久化
- `docs/user-manual/4-proxy/4.6-intent-routing.md` — 用户手册：意图路由使用说明

