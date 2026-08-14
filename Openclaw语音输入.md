> 把以下文件内容发给 OpenClaw，让它自己检查并实现对应的功能
> **需要说明**：OpenClaw 本身其实支持接入实时语音输入，但从我调研结果看应该是不支持接入 Qwen 的对应模型（百炼的语音模型是通过 DashScope 对接）
> 所以本文档主要是针对：
> - 希望 OpenClaw 能接入 Qwen ASR 模型
> - 语音输入不直接作为实时输入内容，而是转译成文本内容一次性输入

# OpenClaw WebChat 接入 DashScope Qwen ASR 语音输入操作手册

> 目标：在 OpenClaw WebChat 中实现“语音输入 → ASR 转文字 → 填入输入框 → 用户手动发送”。  
> 非目标：不做完整语音对话、不自动发送给 agent、不启用助手语音输出/TTS。

## 1. 背景与结论

适用版本：**OpenClaw `2026.7.1`**。

> 本文记录的是 OpenClaw `2026.7.1` 下的本地热修方案。若后续 OpenClaw 版本的 Talk/WebChat 实现发生变化，应以新版本源码和实际 bundle 为准重新核对。

OpenClaw WebChat 的 Talk/麦克风入口默认容易走“Realtime Voice”路径。若没有配置 OpenAI/Gemini 等 realtime voice provider，会报错：

```text
Error: Realtime voice provider "openai" is not configured
```

本次需求只是“听写/转写”，正确路径应是 OpenClaw 的 realtime transcription provider：

```js
api.registerRealtimeTranscriptionProvider(...)
```

最终采用阿里云百炼/DashScope：

- ASR 模型：`qwen3-asr-flash-realtime`
- WebSocket：`wss://dashscope.aliyuncs.com/api-ws/v1/realtime?model=qwen3-asr-flash-realtime`
- API Key：通过 `DASHSCOPE_API_KEY` 或配置项提供，文档中不记录真实 key
- OpenClaw provider id：`dashscope-qwen-asr`

## 2. 关键区分

### 2.1 Realtime Voice Provider

完整实时语音对话，通常包括：

- 用户语音输入
- 模型实时理解
- 工具调用/agent consult
- 助手语音输出

OpenClaw 中相关路径包括：

```json5
talk.realtime.*
```

若配置错误，会触发：

```text
Realtime voice provider "openai" is not configured
Realtime voice provider "dashscope-qwen-asr" is not registered
```

### 2.2 TTS Provider

文字转语音，只负责助手语音输出。DashScope 的 `qwen3-tts-flash-realtime` 属于 TTS，不解决语音输入转文字。

### 2.3 Realtime Transcription Provider

实时语音转文字，符合本次需求。OpenClaw catalog 中应出现在：

```json5
talk.catalog.transcription.providers
```

并具备：

```json5
{
  id: "dashscope-qwen-asr",
  configured: true,
  modes: ["transcription"],
  transports: ["gateway-relay"],
  brains: ["none"]
}
```

## 3. 环境与文件位置

本次环境：

- OpenClaw Gateway：`2026.7.1`
- Gateway 运行方式：systemd user service
- OpenClaw 配置文件：`~/.openclaw/openclaw.json`
- 本地插件目录：`~/.openclaw/local-plugins/dashscope-qwen-asr`
- Control UI route bundle：`/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/index-zot7ymVq.js`
- Control UI chat bundle：`/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/chat-page-CCcBFyis.js`
- Control UI cache-bust chat bundle：`/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/chat-page-CCcBFyis-asrfix.js`
- Talk runtime bundle：`/path/to/nodejs/lib/node_modules/openclaw/dist/talk-CBDoMLuy.js`

> 注意：`dist/*.js` 是安装产物，升级 OpenClaw 后可能被覆盖。长期方案应将前端听写 fallback 做成正式补丁或插件能力。

## 4. 确认 OpenClaw 当前能力

查看插件列表：

```bash
openclaw plugins list --json
```

确认插件系统支持：

```json5
realtimeTranscriptionProviderIds
```

查看 Talk catalog：

```bash
openclaw gateway call talk.catalog
```

注意 CLI 输出前可能有标题，例如：

```text
Gateway call: talk.catalog
{ ...json... }
```

脚本解析时要从第一个 `{` 开始截取 JSON。

## 5. 创建 DashScope Qwen ASR 本地插件

创建目录：

```bash
mkdir -p ~/.openclaw/local-plugins/dashscope-qwen-asr
cd ~/.openclaw/local-plugins/dashscope-qwen-asr
```

### 5.1 `package.json`

关键点：必须包含 `openclaw.extensions`，否则安装时报错：

```text
package.json missing openclaw.extensions
```

示例：

```json
{
  "name": "openclaw-dashscope-qwen-asr-local-plugin",
  "version": "0.1.0",
  "private": true,
  "main": "index.js",
  "dependencies": {
    "ws": "^8.18.0"
  },
  "openclaw": {
    "extensions": ["./index.js"]
  }
}
```

安装依赖：

```bash
npm install --omit=dev
```

### 5.2 `openclaw.plugin.json`

关键点：当前 OpenClaw 要求 manifest 有 `configSchema`，否则安装时报错：

```text
plugin manifest requires configSchema
```

示例结构：

```json
{
  "id": "dashscope-qwen-asr",
  "name": "DashScope Qwen ASR",
  "version": "0.1.0",
  "description": "Realtime transcription provider for DashScope Qwen ASR Flash.",
  "enabledByDefault": true,
  "activation": {
    "onStartup": true
  },
  "setup": {
    "providers": {
      "envVars": ["DASHSCOPE_API_KEY"]
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": true
  }
}
```

### 5.3 `index.js` 插件实现要点

实现需注册 realtime transcription provider：

```js
api.registerRealtimeTranscriptionProvider({
  id: "dashscope-qwen-asr",
  label: "DashScope Qwen ASR Flash Realtime",
  defaultModel: "qwen3-asr-flash-realtime",
  models: ["qwen3-asr-flash-realtime"],
  isConfigured: ({ providerConfig }) => Boolean(getApiKey(providerConfig || {})),
  createSession: (req) => new DashScopeQwenAsrSession(req, req.providerConfig || {}),
});
```

连接 DashScope：

```js
const url = "wss://dashscope.aliyuncs.com/api-ws/v1/realtime?model=qwen3-asr-flash-realtime";
```

请求头：

```js
{
  Authorization: "Bearer <DASHSCOPE_API_KEY>",
  "OpenAI-Beta": "realtime=v1"
}
```

初始化事件：

```json
{
  "type": "session.update",
  "session": {
    "modalities": ["text"],
    "input_audio_format": "pcm",
    "sample_rate": 16000,
    "input_audio_transcription": {
      "language": "zh"
    },
    "turn_detection": {
      "type": "server_vad",
      "threshold": 0.2,
      "silence_duration_ms": 800
    }
  }
}
```

音频事件：

```json
{
  "type": "input_audio_buffer.append",
  "audio": "<base64-pcm16-audio>"
}
```

回调映射：

- partial/delta/in_progress → `req.onPartial(text)`
- completed/done/final → `req.onTranscript(text)`
- speech_started → `req.onSpeechStart()`
- error → `req.onError(error)`

## 6. 安装本地插件

不要手写 `plugins.entries.dashscope-qwen-asr` 到 `openclaw.json`。手写可能导致：

```text
plugins.entries.dashscope-qwen-asr: Invalid input
```

使用正式安装机制：

```bash
openclaw plugins install --link ~/.openclaw/local-plugins/dashscope-qwen-asr
```

不要使用：

```bash
openclaw plugins install --link --force ...
```

因为会报：

```text
--force is not supported with --link
```

安装成功后会提示：

```text
Linked plugin path: ~/.openclaw/local-plugins/dashscope-qwen-asr
Restart the gateway to load plugins.
```

重启 Gateway：

```bash
openclaw gateway restart
```

验证配置：

```bash
openclaw config validate
openclaw plugins list --json
```

预期看到：

```json
"realtimeTranscriptionProviderIds": ["dashscope-qwen-asr"]
```

## 7. 配置 DashScope ASR Provider

当前 OpenClaw Gateway transcription relay 会从 Voice Call streaming 兼容路径读取配置：

```json5
plugins.entries.voice-call.config.streaming.provider
plugins.entries.voice-call.config.streaming.providers.<providerId>
```

本次配置形态：

```json5
{
  "plugins": {
    "entries": {
      "voice-call": {
        "config": {
          "streaming": {
            "provider": "dashscope-qwen-asr",
            "providers": {
              "dashscope-qwen-asr": {
                "apiKey": "<REDACTED_OR_USE_ENV:DASHSCOPE_API_KEY>",
                "model": "qwen3-asr-flash-realtime",
                "baseUrl": "wss://dashscope.aliyuncs.com/api-ws/v1/realtime",
                "language": "zh",
                "encoding": "pcm16",
                "sampleRate": 16000,
                "dashscopeSampleRate": 16000,
                "serverVad": true,
                "vadThreshold": 0.2,
                "silenceDurationMs": 800
              }
            }
          }
        }
      }
    }
  }
}
```

更推荐不要在文件中保存明文 key，而是通过环境变量：

```bash
export DASHSCOPE_API_KEY='<your-key>'
```

若 Gateway 由 systemd user service 管理，应把环境变量放到该服务能读取的位置，或使用 OpenClaw 支持的安全配置方式。不要在文档、聊天或仓库中记录真实 key。

配置校验：

```bash
openclaw config validate
```

可能出现 warning：

```text
plugins.entries.voice-call: plugin not installed: voice-call
```

本次验证中该 warning 不阻止 transcription catalog 工作。

## 8. 必须清理错误的 realtime voice 配置

不要把 `dashscope-qwen-asr` 放入：

```json5
talk.realtime.providers.dashscope-qwen-asr
```

否则 WebChat/Talk 可能把 ASR provider 当成 realtime voice provider，触发：

```text
Error: Realtime voice provider "dashscope-qwen-asr" is not registered
```

也不要配置：

```json5
talk.realtime.mode = "transcription"
```

因为：

```text
talk.client.create only supports mode="realtime"; use talk.catalog for transcription provider discovery
```

正确结果：

- `talk.catalog.transcription.providers` 中有 `dashscope-qwen-asr` 且 `configured: true`
- `talk.catalog.realtime.providers` 中不应出现 configured 的 `dashscope-qwen-asr`
- `configuredRealtime` 应为空或只包含真正的 realtime voice provider

验证命令：

```bash
openclaw gateway call talk.catalog
```

## 9. Gateway relay 音频格式处理

OpenClaw `talk.session.create(mode="transcription")` 创建 Gateway relay session。相关 runtime 逻辑位于：

```text
/path/to/nodejs/lib/node_modules/openclaw/dist/talk-CBDoMLuy.js
```

原始 relay 常量可能是：

```js
const RELAY_INPUT_ENCODING = "g711_ulaw";
const RELAY_INPUT_SAMPLE_RATE_HZ = 8e3;
```

若 provider config 声明 `pcm16/16000`，会被校验拦截：

```text
Gateway transcription relay requires g711_ulaw/8000 audio
```

在 OpenClaw `2026.7.1` 的 WebChat fallback 中，也可能因为前端仍误走 realtime relay 而出现：

```text
gateway-relay realtime talk currently requires PCM16 audio
```

这类错误的根因不是 DashScope ASR 本身，而是 WebChat/Talk 仍进入了 realtime voice path 或旧前端 bundle。

本次为了 WebChat 浏览器录音更自然，改成：

```js
const RELAY_INPUT_ENCODING = "pcm16";
const RELAY_INPUT_SAMPLE_RATE_HZ = 16e3;
```

并让前端 AudioContext 以 16k 采样，发送 PCM16 base64。这样插件可直接转发给 DashScope。

> 备选方案：保持 relay 为 `g711_ulaw/8000`，在插件中把 µ-law/8k 转码成 PCM16/16k 再发给 DashScope。但本次最终采用 relay PCM16/16k。

修改安装产物后需重启：

```bash
openclaw gateway restart
```

## 10. WebChat 前端听写 fallback

问题：Control UI/WebChat 的 `toggleRealtimeTalk()` 原本只启动 realtime voice session：

- 先尝试 `talk.client.create`
- fallback 到 `talk.session.create`，但仍是：

```json
{
  "mode": "realtime",
  "transport": "gateway-relay",
  "brain": "agent-consult"
}
```

这不符合“只转写、不自动发送”的需求。

本次修改 Control UI bundle：

```text
/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/chat-page-CCcBFyis.js
```

新增本地 dictation session：

- 查询 `talk.catalog`
- 若没有 configured realtime voice provider，但有 configured transcription provider，则优先走 dictation path
- 不再先尝试 `mode: "realtime"` + `transport: "gateway-relay"`，避免触发 realtime relay 的 PCM16 guard
- 调用：

```js
talk.session.create({
  sessionKey,
  mode: "transcription",
  transport: "gateway-relay",
  brain: "none",
  provider: "dashscope-qwen-asr"
})
```

- 浏览器通过 `getUserMedia` 获取麦克风
- 使用 `AudioContext({ sampleRate: 16000 })`
- 将 Float32 PCM 转 PCM16 base64
- 调用：

```js
talk.session.appendAudio({
  sessionId,
  audioBase64,
  timestamp
})
```

- 监听 `talk.event`
- `partial` 只显示听写状态
- `transcript final` 追加到 `chatMessage`
- 不自动发送消息，仍由用户手动点击发送

### 10.1 忽略旧 realtime provider/openai

OpenClaw `2026.7.1` 升级后，浏览器本地状态或旧前端逻辑可能继续携带旧的 realtime provider，例如：

```json
{
  "provider": "openai"
}
```

如果 transcription fallback 继续继承这个 provider，会导致服务端仍按 OpenAI realtime voice provider 解析，并再次报：

```text
Realtime voice provider "openai" is not configured
```

因此本次最终热修要求：

- 前端创建 transcription session 时只传 `sessionKey/mode/transport/brain/provider`，不要继承旧 `this.options`
- 服务端 `talk.client.create` fallback 到 transcription 时忽略 `typedParams.provider`，改为从 configured transcription provider 自动选择 `dashscope-qwen-asr`
- 即使请求中仍带 `provider: "openai"`，也应返回 `mode: "transcription"`、`provider: "dashscope-qwen-asr"`

服务端对应 bundle：

```text
/path/to/nodejs/lib/node_modules/openclaw/dist/talk-CBDoMLuy.js
```

核心效果是把 transcription fallback 的配置解析从“使用请求里的 provider”改为“自动选择已配置的 transcription provider”。

### 10.2 前端缓存破坏

静态资源虽然返回 `Cache-Control: no-cache`，但实际测试中浏览器仍可能保留旧模块或旧标签页状态，表现为后端已创建 transcription session，前端仍报 PCM16/realtime 相关错误。

本次最终处理：

- 复制热修后的 chat bundle 为：`chat-page-CCcBFyis-asrfix.js`
- 修改 route bundle：`index-zot7ymVq.js`
- 将 chat route 的动态 import 从 `chat-page-CCcBFyis.js` 改为 `chat-page-CCcBFyis-asrfix.js`

这样浏览器会加载新文件名，避免继续复用旧 `chat-page-CCcBFyis.js` 模块。

## 11. 浏览器麦克风安全上下文

即使 Chrome 设置里允许麦克风，如果页面不是安全上下文，浏览器仍会隐藏：

```js
navigator.mediaDevices
```

典型报错：

```text
Dictation requires browser microphone access
Cannot read properties of undefined (reading 'getUserMedia')
Browser microphone API is unavailable...
```

判断方法，在 DevTools Console 执行：

```js
window.isSecureContext
navigator.mediaDevices
```

安全上下文通常包括：

- `https://...`
- `http://localhost:...`
- `http://127.0.0.1:...`

不安全：

- `http://服务器IP:18080`
- 没有 HTTPS 的内网域名
- 普通 HTTP 反代

### 11.1 快速方案：SSH Tunnel

远端服务器部署 OpenClaw，且暂时没有 HTTPS 证书时，最快方式：

```bash
ssh -L 18789:127.0.0.1:18789 <user>@<server-ip>
```

然后本机浏览器打开：

```text
http://127.0.0.1:18789/
```

对 Chrome 来说这是 localhost，麦克风 API 可用。

### 11.2 长期方案：域名 + HTTPS 反代

使用 Caddy 最简单：

```bash
sudo apt install -y caddy
sudo tee /etc/caddy/Caddyfile >/dev/null <<'EOF'
claw.example.com {
    reverse_proxy 127.0.0.1:18789
}
EOF
sudo systemctl reload caddy
```

Caddy 会自动申请和续期 Let’s Encrypt 证书，并正确代理 WebSocket。

### 11.3 Nginx HTTP 反代不足以启用麦克风

示例 HTTP 反代：

```nginx
server {
    listen 18080;
    server_name <server-ip>;

    location / {
        proxy_pass http://127.0.0.1:18789;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 300s;
        proxy_buffering off;
    }
}
```

这个配置可以代理 WebSocket，但仍是：

```text
http://<server-ip>:18080
```

所以 Chrome 不会开放麦克风。必须改为 HTTPS，或通过 SSH tunnel 使用 localhost。

### 11.4 Chrome 临时绕过：把指定 HTTP Origin 当作安全来源

本次实际验证中，也可以通过 Chrome flags 临时解决远端 HTTP 页面无法启用麦克风的问题。

打开：

```text
chrome://flags/#unsafely-treat-insecure-origin-as-secure
```

启用该项，并在输入框中添加当前 WebChat 的 HTTP origin，例如：

```text
http://<server-ip>:18080
```

重启 Chrome 后，该 origin 会被 Chrome 当作 secure context，`navigator.mediaDevices.getUserMedia` 可用，WebChat Talk/听写可以申请麦克风权限。

注意：这是浏览器本机开发/测试绕过方式，不等同于真正 HTTPS，不建议作为多人共享或长期公网部署方案。长期仍建议使用 HTTPS 反代、SSH tunnel、Cloudflare Tunnel 或 Tailscale Funnel。

## 12. 常见错误与处理

### 12.1 `Realtime voice provider "openai" is not configured`

原因：WebChat 走了 realtime voice path，默认选 OpenAI。

处理：

- 确认需求是否只是 transcription
- 配置 transcription provider
- 修改 WebChat 前端 fallback，避免没有 realtime voice provider 时继续走 OpenAI
- 在 OpenClaw `2026.7.1` 中，服务端 `talk.client.create` 的 transcription fallback 也应忽略旧 `provider: "openai"`，自动选择 configured transcription provider

### 12.1.1 `gateway-relay realtime talk currently requires PCM16 audio`

原因：WebChat fallback 仍先进入了 realtime gateway-relay path，而不是直接创建 transcription session。

处理：

- 前端 fallback 顺序必须是先查 `talk.catalog`
- 若无 configured realtime voice provider 且有 configured transcription provider，直接调用 `talk.session.create(mode: "transcription")`
- 不要先调用 `talk.session.create(mode: "realtime", transport: "gateway-relay")`
- 若服务端和前端均已热修但浏览器仍报旧错，优先怀疑旧标签页/缓存/旧 chunk，使用 cache-bust 文件名或新标签页重新打开 WebChat

### 12.2 `talk.client.create only supports mode="realtime"`

原因：错误地把 transcription mode 传给 `talk.client.create`。

处理：

- transcription provider 用 `talk.catalog.transcription` 发现
- transcription session 用 `talk.session.create`

### 12.3 `Realtime voice provider "dashscope-qwen-asr" is not registered`

原因：ASR provider 被放进 `talk.realtime.providers`，被误当成 realtime voice provider。

处理：

- 删除 `talk.realtime.providers.dashscope-qwen-asr`
- 删除错误的 `talk.realtime.mode/provider/transport/brain`
- 保留 `plugins.entries.voice-call.config.streaming.*`

### 12.4 `plugin manifest requires configSchema`

原因：插件 manifest 缺少 schema。

处理：在 `openclaw.plugin.json` 增加 `configSchema`。

### 12.5 `package.json missing openclaw.extensions`

原因：插件 package 未声明 extension entry。

处理：在 `package.json` 增加：

```json
"openclaw": {
  "extensions": ["./index.js"]
}
```

### 12.6 `--force is not supported with --link`

原因：`openclaw plugins install --link --force` 不支持。

处理：使用：

```bash
openclaw plugins install --link <path>
```

### 12.7 `Browser microphone API is unavailable`

原因：页面不是浏览器安全上下文。

处理：

- 使用 `http://127.0.0.1:18789/`
- 或 SSH tunnel
- 或 HTTPS 反代
- 或在个人测试环境中使用 `chrome://flags/#unsafely-treat-insecure-origin-as-secure`，把当前 HTTP WebChat origin 临时加入安全来源

## 13. 验证清单

### 13.1 配置校验

```bash
openclaw config validate
```

预期：

```text
Config valid: ~/.openclaw/openclaw.json
```

允许存在 `voice-call` plugin warning，只要 catalog 正常。

### 13.2 插件注册

```bash
openclaw plugins list --json
```

预期插件能力包含：

```json
"realtimeTranscriptionProviderIds": ["dashscope-qwen-asr"]
```

### 13.3 Catalog

```bash
openclaw gateway call talk.catalog
```

预期：

```json
{
  "transcription": {
    "providers": [
      {
        "id": "dashscope-qwen-asr",
        "configured": true
      }
    ]
  },
  "realtime": {
    "providers": []
  }
}
```

实际 `realtime.providers` 可以不为空，但不应有误配置的 `dashscope-qwen-asr` realtime voice provider。

### 13.4 Gateway 状态

```bash
openclaw gateway status
```

预期：

```text
Runtime: running
Connectivity probe: ok
```

### 13.4.1 Stale provider 探针

OpenClaw `2026.7.1` 热修后，即使请求还带旧 provider，也应兜底到 DashScope transcription：

```bash
openclaw gateway call talk.client.create --params '{"sessionKey":"agent:main:dashboard:asr-probe","provider":"openai"}' --json
```

预期关键字段：

```json
{
  "mode": "transcription",
  "transport": "gateway-relay",
  "brain": "none",
  "provider": "dashscope-qwen-asr",
  "audio": {
    "inputEncoding": "pcm16",
    "inputSampleRateHz": 16000
  }
}
```

### 13.4.2 前端资源探针

确认 route bundle 已引用 cache-bust chat bundle：

```bash
rg -n -F './chat-page-CCcBFyis-asrfix.js' /path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/index-zot7ymVq.js
```

确认浏览器可请求新 chat bundle：

```bash
curl -sI http://127.0.0.1:18789/assets/chat-page-CCcBFyis-asrfix.js
```

预期：HTTP `200 OK`，`Content-Type: application/javascript`。

### 13.5 浏览器

在 WebChat 页面 Console：

```js
window.isSecureContext
navigator.mediaDevices
```

预期：

```js
true
MediaDevices {...}
```

### 13.6 功能

点击 WebChat Talk/麦克风按钮：

- 浏览器弹出麦克风授权
- 状态进入 listening/dictation ready
- 说话后 partial/final transcript 出现
- final transcript 追加到输入框
- 不自动发送，需要用户手动发送

## 14. 安全注意事项

1. 不要在文档、仓库、聊天记录中保存真实 DashScope API key。
2. 如果 API key 曾经出现在聊天或日志中，建议在 DashScope 控制台轮换 key。
3. 优先使用环境变量 `DASHSCOPE_API_KEY` 或 OpenClaw 支持的 secret/config 机制。
4. OpenClaw Gateway 建议继续只绑定 `127.0.0.1`，公网访问通过 HTTPS 反代或 SSH tunnel。
5. 直接修改 OpenClaw `dist` 文件属于本地热修，升级后需要复核或重新应用。

## 15. 本次最终状态

- 本地插件已安装并加载：`dashscope-qwen-asr`
- `talk.catalog.transcription.providers` 中 `dashscope-qwen-asr` 为 `configured: true`
- `talk.catalog.realtime.providers` 中无误配置的 `dashscope-qwen-asr`
- Gateway 状态正常
- `talk.client.create` 即使带旧 `provider: "openai"`，也会返回 `mode: "transcription"`、`provider: "dashscope-qwen-asr"`、`pcm16/16000`
- 浏览器在安全上下文中已能正常使用 WebChat Talk/麦克风音频输入
- 真实音频 → DashScope ASR → 文本填入的端到端链路已由用户验证可用
- 最后一轮 residual 错误判断为浏览器缓存/旧 chunk 导致，已通过新文件名 `chat-page-CCcBFyis-asrfix.js` 避免复用旧模块

## 16. 后续建议

1. 将前端 dictation fallback、服务端 transcription fallback、relay PCM16/16k 修改整理成正式补丁，避免 OpenClaw 升级覆盖。
2. 记录更多 DashScope 返回事件类型，必要时扩展 `extractTranscript()` 的字段兼容。
3. 若升级 OpenClaw，重新核对 dist bundle 文件名、Talk API 行为和 `talk.catalog` 输出。
4. 若长期公网使用，配置域名 + HTTPS 反代；临时使用优先 SSH tunnel。
