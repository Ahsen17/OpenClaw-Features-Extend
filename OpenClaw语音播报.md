> 把以下文件内容发给 OpenClaw，让它自己检查并实现对应的功能
> 
> **需要说明**：本文档主要针对 OpenClaw 使用 Chrome 自带的语音输出，如使用其他浏览器可让 OpenClaw 自行探索调整

# OpenClaw Control UI 接入 Chrome TTS 自动语音输出操作手册

> 目标：在 OpenClaw Dashboard/Control UI 的聊天窗口中，实现“助手文本回复正常显示 + Chrome 自动实时朗读”。
> 非目标：不生成可点击音频附件、不做完整实时语音对话、不改变模型输出链路。

## 1. 背景与结论

适用版本：**OpenClaw `2026.7.1 (2d2ddc4)`**。

> 本文记录的是 OpenClaw `2026.7.1` 下的本地热修方案。改动位于 `dist/control-ui` 安装产物中，升级或重装 OpenClaw 后可能被覆盖，应以新版本源码、实际 bundle 和浏览器行为重新核对。

用户需求：

- 助手回复仍然以文本形式显示在 WebChat/Dashboard 中
- 语音自动播放，不需要点击音频附件
- 尽量实时朗读，生成一句读一句，不等整段回复完成
- 只朗读助手最终可见回复，不朗读工具输出、调试日志或中间过程
- 中文语速接近自然交流，本次最终使用 `rate: 1.3`

最终结论：OpenClaw 服务端 `messages.tts` 更适合“回复结束后生成音频文件/附件”的批处理 TTS，不适合本需求的自动播放和流式朗读。要实现 Chrome 中自动、实时、保留文本的朗读，最直接可行的方案是浏览器端 Web Speech API：

```js
window.speechSynthesis
SpeechSynthesisUtterance
```

最终采用方案：

- 服务端自动 TTS 关闭：`messages.tts.auto = "off"`
- 保留 Microsoft TTS provider，用于手动 `/tts audio`
- 在 Control UI 注入本地脚本 `openclaw-tts-autoplay.js`
- 脚本使用 `MutationObserver` 监听聊天 DOM 中最后一条助手消息的文本增长
- 按句子或短块增量加入浏览器 `speechSynthesis` 队列
- 页面右下角提供浮动控制岛：语音开关 + 菜单按钮
- 菜单可手动调整语速，默认 `1.3x`，设置保存到 localStorage
- 右上角显示朗读状态和当前语速
- 通过 `?ver=0.1Meta` cache busting 确保浏览器重新加载补丁脚本

## 2. 关键区分

### 2.1 OpenClaw `messages.tts`

`messages.tts` 是服务端 TTS 配置，能把助手回复转成音频。它适合：

- 手动生成音频
- 回复完成后生成可播放附件
- 接入 Microsoft/Azure、OpenAI、ElevenLabs 等服务端 TTS provider

但它不适合本次需求：

- 浏览器自动播放通常受限于 autoplay 策略
- 服务端 TTS 要等文本完成后才能稳定生成完整音频
- 用户实际看到的是可点击音频文件，不是边输出边说

因此本次把自动 TTS 关闭：

```json5
{
  "messages": {
    "tts": {
      "auto": "off",
      "provider": "microsoft"
    }
  }
}
```

### 2.2 Realtime Voice Provider

Realtime Voice 是完整语音对话链路，通常包含用户语音输入、模型实时理解、语音输出等。它不是本次方案的重点。

本次目标不是“实时语音通话”，而是“普通文本聊天回复自动朗读”。因此不需要配置 OpenAI/Gemini realtime voice provider。

### 2.3 浏览器 Web Speech API

Chrome 内置 Web Speech API，可直接在页面内朗读文本：

```js
const u = new SpeechSynthesisUtterance("你好");
u.lang = "zh-CN";
u.rate = 1.3;
window.speechSynthesis.speak(u);
```

优点：

- 免费、浏览器内置
- 能在文本流式增长时增量朗读
- 不需要生成音频文件
- 不影响原本文本显示

限制：

- 需要用户手势解锁，用户点击/输入/发送消息通常已经满足
- Chrome `speechSynthesis` 在较高语速或长队列下可能出现 `interrupted`、无声卡死、`onend` 不触发等问题
- 不同桌面/手机 Chrome 版本表现可能不同

## 3. 环境与文件位置

本次环境：

- OpenClaw Gateway：`2026.7.1 (2d2ddc4)`
- Control UI 目录：`/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui`
- 注入脚本：`/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/openclaw-tts-autoplay.js`
- HTML 入口：`/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/index.html`
- OpenClaw 配置：`/root/.openclaw/openclaw.json`

当前 HTML 注入点：

```html
<script src="./assets/openclaw-tts-autoplay.js?ver=0.1Meta"></script>
```

> 注意：`dist/control-ui` 是安装产物，不是稳定插件接口。升级 OpenClaw 后，`index.html` 和资产文件可能被覆盖，需要重新检查并注入。

## 4. 关闭服务端自动 TTS

配置位置：

```text
/root/.openclaw/openclaw.json
```

最终配置片段：

```json
{
  "messages": {
    "tts": {
      "auto": "off",
      "provider": "microsoft",
      "providers": {
        "microsoft": {
          "enabled": true,
          "speakerVoice": "zh-CN-XiaoxiaoNeural",
          "lang": "zh-CN",
          "outputFormat": "audio-24khz-48kbitrate-mono-mp3",
          "rate": "+10%"
        }
      }
    }
  }
}
```

这样普通助手回复不会再自动附带需要点击的音频文件，但仍保留 Microsoft provider 供手动 TTS 命令使用。

## 5. 前端自动朗读脚本

脚本文件：

```text
/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/assets/openclaw-tts-autoplay.js
```

核心配置：

```js
var CONFIG = {
  rate: 1.3,
  pitch: 1.0,
  volume: 1.0,
  debounceMs: 120,
  idleFlushMs: 700,
  maxChunk: 60,
  msPerChar: 260,
  stallMarginMs: 4000,
};
```

关键实现点：

- 使用 `MutationObserver` 监听聊天页面 DOM 变化
- 定位最后一条助手气泡：

```js
'.chat-line.assistant .chat-bubble, .chat-group.assistant .chat-bubble'
```

- 只读取助手气泡 `textContent`，避开工具消息区域
- 根据文本长度差量识别新生成内容
- 用中文标点和换行做句子级缓冲：`。！？!?；;\n`
- 长句按 `maxChunk: 60` 切短，降低 Chrome 卡死概率
- 使用自建 `speechQueue`，一次只播放一个 `SpeechSynthesisUtterance`
- `onend` 后播放下一句，避免直接把大量 utterance 塞进 Chrome 内部队列
- 页面首次点击/键盘/触摸时用空 utterance 解锁 speechSynthesis
- 右下角提供浮动控制岛：`🔊/🔇` 播报开关 + `☰` 语速菜单，并用 localStorage 持久化
- 语速菜单只保留 `-`、`+` 微调，范围限制为 `1.0x` 到 `3.0x`，默认 `1.3x`
- 右上角显示状态：`就绪`、`排队N`、`朗读中`、`卡死恢复N次`、错误码、当前语速

## 6. 注入 Control UI

修改文件：

```text
/path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/index.html
```

在入口脚本前加入：

```html
<script src="./assets/openclaw-tts-autoplay.js?ver=0.1Meta"></script>
```

为什么要带 `?ver=0.1Meta`：

- Control UI 的 `/assets/` 资源可能受 service worker 或浏览器缓存影响
- 只改同名 JS 文件时，浏览器可能继续使用旧版本
- 每次改脚本后 bump query string，能强制加载新脚本

验证当前注入：

```bash
rg -n "openclaw-tts-autoplay" \
  /path/to/nodejs/lib/node_modules/openclaw/dist/control-ui/index.html
```

预期输出包含：

```text
<script src="./assets/openclaw-tts-autoplay.js?ver=0.1Meta"></script>
```

## 7. 版本演进与踩坑记录

### 7.1 初始思路：服务端 TTS 自动音频

现象：助手回复后出现音频文件，需要用户点击播放。

问题：不满足“自动播放”和“实时流式朗读”。

解决：关闭 `messages.tts.auto`，改走浏览器 Web Speech API。

### 7.2 DOM 选择器不准

早期脚本尝试监听 `.assistant .msg-body`。后来反查当前 Control UI chat bundle，实际消息结构包含：

```text
.chat-line.assistant .chat-bubble
.chat-group.assistant .chat-bubble
.chat-line.user .chat-bubble
.chat-group.user .chat-bubble
.chat-bubble.streaming
```

问题：选择器不准会导致脚本捕获不到文本，或者无法正确识别流式状态。

解决：改为监听 `.chat-bubble`，并限定 assistant wrapper。

### 7.3 旧消息被朗读

问题：切换会话或刷新页面时，如果脚本直接读取最后一条助手消息，可能把历史消息也朗读出来。

解决：首次看到非流式消息只建立基线，不朗读；只有检测到流式文本增长时才朗读增量。

### 7.4 流式气泡替换导致误判为新消息

现象：文本输出结束时，流式 DOM 可能从 `.chat-bubble.streaming` 替换成最终消息 DOM。若用 DOM 节点身份判断消息是否连续，会把同一条消息误判为新消息，出现漏读或取消。

解决：不用单纯依赖 DOM node identity，而用文本连续性判断：

```js
full.startsWith(lastFull) || lastFull.startsWith(full)
```

只要文本是连续关系，就视作同一条助手回复。

### 7.5 Chrome 报错：`utter error: interrupted`

现象：浏览器控制台出现：

```text
[tts-autoplay] utter error: interrupted
```

可能原因：

- 脚本主动 `speechSynthesis.cancel()` 打断当前 utterance
- 新旧队列管理冲突
- DOM 重渲染时误判为用户新一轮输入或新消息
- Chrome 内部 speechSynthesis 队列不稳定

解决措施：

- 减少正常路径中的 `cancel()` 调用
- 自建队列，一次只提交一个 utterance
- `onend` 后再播放下一段，避免让 Chrome 内部队列堆积过多
- 将错误码显示到右上角徽标，便于现场定位

### 7.6 没有错误但语音提前停止

现象：控制台不再报错，但语音仍可能在回复尾部停止。

判断：这更像 Chrome `speechSynthesis` 自身卡死，可能既不触发 `onend`，也不触发 `onerror`。

解决措施（0.1Meta）：

- 每一句按长度估算最大朗读时间：

```js
var estMs = text.length * CONFIG.msPerChar + CONFIG.stallMarginMs;
```

- 超过估算时间仍未结束，则判定卡死
- 执行 `speechSynthesis.cancel()` + `speechSynthesis.resume()` 重置引擎
- 将当前句子重新放回队首重播
- 右上角徽标显示 `卡死恢复N次`

### 7.7 语速调整

早期 `rate: 1.1` 更稳，但用户反馈偏慢。本次最终使用：

```js
rate: 1.3
```

在 `rate: 1.3` 下，必须配合短句切分和卡死恢复，否则 Chrome 更容易出现中途停止。

## 8. 验证方法

### 8.1 静态验证

检查脚本语法：

```bash
cd /path/to/nodejs/lib/node_modules/openclaw/dist/control-ui
node --check assets/openclaw-tts-autoplay.js
```

检查注入版本：

```bash
rg -n "openclaw-tts-autoplay.js\?ver=0.1Meta" index.html
```

检查静态资源是否可访问：

```bash
curl -sS -m 8 -o /dev/null -w "HTTP %{http_code}\n" \
  "http://127.0.0.1:18789/assets/openclaw-tts-autoplay.js?ver=0.1Meta"
```

预期：

```text
HTTP 200
```

### 8.2 浏览器验证

Chrome 中打开 Dashboard/Control UI 后：

1. 强制刷新页面，建议 `Ctrl+Shift+R`
2. 打开 DevTools Console，确认加载日志：

```text
[tts-autoplay] loaded 0.1Meta
```

3. 发送一条测试消息，让助手输出多句中文
4. 检查右下角是否出现浮动控制岛，包含 `🔊/🔇` 和 `☰` 两个按钮
5. 点击 `☰`，确认能打开语速菜单，并能用 `-`、`+` 在 `1.0x` 到 `3.0x` 范围内调整朗读速度
6. 检查右上角状态是否在 `朗读中`、`排队N`、`就绪` 之间变化，并显示当前语速
6. 确认文本仍正常显示，同时语音自动朗读
7. 确认没有生成需要点击的音频附件

测试文本示例：

```text
这是一段语音播放测试内容。现在我会连续输出几句话，用来观察浏览器是否能够稳定朗读完整回复。

第一句用于确认语音是否能够自动开始播放。第二句用于测试中途是否会突然中断。第三句稍微长一些，看看在语速一点三倍的情况下，Chrome 的朗读队列是否还能保持稳定。最后一句，如果你能听到这里，说明这次测试至少完整读到了结尾。
```

## 9. 故障排查

### 9.1 完全没有声音

检查：

- 右下角浮动岛的播报按钮是否为 `🔇`
- Chrome 标签页是否被静音
- 系统音量是否正常
- 页面是否已经有过点击、输入或触摸手势
- Console 是否有：

```text
[tts-autoplay] no speechSynthesis
```

若没有用户手势，Chrome 可能不允许自动语音播放。点击页面或发送一条消息后再试。

### 9.2 控制台仍加载旧版本

现象：Console 显示：

```text
openclaw-tts-autoplay.js?ver=<旧标识>
```

但当前应为：

```text
openclaw-tts-autoplay.js?ver=0.1Meta
```

解决：

- 检查 `index.html` 中 query string 是否已更新
- Chrome 执行 `Ctrl+Shift+R` 强制刷新
- 必要时清理该站点缓存或 unregister service worker

### 9.3 文本输出结束后尾部没读完

观察右上角徽标：

- `排队N`：文本已捕获但队列没继续播放，偏向 Chrome speechSynthesis 卡死
- `朗读中` 长时间不变：当前 utterance 可能卡死，等待 0.1Meta 超时恢复
- `就绪` 但没读完：可能是 DOM 文本增量没有被完整捕获，需要重新检查选择器或流式 DOM 结构
- `卡死恢复N次`：0.1Meta 已经触发恢复逻辑

### 9.4 出现 `interrupted`

若出现：

```text
[tts-autoplay] utter error: interrupted
```

优先确认是否加载的是最新 `?ver=0.1Meta`。旧版本中更容易因为 cancel 或队列冲突触发该错误。

如果 0.1Meta 仍频繁出现：

- 降低 `rate`，例如 `1.2`
- 缩短 `maxChunk`，例如 `45`
- 增大 `stallMarginMs`，避免误判正常长句为卡死
- 检查是否有其他浏览器扩展或页面脚本调用 `speechSynthesis.cancel()`

## 10. 当前最终状态

当前已验证状态：

- OpenClaw 版本：`2026.7.1 (2d2ddc4)`
- 浏览器：Chrome
- 服务端自动 TTS：关闭
- 手动 TTS provider：Microsoft，`zh-CN-XiaoxiaoNeural`
- 前端脚本：`openclaw-tts-autoplay.js?ver=0.1Meta`
- 朗读方式：Chrome Web Speech API
- 语速：`rate: 1.3`
- 文本显示：保留
- 自动播放：可用
- 流式朗读：可用
- 已遇到并处理的问题：选择器不准、缓存旧脚本、旧消息误读、DOM 替换误判、`interrupted`、无错误提前停止

## 11. 后续维护建议

长期更稳的实现方向：

- 将脚本注入能力做成 OpenClaw 官方配置项或插件机制，避免直接改 `dist/control-ui`
- 若 Control UI 后续提供消息流事件 API，优先监听结构化消息事件，而不是 DOM MutationObserver
- 将 TTS 开关、语速、语言、voice 选择做成 Control UI 设置项
- 若 Chrome Web Speech 在移动端仍不稳定，可考虑后端生成短音频片段并由前端 MediaSource/WebAudio 播放，但这会显著增加实现复杂度

本次热修的价值在于实现成本低、无额外服务费用、保留文本输出，并满足 Chrome 中“边生成边朗读”的核心体验。
