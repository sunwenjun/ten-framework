# 语音问题在服务端处理后如何下发到端侧（文本 + 语音）

本文基于当前仓库中的 **ai_agents/voice-assistant + playground** 实现进行分析，回答：

1. 服务器端收到用户语音后，结果如何下发？
2. 是否“先回复文本和语音，然后端侧一边显示文字、一边播报语音”？
3. 给出一条完整案例并逐步推理。

---

## 一句话结论

是的，当前实现本质上是**双通道并行下发**：

- **语音通道**：LLM 生成内容被切句后送入 TTS，TTS 输出 `pcm_frame`，通过 `agora_rtc` 音频轨发布，端侧订阅后直接播放。
- **文本通道**：ASR/LLM 文本被封装为 `message` 数据，经 `message_collector2` 做 base64 分片后发为 `data`，由 `agora_rtc` 数据流送到前端，前端重组后写入聊天 UI。

这两个通道是并行和流式的，不是“等整句都完成再一次性发”。

---

## 端到端流程（结合当前代码）

## 1) 图结构决定了数据走向

`voice_assistant` 预定义图中：

- `agora_rtc` 负责 RTC 音视频/数据。
- `stt` 提供 ASR。
- `llm` 提供大模型回答。
- `tts` 合成音频。
- `main_control` 负责编排。
- `message_collector`（`message_collector2`）负责文本消息分片下发。
- `streamid_adapter` 将 `stream_id` 写入音频 metadata 供后续链路使用。

关键连接：

- `stt.asr_result -> main_control`
- `tts.pcm_frame -> agora_rtc`
- `message_collector.data -> agora_rtc`
- `agora_rtc.pcm_frame -> streamid_adapter -> stt`

见 `property.json` 的 nodes/connections 定义。

## 2) 用户语音上行（端侧 -> 服务器/Agent）

- 前端 `rtcManager` 创建麦克风轨并发布到 RTC。 
- Agent 侧 `agora_rtc` 订阅远端音频，音频帧进入图。
- `streamid_adapter` 在音频帧上写入 `metadata.session_id = stream_id`，再转发到 STT。

这样后续 ASR 结果能关联到哪个用户会话。

## 3) ASR 结果进入主控

`main_control/agent.py` 在 `on_data` 收到 `asr_result` 时，构造成 `ASRResultEvent(text, final, metadata)`。

`main_control/extension.py` 的 `_on_asr_result`：

- 读取 `event.metadata.session_id` 作为会话来源。
- 非空文本时：
  - 若 `final` 或长度超过阈值，先触发 `_interrupt()`（打断当前 LLM/TTS 输出）。
  - 若 `final=true`，把整句送入 LLM 队列。
- 同时把 ASR 文本通过 `_send_transcript(role="user", data_type="transcribe")` 送到 `message_collector`，供前端实时显示。

## 4) LLM 输出被“同时”用于文本和语音

`_on_llm_response` 中有两条并行路径：

- **语音路径**：
  - 非 final 时，先 `parse_sentences` 按标点切句；每个完整句立即 `_send_to_tts(..., is_final=False)`。
  - final 时，把剩余片段 `_send_to_tts(..., is_final=True)` 收尾。
- **文本路径**：
  - 每次回调都 `_send_transcript(role="assistant", text=event.text, is_final=event.is_final)` 到 `message_collector`。

因此不是“先全量文本，再播音”；而是边生成边推送，语音与文本近实时并行。

## 5) message_collector2 如何下发文本

`message_collector2` 做了三件事：

1. 收到 `message` 数据后，转为 JSON 字符串。
2. 进行 base64 编码并按 `MAX_CHUNK_SIZE_BYTES=1024` 分片，格式：
   `message_id|part_index|total_parts|base64_content`
3. 逐片以 `Data("data")` 下发（带 40ms 节流），由 `agora_rtc` 走 RTC 数据流发送到前端。

## 6) 前端如何“边播边显示”

- **播报**：前端 `rtcManager` 在 `user-published(audio)` 时调用 `_playAudio()`，远端音轨直接播放。
- **文字**：前端监听 `stream-message`，解析分片、按 `message_id` 重组、base64 解码出 JSON，然后 `emit("textChanged")`。
- `RTCCard` 订阅 `textChanged` 后 `dispatch(addChatItem)`。
- `global.ts` 的 `addChatItem` 会把同一 user 的非 final 项就地更新、final 项定稿，形成“打字机/流式”体验。

---

## 完整案例（一步一步演示）

假设用户说：

> “今天天气怎么样？顺便给我三条穿衣建议。”

### Step 0: 会话建立

- 浏览器加入 Agora channel，发布麦克风轨。
- Agent 通过 server `/start` 启动后也加入同一 channel。

### Step 1: 用户语音进入 ASR

- 用户音频帧经 `agora_rtc -> streamid_adapter -> stt`。
- `streamid_adapter` 把 `stream_id` 注入 metadata。 
- STT 输出中间稿与终稿：
  - 中间稿：`"今天天气怎么样" final=false`
  - 终稿：`"今天天气怎么样？顺便给我三条穿衣建议。" final=true`

### Step 2: main_control 收到 ASR

- 中间稿：触发 `_send_transcript(role=user)`，前端先看到用户文本正在变化。
- 终稿：
  1. `_interrupt()` 打断旧轮回答；
  2. `queue_llm_input(final_text)` 送入 LLM；
  3. 同时 `_send_transcript(role=user, final=true)`，前端把用户问题定稿。

### Step 3: LLM 流式输出

LLM 可能依次给出：

- delta1: `"今天气温"`
- delta2: `"偏低，建议"`
- delta3: `"分层穿搭。"`（出现句号）

此时：

- 文本路径：每次都把当前 `event.text` 发到 message_collector，前端不断刷新 assistant 气泡。
- 语音路径：当 `parse_sentences` 凑成完整句 `"今天气温偏低，建议分层穿搭。"`，立即送 TTS 开播，不等整段结束。

### Step 4: TTS 与文本并行下发

- TTS 输出 `pcm_frame`，通过 `agora_rtc` 发成远端音频轨，前端直接播放。
- message_collector 将 assistant 文本分片后通过 RTC data stream 发给前端。
- 用户端看到文字持续增长，同时听到语音持续播报。

### Step 5: 终稿收尾

- LLM `is_final=true` 后，`main_control` 把残余片段最后一次送 TTS（`text_input_end=true`）。
- 文本也发 final，前端将最后一条 assistant 消息标记为终稿。

---

## 为什么看起来像“同时显示+播报”

根因有三：

1. LLM 事件是流式回调（delta + final）。
2. main_control 对同一 LLM 事件同时走文本和TTS两路。
3. 前端对文本是增量更新（非 final 覆盖），对音频是轨道直接播放。

所以体验层面是“边生成边说，边说边显示”。

---

## ANSI Art 时序图

```ansi
+---------+      +------------+      +--------------+      +------+      +------+      +------------------+      +--------------+
| Browser |      | agora_rtc  |      | streamid_adp |      | STT  |      | Main |      | message_collector|      | Front rtcMgr |
| (Mic/UI)|      | (Agent侧)  |      |              |      |      |      | Ctrl |      |   (message_col2) |      | + Redux UI   |
+----+----+      +-----+------+      +------+-------+      +--+---+      +--+---+      +---------+--------+      +------+-------+
     |                 |                    |                 |             |                      |                      |
     | audio track --->|                    |                 |             |                      |                      |
     |                 | pcm_frame -------->|                 |             |                      |                      |
     |                 |                    | add metadata    |             |                      |                      |
     |                 |                    | pcm_frame ----->|             |                      |                      |
     |                 |                    |                 | asr_result ->|                      |                      |
     |                 |                    |                 |             | send user transcript ->| data chunks -------->|
     |                 |                    |                 |             | queue llm input        |                      |
     |                 |                    |                 |             |<-- llm delta/final ----| (internal)           |
     |                 |<--- pcm_frame (tts)------------------|             | send tts_text_input    |                      |
     |<==== remote audio playback ============================|             |                      |                      |
     |                 |<--- data(stream-message) --------------------------|<----- chunks ----------|                      |
     |                 |                    |                 |             |                      | reconstruct + decode  |
     |                 |                    |                 |             |                      | textChanged ----------> UI append/update
     |  用户感知：一边听语音播报，一边看文字实时更新                                                                         |
```

---

## 关键结论复述

- 不是“严格先文本后语音”或“先语音后文本”，而是**同一轮回答走两个并行流**。
- 文本走 RTC data stream（经 message_collector 分片），语音走 RTC audio track（经 tts 输出）。
- 前端对文本做流式重组/更新，对音频直接播放，因此用户感知为同步进行。

