# Pre 讲解文档：状态机驱动的睡前交互逻辑

> 这份文档用于 presentation / pre：不是逐行讲代码，而是告诉听众“这套睡前机器人为什么要用状态机、主流程在哪些文件、哪些代码最值得讲”。

## 1. 一句话总览

这个 project 的核心改造是：把原来开放式的“用户说一句、机器人答一句”聊天机器人，收束成一个适合睡前使用的**单向状态机**：

```text
chat → ask_audio → playing → auto_shutdown
```

在当前代码命名里，这条产品流程对应为：

```text
BOOT → CHAT → TRANSITION → AUDIO → SLEEP → SHUTDOWN
```

其中：

- `CHAT`：睡前情绪梳理，机器人只接住情绪，不把用户越聊越清醒。
- `TRANSITION`：对应产品里的 `ask_audio`，把用户从“聊天”轻轻带到“是否开始放松音频 / 准备放松”。
- `AUDIO`：对应产品里的 `playing`，播放白噪音、轻音乐或冥想音频。
- `SHUTDOWN`：定时结束后自动退出，生产环境可执行系统关机。

## 2. Pre 推荐讲解顺序

### 第一步：先讲为什么要用状态机

睡前场景和普通聊天不同。普通聊天追求“继续聊”，睡前机器人追求“帮助用户停下来”。所以这里不能让 LLM 自己决定流程，而要由程序控制节奏：

1. 先听用户讲今天的情绪。
2. 到达轮次上限或用户安静后，自动收束。
3. 切换到放松音频。
4. 定时自动关机。

可以强调一句：**LLM 只负责每一句怎么说，状态机负责什么时候该进入下一步。**

### 第二步：展示主流程图

建议在 slide 上放这张简化图：

```text
用户开机
  │
  ▼
BOOT：晚上好，今天过得怎么样？
  │
  ▼
CHAT：VAD 免手录音 → STT → LLM → TTS
  │        │
  │        ├─ 用户沉默 75s：软收尾
  │        └─ 达到 5 轮：强制收尾
  ▼
TRANSITION / ask_audio：现在让我们慢慢放松
  │
  ▼
AUDIO / playing：白噪音 / 轻音乐 / 冥想音频
  │
  ▼
SLEEP：晚安
  │
  ▼
SHUTDOWN：退出或关机
```

### 第三步：再讲代码文件分工

Pre 里不要平均分配时间，建议重点讲下面 5 个文件。

| 优先级 | 文件 | 讲什么 | 为什么重要 |
|---|---|---|---|
| 1 | `flow.py` | 睡前状态机本体 | 最核心，决定交互流程如何从聊天进入音频和关机 |
| 2 | `prompts.py` | LLM 每轮怎么说 | 说明 LLM 被限制在“睡前安抚”，不负责流程跳转 |
| 3 | `config.json` | 可调参数 | 说明轮次、静默超时、音频类型、关机时间都不用改代码 |
| 4 | `main.py` | 新入口 | 说明程序启动后由 `SleepFlow` 接管，而不是原来的开放聊天循环 |
| 5 | `agent.py` | GUI / STT / TTS 能力层 | 说明状态机复用原项目能力，不重写底层音频和界面 |

## 3. 重点代码讲解：`flow.py`

### 3.1 状态枚举：把产品流程写死成有限状态

`SleepState` 定义了整个睡前流程的所有节点：

```python
class SleepState(Enum):
    BOOT       = "boot"
    CHAT       = "chat"
    TRANSITION = "transition"
    AUDIO      = "audio"
    SLEEP      = "sleep"
    SHUTDOWN   = "shutdown"
```

Pre 里可以这样讲：

- 这里不是让 LLM 自由决定“下一步做什么”。
- 程序只允许它沿着固定顺序走。
- 这样睡前体验更稳定，不会突然跑题、反复追问或一直聊天。

### 3.2 `start()`：状态机主循环

`SleepFlow.start()` 是整个交互编排的入口：

```python
while self.state != SleepState.SHUTDOWN and not self._exit_flag:
    if self.state == SleepState.BOOT:
        self._run_boot()
    elif self.state == SleepState.CHAT:
        self._run_chat()
    elif self.state == SleepState.TRANSITION:
        self._run_transition()
    elif self.state == SleepState.AUDIO:
        self._run_audio()
    elif self.state == SleepState.SLEEP:
        self._run_sleep()
```

建议讲解重点：

- 每个状态只做自己的事情。
- 状态结束后调用 `_transition_to(...)` 进入下一个状态。
- 异常时 `_force_next()` 会强制推进，避免睡前机器人卡死在某一步。

### 3.3 `CHAT`：睡前对话不是无限聊天

`_run_chat()` 是 pre 的重点。它体现了“状态机控制节奏，LLM 控制语气”：

```text
录音 → 转写 → LLM 回复 → TTS 播放 → 轮次 +1
```

它有两个自动退出条件：

1. 用户安静超过 `silence_timeout_chat`，说明可能不想再说了。
2. 达到 `max_chat_rounds`，说明应该主动收束，避免越聊越精神。

可以用伪代码讲：

```python
while chat_round < max_rounds:
    audio = record_vad_with_timeout(timeout=75)

    if audio is None:
        play("soft_close")
        goto TRANSITION

    text = STT(audio)
    reply = LLM(text, prompt_with_round_info)
    TTS(reply)

    chat_round += 1

    if chat_round >= max_rounds:
        play("round5_close")
        goto TRANSITION
```

### 3.4 `TRANSITION`：对应产品里的 ask_audio

当前代码里的 `TRANSITION` 就是你说的 `ask_audio` 阶段。它不再继续追问烦心事，而是播放固定过渡语：

```python
self._set_state("idle", "准备放松")
self._play_cached("transition")
self._transition_to(SleepState.AUDIO)
```

Pre 里可以说：

- 这个阶段负责“心理换挡”。
- 从语言交互切到非语言陪伴。
- 如果后续要真的让用户选择 `[AUDIO:white]` 或 `[AUDIO:music]`，也应该放在这个阶段。

### 3.5 `AUDIO`：对应产品里的 playing

`_run_audio()` 读取配置里的音频类型和时长：

```python
audio_type = self.sleep_cfg.get("audio_type", "white_noise")
timeout = self.sleep_cfg.get("shutdown_timeout", 2700)
self._play_relax_audio(audio_type, timeout)
```

讲解重点：

- `audio_type` 可以是 `white_noise` / `light_music` / `meditation`。
- 如果存在 `sounds/relax/<audio_type>.wav`，就循环播放文件。
- 如果没有音频文件，就生成白噪音作为兜底。
- 到达 `shutdown_timeout` 后自动进入 `SLEEP` 和 `SHUTDOWN`。

## 4. LLM 内部控制标签 `[AUDIO:white/music]`

你的产品设想里有一个很重要的点：

```text
LLM 输出内部控制标签：[AUDIO:white] 或 [AUDIO:music]
程序剥离标签，不读给用户。
```

### 4.1 为什么要用控制标签

普通自然语言适合说给用户听，但不适合给程序做稳定判断。例如：

```text
“那我们放一点轻音乐吧。”
```

这句话人能理解，但程序要解析会比较不稳定。控制标签更明确：

```text
好，那我们放一点轻音乐，慢慢休息。[AUDIO:music]
```

程序可以：

1. 识别 `[AUDIO:music]`。
2. 把 `audio_type` 设置成 `light_music`。
3. 从回复里删掉标签。
4. TTS 只读：“好，那我们放一点轻音乐，慢慢休息。”

### 4.2 Pre 里怎么讲当前实现状态

建议如实说明：

- 当前代码已经有 `AUDIO` 播放状态，并通过 `config.json` 的 `sleep_flow.audio_type` 决定播放哪类音频。
- 当前代码还没有把 LLM 输出里的 `[AUDIO:white/music]` 标签解析出来。
- 这个标签解析可以作为下一步扩展，放在 `flow.py` 的 `_run_chat()` 或 `TRANSITION` 前后。

### 4.3 建议扩展位置

推荐在 `_run_chat()` 得到 `reply` 后、`gui.speak(reply)` 前增加一层解析：

```python
audio_tag = parse_audio_tag(reply)      # 识别 [AUDIO:white] / [AUDIO:music]
reply = strip_control_tags(reply)       # 删除内部标签，避免 TTS 读出来
if audio_tag:
    self.sleep_cfg["audio_type"] = audio_tag
```

这样就能保持架构清晰：

- LLM：表达“用户适合什么音频”。
- 程序：负责解析标签并执行。
- TTS：只读干净的自然语言。

## 5. 隔夜“一句话摘要”记忆

你的设想是：

```text
每晚压缩存档 → 次日开场回顾
```

### 5.1 为什么不能直接保存完整聊天

睡前聊天通常包含很多细节，如果第二天全部读回来会有两个问题：

1. 太长，影响开场体验。
2. 可能让用户重新陷入昨晚的情绪。

所以更适合保存一句非常轻的摘要，例如：

```json
{
  "date": "2026-06-20",
  "summary": "昨晚你提到工作压力有点重，我们把它先放到今天再处理。"
}
```

### 5.2 当前项目里已有的记忆基础

`agent.py` 里已有普通聊天历史的保存和加载：

- `load_chat_history()`：读取 `memory.json`。
- `save_chat_history()`：只保留最近 10 条 user / assistant 轮次。
- `chat_and_respond()`：会维护 `session_memory` 和 `permanent_memory`。

### 5.3 建议扩展位置

隔夜摘要可以加在 `flow.py` 的 `SHUTDOWN` 前：

```text
CHAT 收集当晚用户表达
  │
  ▼
进入 SHUTDOWN 前调用 summarize_night()
  │
  ▼
写入 nightly_summary.json
  │
  ▼
第二天 BOOT 时读取昨晚摘要
```

推荐讲法：

- “完整记忆”用于模型上下文。
- “一句话摘要”用于第二天开场。
- 第二天开场不是复述隐私细节，而是轻轻承接，例如：

```text
晚上好。昨天你把工作压力先放下了，今天我们也慢慢来。
```

## 6. 重点代码讲解：`prompts.py`

`prompts.py` 决定 LLM 的说话风格。Pre 里要强调：

- `BASE_SYSTEM_PROMPT`：定义它是“睡前情绪梳理机器人”。
- `CHAT_STAGE_PROMPT`：规定每轮只回应当前一句，不主动解决问题。
- `CHAT_FEW_SHOTS`：给小模型示范“短、接住、不深挖”。
- `get_chat_prompt(round_num, max_rounds)`：把当前轮次塞进 prompt，让越接近结束越少追问。

这一部分可以和 `flow.py` 对照讲：

```text
flow.py 控制：第几轮、什么时候结束、什么时候放音乐。
prompts.py 控制：这一轮应该怎么温柔地说。
```

## 7. 重点代码讲解：`config.json`

`config.json` 是演示时最适合展示的配置文件，因为它能证明这套流程不是写死在代码里的。

建议重点讲这些字段：

```json
"sleep_flow": {
  "max_chat_rounds": 5,
  "silence_timeout_chat": 75,
  "audio_type": "white_noise",
  "audio_types": ["white_noise", "light_music", "meditation"],
  "shutdown_timeout": 2700,
  "shutdown_enabled": false,
  "pre_synthesize_texts": {
    "greeting": "晚上好，今天过得怎么样？有什么想说的吗？",
    "transition": "好的，已经记下了。现在让我们慢慢放松，准备休息吧。",
    "soft_close": "如果你没什么想说的了，我们就开始放松吧。",
    "round5_close": "我们先到这里，准备休息吧。"
  }
}
```

讲解重点：

- `max_chat_rounds`：最多聊几轮。
- `silence_timeout_chat`：用户多久不说话就自动收尾。
- `audio_type`：默认播放哪种音频。
- `shutdown_timeout`：放多久后进入关机。
- `shutdown_enabled`：开发时不真关机，部署到树莓派后再打开。
- `pre_synthesize_texts`：固定话术提前缓存，减少树莓派运行时 TTS 延迟。

## 8. 重点代码讲解：`main.py`

`main.py` 是新的启动入口。它做的事情很简单：

1. 创建 Tk GUI。
2. 创建 `BotGUI`。
3. 创建 `SleepFlow(config=CURRENT_CONFIG, gui=app)`。
4. 在后台线程运行 `flow.start()`。
5. 前台继续跑 `root.mainloop()`。

Pre 里可以用一句话总结：

> `main.py` 把原项目的“能力层”接到新的“睡前状态机编排层”上。

## 9. 重点代码讲解：`agent.py`

`agent.py` 不建议逐行讲，因为它很长。只需要讲它给状态机提供了哪些能力：

| 能力 | `SleepFlow` 怎么用 |
|---|---|
| `set_state()` | 切换 GUI 表情和状态栏，比如 listening / speaking / sleep |
| `transcribe_audio()` | 把 VAD 录到的 wav 转成文字 |
| `speak()` | 把 LLM 回复或固定话术读出来 |
| `play_sound()` | 播放预合成话术或放松音频 |
| `_render()` | 预合成固定话术，写入 cache |
| `safe_exit()` | 状态机结束后安全退出 GUI 和音频资源 |

可以强调：

- `agent.py` 是底层能力层。
- `flow.py` 是产品流程层。
- 这样分层后，后续改交互流程主要改 `flow.py`，不用频繁动底层音频代码。

## 10. Pre 最后可以讲的扩展路线

建议最后用 “已完成 / 下一步” 收束：

### 已完成

- 用 `SleepFlow` 把睡前流程变成单向状态机。
- `CHAT` 阶段支持 VAD 免手录音、STT、LLM、TTS。
- 用户沉默或达到轮次上限后自动进入放松音频。
- `AUDIO` 阶段支持白噪音 / 文件音频播放和定时结束。
- `config.json` 支持调轮次、超时、音频类型和关机开关。

### 下一步

- 把 `TRANSITION` 命名或产品表达对齐为 `ask_audio`。
- 增加 `[AUDIO:white]` / `[AUDIO:music]` 控制标签解析。
- TTS 前剥离内部控制标签，避免读给用户。
- 增加 nightly summary：每晚生成一句话摘要，第二天 `BOOT` 阶段轻轻回顾。
- 把 `shutdown_enabled` 在树莓派部署时打开，实现真正自动关机。

## 11. 推荐 Pre 讲稿提纲

可以按 6 分钟来讲：

1. **问题背景（40 秒）**  
   普通聊天机器人会越聊越久，但睡前场景需要收束。

2. **核心设计（60 秒）**  
   LLM 负责语言，状态机负责流程。展示 `chat → ask_audio → playing → auto_shutdown`。

3. **代码主线（120 秒）**  
   重点讲 `flow.py`：`SleepState`、`start()`、`_run_chat()`、`_run_audio()`。

4. **Prompt 约束（60 秒）**  
   讲 `prompts.py` 如何限制模型“不解决问题、不深挖、接近结束就收束”。

5. **配置化（40 秒）**  
   讲 `config.json` 里的轮次、静默时间、音频类型、关机时间。

6. **扩展点（80 秒）**  
   讲 `[AUDIO:white/music]` 标签剥离和隔夜一句话摘要。

## 12. 一句话结尾

这部分 pre 的核心结论可以这样说：

> 我们不是让 LLM 自己决定整个睡前流程，而是用状态机把体验固定成“倾听、收束、放松、关机”。LLM 只在每个状态里生成温柔、短句的回应；真正的节奏、音频播放和自动关机都由程序控制，所以体验更稳定，也更适合睡前使用。
