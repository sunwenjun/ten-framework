# 语音上传“断点”机制分析（TEN Framework）

> 这个文档回答一个常见问题：用户说一段很长的话时，系统到底是“按标点上传”，还是“按音频流持续上传”？

## 结论先说

在本项目（以 `memory-memos-example` + `deepgram_asr_python` 为例）里：

1. **不是按标点上传音频**。音频是 `pcm_frame` 连续流式发送到 ASR（WebSocket 二进制帧）。
2. **“断点”主要由 ASR 的 `is_final` 与静音/端点策略决定**（供应商端 or 额外 Turn/VAD 扩展），而不是逗号句号。
3. `punctuate=true` 只是让识别文本更像自然书写（加标点），**不等于按标点切包上传**。

---

## 1) 当前项目的主链路（以 memory-memos-example 为例）

`property.json` 里图定义了：

- 音频从 `agora_rtc` 输出 `pcm_frame`，先到 `streamid_adapter`，再到 `stt(deepgram_asr_python)`。
- `stt` 产出的 `asr_result` 再送给 `main_control`。

见：`ai_agents/agents/examples/memory-memos-example/tenapp/property.json`。

---

## 2) 音频是如何“上传”的：连续流，而不是标点触发

### 2.1 streamid_adapter 仅附加会话信息

`streamid_adapter` 在 `on_audio_frame` 中读取 `stream_id`，写到 `metadata.session_id` 后继续转发原始音频帧，不做切句。

### 2.2 deepgram_asr_python 直接把每个音频帧发到 WebSocket

`DeepgramASRExtension.send_audio()`：

- 从 `AudioFrame` 取出 bytes；
- 调用 `recognition.send_audio_frame(audio_data)` 发送。

`DeepgramASRRecognition.send_audio_frame()`：

- 计算该帧时长（基于采样率和字节数）；
- `await self.websocket.send(audio_data)` 直接发给 Deepgram。

这就是典型流式 ASR 上传方式：**边采集边发，持续推流**。

---

## 3) “什么时候算一句结束”

### 3.1 ASR 回调给出 `is_final`

`DeepgramASRExtension.on_result()` 读取厂商返回的 `message_data["is_final"]`，再封装成 `ASRResult(final=...)` 发到图里。

所以一句是否结束，取决于 ASR 端点检测（静音、内部策略、参数），不是本地遇到句号。

### 3.2 `main_control` 只在 final 时把文本送入 LLM

`MainControlExtension._on_asr_result()` 行为非常关键：

- 收到非空文本就会根据条件中断当前播报（`event.final` 或文本长度 > 2）；
- **只有 `event.final` 为真时，才 `queue_llm_input(...)` 进入 LLM 新轮次**。

因此用户说很长一句时，系统通常会收到多次 interim（非 final）+ 某次 final，最终以 final 作为“轮次提交点”。

---

## 4) 标点在这个链路里扮演什么角色？

`deepgram_asr_python/property.json` 里默认有 `punctuate: true`。

这会影响识别文本可读性（例如补逗号句号），但项目代码里**没有“遇到标点就上传音频/就提交轮次”**的逻辑。上传行为由 `send_audio_frame` 连续执行；轮次提交由 `is_final` 驱动。

---

## 5) 进阶：项目里还有“Turn Detection + VAD”路径

在 `huggingface/property.json` 里有 `with_ten_turn_detection_and_ten_vad` 这套图：

- `vad` 监听音频，`start_of_sentence` 时触发 flush；
- `interrupt_detector(ten_turn_detection)` 接收 `text_data`，在 `is_final` 到来后调用一个轻量 LLM 判定 `finished / wait / unfinished`；
- 最终再决定是否把该 turn 作为 final 文本发给 LLM。

`turn_detector.eval()` 还会先 `remove_punctuation(text)` 再判定，说明这里也不是“按标点切轮”。

---

## 6) 完整案例（一步一步推演）

假设用户说：

“我今天要去上海出差，然后想顺便看看外滩和豫园，晚上有没有推荐的本帮菜餐厅？”

### Step A：音频采集与上传

1. RTC 每 10~20ms（常见帧长）产出 `pcm_frame`。
2. `streamid_adapter` 给帧加 `metadata.session_id` 后转发。
3. `stt(deepgram)` 把每个帧 bytes 通过 WebSocket 发送。

> 这一步完全与标点无关，因为用户说话时还没有文本标点这个概念。

### Step B：ASR 持续回传

1. Deepgram 持续回传 interim 识别结果（`is_final=false`）。
2. 用户停顿到满足端点条件后，回传某段 `is_final=true`。

### Step C：主控处理

1. `main_control` 收到 interim：可用于中断打断策略与转写展示。
2. 收到 final：`turn_id += 1`，把最终文本送入 LLM。

### Step D：LLM/TTS

1. LLM 流式输出；
2. 主控按句切给 TTS（这是输出文本分句，不是输入音频上传分段）。

---

## 7) ANSI Art 时序图

```text
+---------+        +------------------+        +----------------------+        +--------------+        +-----+
|  User   |        |   agora_rtc      |        |   streamid_adapter   |        | stt(deepgram)|        | LLM |
+----+----+        +---------+--------+        +----------+-----------+        +------+-------+        +--+--+
     |                       |                            |                           |                 |
     | speak long sentence   |                            |                           |                 |
     |---------------------->| pcm_frame(continuous)      |                           |                 |
     |                       |--------------------------->| add metadata.session_id    |                 |
     |                       |                            |--------------------------->| ws.send(audio)   |
     |                       |                            |                           |=================>| (ASR engine)
     |                       |                            |                           |<=================| interim/final
     |                       |                            |                           | asr_result       |
     |                       |                            |                           |-----> main_control
     |                       |                            |                           |       interim: not commit
     |                       |                            |                           |       final: queue_llm_input
     |                       |                            |                           |----------------------------->|
     |                       |                            |                           |                 | generate
     |                       |<--------------------------- tts pcm_frame <------------|<----------------|
```

---

## 8) 实操建议（如果你要调“断点”体验）

1. 优先调 ASR 端点参数（例如 Deepgram query params），因为 `is_final` 是核心。  
2. 需要更智能对话轮次时，启用 `ten_turn_detection + ten_vad` 图。  
3. 不要把 `punctuate` 当作断点控制参数，它主要影响文本格式。

