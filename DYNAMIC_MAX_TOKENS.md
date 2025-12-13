# 动态 max_tokens 和 Context 自动裁剪功能指南

## 功能说明

此功能解决了使用上下文长度受限的 LLM 模型（如 QWen 3.3 coder 64K）时的两个核心问题：

1. **max_tokens 溢出错误**：因输入 tokens 过多而导致的 `max_tokens` 溢出
2. **上下文过长问题**：对话历史累积导致超出模型最大上下文限制

### 问题示例

当你的模型最大上下文是 65536 tokens，但输入已经使用了 59062 tokens，系统仍然请求 8192 tokens 的输出时：

```
Error: 'max_tokens' or 'max_completion_tokens' is too large: 8192.
This model's maximum context length is 65536 tokens and your request has
59062 input tokens (8192 > 65536 - 59062).
```

## 配置方法

### 1. Provider 配置

为每个 provider 添加 `max_context_tokens` 字段，指定该模型的最大上下文长度：

```json
{
  "Providers": [
    {
      "name": "qwen3-64k",
      "api_base_url": "http://qwen.finetuning/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": ["qwen3-coder"],
      "max_context_tokens": 65536
    },
    {
      "name": "qwen3-32k",
      "api_base_url": "http://qwen.finetuning/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": ["qwen3-coder"],
      "max_context_tokens": 32768
    }
  ]
}
```

### 2. Completion 配置

在配置文件中添加 `Completion` 部分来自定义行为：

```json
{
  "Completion": {
    "auto_limit": true,
    "auto_trim": true,
    "safety_margin_tokens": 1024,
    "max_completion_tokens_cap": 8192
  }
}
```

#### 配置项说明

- **`auto_limit`** (boolean, 默认: `true`)
  - 是否启用自动 max_tokens 限制
  - 设为 `false` 可以禁用此功能

- **`auto_trim`** (boolean, 默认: `true`) 🆕
  - **新功能**：是否启用自动消息裁剪
  - 当输入 tokens 超过预算时，自动删除旧消息
  - 始终保留最新的用户消息和系统提示
  - 从最旧的消息开始删除，直到满足 token 预算

- **`safety_margin_tokens`** (number, 默认: `1024`)
  - 安全边距 tokens 数量
  - 从最大上下文中预留的 tokens，用于防止边界错误

- **`max_completion_tokens_cap`** (number, 默认: `8192`)
  - 单次请求允许的最大 completion tokens
  - 即使计算出的可用 tokens 更多，也不会超过这个上限

### 3. 完整配置示例

```json
{
  "LOG": true,
  "LOG_LEVEL": "debug",
  "APIKEY": "your-api-key",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "API_TIMEOUT_MS": "600000",

  "Completion": {
    "auto_limit": true,
    "auto_trim": true,
    "safety_margin_tokens": 1024,
    "max_completion_tokens_cap": 8192
  },

  "Providers": [
    {
      "name": "qwen3-64k",
      "api_base_url": "http://qwen.finetuning/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": ["qwen3-coder"],
      "max_context_tokens": 65536
    }
  ],

  "Router": {
    "default": "qwen3-64k,qwen3-coder"
  }
}
```

## 工作原理

### 1. Token 计算
系统在路由阶段计算输入的 token 数量（包括 messages、system、tools）

### 2. Context 自动裁剪 🆕

**当输入 tokens 超过预算时：**

```
预算 = max_context_tokens - max_completion_tokens_cap - safety_margin_tokens
```

**裁剪策略：**
1. 从最旧的消息开始删除
2. 保留最后一条用户消息（必需）
3. 保留 system prompt 和 tools（不计入裁剪）
4. 持续删除直到满足 token 预算
5. 记录详细的裁剪日志

**示例日志：**
```
[INFO] Input tokens (59062) exceed budget (56320). Auto-trimming messages...
[INFO] Trimmed 15 messages (59062 -> 45120 tokens)
```

### 3. 动态 max_tokens 调整

裁剪后（或无需裁剪时），系统自动计算并设置 `max_tokens`：

```
available_tokens = max_context_tokens - input_tokens - safety_margin_tokens
max_tokens = min(available_tokens, max_completion_tokens_cap)
```

**示例：**

假设配置如下：
- `max_context_tokens`: 65536
- `max_completion_tokens_cap`: 8192
- `safety_margin_tokens`: 1024

**场景 1**：输入 tokens = 59062
```
预算 = 65536 - 8192 - 1024 = 56320
59062 > 56320 → 触发裁剪
裁剪后输入 = 45120
available = 65536 - 45120 - 1024 = 19392
max_tokens = min(19392, 8192) = 8192 ✅
```

**场景 2**：输入 tokens = 51610（裁剪后）
```
available = 65536 - 51610 - 1024 = 12902
max_tokens = min(12902, 8192) = 8192 ✅
```

**场景 3**：输入 tokens = 10000
```
available = 65536 - 10000 - 1024 = 54512
max_tokens = min(54512, 8192) = 8192 ✅（受 cap 限制）
```

### 4. 智能限制

- **最小值**：不会低于 100 tokens（保证有基本响应）
- **最大值**：不会超过配置的 `max_completion_tokens_cap`
- **强制设置**：每次请求都会强制设置 max_tokens，确保生效
- **详细日志**：记录每次调整和裁剪的详细信息

## 测试和验证

### 1. 启动服务

```bash
ccr stop    # 停止旧服务
ccr start   # 启动新服务
```

### 2. 查看日志

```bash
# 实时查看日志
tail -f ~/.claude-code-router/logs/*.log

# 或者使用 grep 过滤相关日志
tail -f ~/.claude-code-router/logs/*.log | grep -E "auto-trim|Set max_tokens|Trimmed"
```

### 3. 验证功能

**查找裁剪日志：**
```
[INFO] Input tokens (59062) exceed budget (56320). Auto-trimming messages...
[INFO] Trimmed 15 messages (59062 -> 45120 tokens)
```

**查找 max_tokens 设置日志：**
```
[INFO] Set max_tokens to 8192 (input: 45120, max_context: 65536, available: 19392, safety_margin: 1024, cap: 8192)
```

如果看到这些日志，说明功能已经正常工作！

## 故障排除

### 问题 1：仍然出现 max_tokens 错误

**检查项：**
1. 确认 provider 配置了 `max_context_tokens`
2. 确认 `Completion.auto_limit` 不是 `false`
3. 确认 `Completion.auto_trim` 不是 `false`
4. 检查日志中是否有相关的错误信息
5. 确保服务已重启（`ccr stop && ccr start`）

### 问题 2：对话上下文丢失太快

**解决方法：**
- 增大 `max_completion_tokens_cap` 值（允许更多输出）
- 减小 `safety_margin_tokens` 值（减少安全边距）
- 使用更大上下文的模型（如切换到 128K 模型）

### 问题 3：响应被截断

**解决方法：**
- 增大 `max_completion_tokens_cap` 值
- 减小 `safety_margin_tokens` 值
- 考虑使用更大上下文的 provider

### 问题 4：警告提示上下文接近满载

**建议：**
- 在 Router 配置中设置 `longContext` 模型
- 调整 `longContextThreshold` 阈值
- 检查是否有特别大的消息需要优化

## 优势和限制

### 优势 ✅

1. **自动化管理**：无需手动调整参数，系统自动优化
2. **防止错误**：彻底解决 max_tokens 溢出问题
3. **上下文优化**：自动裁剪旧消息，保持对话流畅
4. **灵活配置**：可以根据需求调整各项参数
5. **详细日志**：方便调试和监控
6. **智能保留**：始终保留最新的消息和重要信息

### 限制 ⚠️

1. **Token 估算**：使用 tiktoken 估算，可能与实际略有差异
2. **简单裁剪**：当前只删除旧消息，未来可能支持更智能的摘要
3. **上下文损失**：裁剪会丢失部分对话历史
4. **计算开销**：token 计算和裁剪会增加轻微延迟

## 未来改进

考虑中的功能：

1. **智能摘要**：对旧消息进行摘要而不是删除
2. **重要性评分**：保留重要消息，删除不重要的
3. **分段对话**：自动将长对话分段处理
4. **缓存优化**：利用模型的缓存机制减少重复计算

## 技术细节

### 实现位置

- **文件**：`src/utils/router.ts`
- **函数**：
  - `trimMessagesToFit()`：消息裁剪逻辑
  - `router()`：主路由函数
  - `calculateTokenCount()`：Token 计算
- **时机**：在确定使用的 model 之后，发送请求之前

### Token 计算

使用 `tiktoken` 库（cl100k_base 编码）来估算 token 数量。这是一个估算值，可能与实际 LLM 的 tokenizer 略有差异，因此需要 `safety_margin_tokens`。

### 裁剪算法

```typescript
function trimMessagesToFit(messages, maxTokens, system, tools) {
  while (currentTokens > maxTokens && messages.length > 2) {
    messages.shift(); // 删除最旧的消息
    recalculate(); // 重新计算 tokens
  }
  return messages;
}
```

## 总结

现在你可以：

1. ✅ **安全使用 QWen 64K**：自动管理 max_tokens，防止溢出
2. ✅ **自动裁剪上下文**：对话再长也不会超出限制
3. ✅ **最大化 QWen 使用**：优先使用 QWen，避免切换到 Claude
4. ✅ **详细监控**：通过日志了解每次调整的细节

只需：
1. ✅ 为每个 provider 添加 `max_context_tokens`
2. ✅ 配置 `Completion` 部分（启用 `auto_trim`）
3. ✅ 重启服务

就可以享受无忧的 LLM 体验了！🚀

## 功能说明

此功能解决了使用上下文长度受限的 LLM 模型（如 QWen 3.3 coder 64K）时，因输入 tokens 过多而导致的 `max_tokens` 溢出错误。

### 问题示例

当你的模型最大上下文是 65536 tokens，但输入已经使用了 51610 tokens，系统仍然请求 21333 tokens 的输出时：

```
Error: 'max_tokens' or 'max_completion_tokens' is too large: 21333.
This model's maximum context length is 65536 tokens and your request has
51610 input tokens (21333 > 65536 - 51610).
```

## 配置方法

### 1. Provider 配置

为每个 provider 添加 `max_context_tokens` 字段，指定该模型的最大上下文长度：

```json
{
  "Providers": [
    {
      "name": "qwen3-64k",
      "api_base_url": "http://qwen.finetuning/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": ["qwen3-coder"],
      "max_context_tokens": 65536
    },
    {
      "name": "qwen3-32k",
      "api_base_url": "http://qwen.finetuning/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": ["qwen3-coder"],
      "max_context_tokens": 32768
    },
    {
      "name": "claude-sonnet",
      "api_base_url": "https://api.anthropic.com",
      "api_key": "sk-xxx",
      "models": ["claude-3-5-sonnet"],
      "max_context_tokens": 200000
    }
  ]
}
```

### 2. Completion 配置（可选）

在配置文件中添加 `Completion` 部分来自定义行为：

```json
{
  "Completion": {
    "auto_limit": true,
    "safety_margin_tokens": 1024,
    "max_completion_tokens_cap": 8192
  }
}
```

#### 配置项说明

- **`auto_limit`** (boolean, 默认: `true`)
  - 是否启用自动 max_tokens 限制
  - 设为 `false` 可以禁用此功能

- **`safety_margin_tokens`** (number, 默认: `1024`)
  - 安全边距 tokens 数量
  - 从最大上下文中预留的 tokens，用于防止边界错误
  - 计算公式：`max_tokens = max_context_tokens - input_tokens - safety_margin_tokens`

- **`max_completion_tokens_cap`** (number, 默认: `8192`)
  - 单次请求允许的最大 completion tokens
  - 即使计算出的可用 tokens 更多，也不会超过这个上限
  - 有助于控制单次请求的响应长度和成本

### 3. 完整配置示例

```json
{
  "LOG": true,
  "LOG_LEVEL": "debug",
  "APIKEY": "your-api-key",
  "HOST": "127.0.0.1",
  "PORT": 3456,
  "API_TIMEOUT_MS": "600000",

  "Completion": {
    "auto_limit": true,
    "safety_margin_tokens": 1024,
    "max_completion_tokens_cap": 8192
  },

  "Providers": [
    {
      "name": "qwen3-64k",
      "api_base_url": "http://qwen.finetuning/v1/chat/completions",
      "api_key": "sk-xxx",
      "models": ["qwen3-coder"],
      "max_context_tokens": 65536
    }
  ],

  "Router": {
    "default": "qwen3-64k,qwen3-coder"
  }
}
```

## 工作原理

1. **Token 计算**
   - 系统在路由阶段计算输入的 token 数量（包括 messages、system、tools）

2. **动态调整**
   - 如果 provider 配置了 `max_context_tokens`
   - 系统会自动计算可用的 completion tokens：
     ```
     available_tokens = max_context_tokens - input_tokens - safety_margin_tokens
     calculated_max_tokens = min(available_tokens, max_completion_tokens_cap)
     ```

3. **智能限制**
   - 确保 `max_tokens` 不会小于 100（最小值）
   - 只在计算值小于当前 `max_tokens` 时才调整（不会增大）
   - 当输入 tokens 超过上下文 90% 时会记录警告日志

4. **日志输出**
   - 系统会记录每次调整的详细信息：
     ```
     Auto-adjusted max_tokens to 13402
     (input: 51610, max_context: 65536, safety_margin: 1024, cap: 8192)
     ```

## 迁移现有配置

如果你之前尝试过这些配置但没有生效：

### 移除无效配置

```json
// ❌ 这些配置在当前版本中不起作用，可以删除
{
  "Providers": [{
    "transformer": {
      "use": [
        ["maxtoken", {
          "max_tokens": "auto",       // ❌ 不支持
          "model_max_len": 65536,    // ❌ 不支持
          "reserve_input_ratio": 0.95 // ❌ 不支持
        }]
      ]
    }
  }],
  "Router": {
    "strategy": "token_count",  // ❌ 不支持
    "routes": [...]             // ❌ 不支持
  },
  "History": {...}              // ❌ 不支持
}
```

### 使用新配置

```json
// ✅ 使用新的配置方式
{
  "Completion": {
    "auto_limit": true,
    "safety_margin_tokens": 1024,
    "max_completion_tokens_cap": 8192
  },
  "Providers": [{
    "name": "qwen3-64k",
    "max_context_tokens": 65536,  // ✅ 添加这个字段
    // 其他配置保持不变
  }]
}
```

## 测试和验证

### 1. 启动服务

```bash
ccr stop    # 停止旧服务
ccr start   # 启动新服务
```

### 2. 查看日志

```bash
tail -f ~/.claude-code-router/logs/*.log
```

### 3. 测试请求

在日志中查找类似的输出：

```
[INFO] Auto-adjusted max_tokens to 13402 (input: 51610, max_context: 65536, safety_margin: 1024, cap: 8192)
```

如果看到这条日志，说明功能已经正常工作。

## 故障排除

### 问题：仍然出现 max_tokens 错误

**检查项：**
1. 确认 provider 配置了 `max_context_tokens`
2. 确认 `Completion.auto_limit` 不是 `false`
3. 检查日志中是否有相关的错误信息
4. 确保服务已重启（`ccr stop && ccr start`）

### 问题：响应被截断

**解决方法：**
- 增大 `max_completion_tokens_cap` 值
- 减小 `safety_margin_tokens` 值
- 考虑使用更大上下文的模型

### 问题：警告提示上下文接近满载

**建议：**
- 在 Router 配置中设置 `longContext` 模型
- 调整 `longContextThreshold` 阈值
- 使用更大上下文的 provider

## 技术细节

### 实现位置

- **文件**：`src/utils/router.ts` 的 `router()` 函数
- **时机**：在确定使用的 model 之后，发送请求之前
- **依赖**：使用 `calculateTokenCount()` 计算输入 tokens

### Token 计算

使用 `tiktoken` 库（cl100k_base 编码）来估算 token 数量。这是一个估算值，可能与实际 LLM 的 tokenizer 略有差异，因此需要 `safety_margin_tokens`。

## 总结

这个功能会自动为你管理 `max_tokens`，避免超出模型的上下文限制。只需：

1. ✅ 为每个 provider 添加 `max_context_tokens`
2. ✅ （可选）配置 `Completion` 部分自定义行为
3. ✅ 重启服务

就可以安全地使用上下文受限的模型了！
