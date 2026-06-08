# Reachy Mini Conversation App — 架构文档

本文档基于实际代码，详细描述主流程的类结构、模块依赖与运行时序，涵盖语音输入/输出、人脸检测、声源方向估计和转头控制。

---

## 1. 模块依赖图

> 各模块的导入/调用关系，展示整体分层结构。

```mermaid
graph TD
    subgraph 入口
        MAIN["main.py\n(run / main)"]
    end

    subgraph 对话后端
        CH["conversation_handler.py\nConversationHandler (ABC)"]
        BR["base_realtime.py\nBaseRealtimeHandler (ABC)"]
        OAI["openai_realtime.py\nOpenAIRealtimeHandler"]
        HF["huggingface_realtime.py\nHFRealtimeHandler"]
        GEM["gemini_live.py\nGeminiLiveHandler"]
    end

    subgraph 运动控制
        MM["moves.py\nMovementManager"]
        MOVES["moves.py\nBreathingMove / Move"]
    end

    subgraph 感知
        CW["camera_worker.py\nCameraWorker"]
        HT["vision/head_tracking.py\nHeadTracker"]
    end

    subgraph 工具与配置
        TOOLS["tools/core_tools.py\nToolDependencies / initialize_tools"]
        MCP["mcp_client.py\nMCPClient"]
        CFG["config.py\n后端/模型/语言配置"]
        IP["idle_policy.py\nstart_idle_tool_call"]
        BTMGR["tools/background_tool_manager.py\nBackgroundToolManager"]
    end

    subgraph 机器人 SDK
        SDK["reachy_mini SDK\nReachyMini"]
    end

    MAIN --> CH
    MAIN --> MM
    MAIN --> CW
    MAIN --> TOOLS
    MAIN --> CFG

    CH --> BR
    BR --> OAI
    BR --> HF
    CH --> GEM

    BR --> TOOLS
    BR --> IP
    BR --> BTMGR
    BR --> CFG

    TOOLS --> MM
    TOOLS --> CW
    TOOLS --> SDK

    MM --> SDK
    MM --> CW
    MM --> MOVES

    CW --> HT
    CW --> SDK

    TOOLS --> MCP
```

---

## 2. 类图 (UML)

> 展示核心类、方法、属性及继承/组合关系。

```mermaid
classDiagram
    class ConversationHandler {
        <<abstract>>
        +deps: ToolDependencies
        +output_queue: asyncio.Queue
        +copy() ConversationHandler
        +start_up() None
        +shutdown() None
        +receive(frame: AudioFrame) None
        +emit() HandlerOutput
        +apply_personality(profile) str
        +get_available_voices() list
        +get_current_voice() str
        +change_voice(voice) str
    }

    class BaseRealtimeHandler {
        <<abstract>>
        +BACKEND_PROVIDER: str
        +SAMPLE_RATE: int
        +REFRESH_CLIENT_ON_RECONNECT: bool
        +client: AsyncOpenAI
        +connection: AsyncRealtimeConnection
        +cumulative_cost: float
        +receive(frame) None
        +emit() HandlerOutput
        +start_up() None
        +shutdown() None
        +apply_personality(profile) str
        -_run_realtime_session() None
        -_response_sender_loop() None
        -_tap_audio_for_daemon_wobbler(pcm) None
        -_handle_tool_result(bg_tool) None
        -_get_session_config(tool_specs) SessionConfig
    }

    class OpenAIRealtimeHandler {
        +BACKEND_PROVIDER = "openai"
        +SAMPLE_RATE = 24000
        +REFRESH_CLIENT_ON_RECONNECT = False
    }

    class HFRealtimeHandler {
        +BACKEND_PROVIDER = "huggingface"
        +SAMPLE_RATE = 16000
        +REFRESH_CLIENT_ON_RECONNECT = True
    }

    class GeminiLiveHandler {
        +BACKEND_PROVIDER = "gemini"
        +start_up() None
        +receive(frame) None
        +emit() HandlerOutput
    }

    class MovementManager {
        +current_robot: ReachyMini
        +camera_worker: CameraWorker
        +state: MovementState
        +move_queue: deque
        -_command_queue: Queue
        -_face_offsets_lock: Lock
        -_pending_face_offsets: tuple
        -_face_offsets_dirty: bool
        -_sound_direction_lock: Lock
        -_pending_sound_direction: float|None
        -_sound_direction_dirty: bool
        -_is_listening: bool
        -_breathing_active: bool
        +start() None
        +stop() None
        +queue_move(move) None
        +set_listening(listening) None
        +set_sound_direction(yaw_radians) None
        +is_idle() bool
        -_control_loop() None
        -_poll_signals(t) None
        -_apply_pending_offsets() None
        -_get_primary_pose(t) FullBodyPose
        -_get_secondary_pose() FullBodyPose
        -_compose_full_body_pose(t) FullBodyPose
        -_calculate_blended_antennas(target) tuple
        -_issue_control_command(head,ant,yaw) None
        -_manage_breathing(t) None
        -_manage_move_queue(t) None
    }

    class MovementState {
        +current_move: Move|None
        +move_start_time: float|None
        +last_activity_time: float
        +face_tracking_offsets: tuple[float x6]
        +last_primary_pose: FullBodyPose|None
        +update_activity() None
    }

    class BreathingMove {
        +interpolation_start_pose: NDArray
        +interpolation_start_antennas: tuple
        +duration = inf
        +evaluate(t) FullBodyPose
    }

    class CameraWorker {
        +reachy_mini: ReachyMini
        +head_tracker: HeadTracker|None
        +latest_frame: NDArray|None
        +face_tracking_offsets: list[float x6]
        +face_lost_delay = 2.0s
        +interpolation_duration = 1.0s
        +start() None
        +stop() None
        +working_loop() None
        +get_latest_frame() NDArray|None
        +get_face_tracking_offsets() tuple
        +set_head_tracking_enabled(enabled) None
    }

    class HeadTracker {
        +get_head_position(frame) tuple[eye_center, bbox]
    }

    class ToolDependencies {
        +reachy_mini: ReachyMini
        +movement_manager: MovementManager
        +camera_worker: CameraWorker
        +vision_processor: VisionProcessor
    }

    class ReachyMini {
        +set_target(head, antennas, body_yaw) None
        +get_current_head_pose() NDArray
        +get_current_joint_positions() tuple
        +look_at_image(x, y, ...) NDArray
        +media.get_frame() NDArray
        +media.push_audio_sample(audio) None
        +media.get_output_audio_samplerate() int
    }

    ConversationHandler <|-- BaseRealtimeHandler
    ConversationHandler <|-- GeminiLiveHandler
    BaseRealtimeHandler <|-- OpenAIRealtimeHandler
    BaseRealtimeHandler <|-- HFRealtimeHandler

    BaseRealtimeHandler --> ToolDependencies : deps
    BaseRealtimeHandler ..> MovementManager : deps.movement_manager

    MovementManager --> ReachyMini : current_robot
    MovementManager --> CameraWorker : camera_worker
    MovementManager --> MovementState : state
    MovementManager --> BreathingMove : creates (idle)

    CameraWorker --> HeadTracker : head_tracker
    CameraWorker --> ReachyMini : reachy_mini

    ToolDependencies --> ReachyMini
    ToolDependencies --> MovementManager
    ToolDependencies --> CameraWorker
```

---

## 3. 序列图

### 3.1 系统启动流程

> `main.py` 初始化所有组件并启动 fastrtc/Gradio 服务。

```mermaid
sequenceDiagram
    participant M as main.py
    participant SDK as ReachyMini SDK
    participant CW as CameraWorker
    participant MM as MovementManager
    participant TD as ToolDependencies
    participant CH as ConversationHandler
    participant APP as Gradio/FastAPI

    M->>SDK: ReachyMini(robot_name)
    SDK-->>M: robot 实例
    M->>M: initialize_camera_and_vision(args, robot)
    M->>CW: CameraWorker(robot, head_tracker)
    M->>MM: MovementManager(robot, camera_worker)
    M->>TD: ToolDependencies(robot, MM, CW, vision)
    M->>CH: ConversationHandler(deps) [按后端配置选择]
    M->>CW: camera_worker.start()
    M->>MM: movement_manager.start()
    M->>APP: Stream(handler=CH) + Gradio/FastAPI 启动
    APP-->>M: 服务已就绪
```

---

### 3.2 语音输入 → AI 推理 → 语音输出（主对话流程）

> 音频帧经 fastrtc 传入 `BaseRealtimeHandler.receive()`，发往 AI API，响应音频由 `emit()` 输出。

```mermaid
sequenceDiagram
    participant MIC as 麦克风 (fastrtc)
    participant BRH as BaseRealtimeHandler
    participant WS as AI Realtime API (WebSocket)
    participant OQ as output_queue
    participant SPK as 扬声器 / Gradio

    MIC->>BRH: receive(frame: AudioFrame)
    BRH->>BRH: 音频重采样 (resample to SAMPLE_RATE)
    BRH->>WS: input_audio_buffer.append(pcm_bytes)

    Note over WS: VAD 检测语音活动
    WS-->>BRH: input_audio_buffer.speech_started
    BRH->>BRH: _mark_activity("speech_started")
    WS-->>BRH: conversation.item.input_audio_transcription.delta
    BRH->>OQ: put(AdditionalOutputs {role:user_partial})

    WS-->>BRH: input_audio_buffer.speech_stopped
    WS-->>BRH: response.created
    WS-->>BRH: response.audio.delta (TTS 音频分片)
    BRH->>BRH: _tap_audio_for_daemon_wobbler(pcm)
    BRH->>OQ: put((sample_rate, pcm_int16))
    OQ-->>SPK: emit() → 播放语音

    WS-->>BRH: response.audio_transcript.done (完整文字)
    BRH->>OQ: put(AdditionalOutputs {role:assistant})

    WS-->>BRH: response.done (含 token 用量)
    BRH->>BRH: _compute_response_cost(usage)
```

---

### 3.3 工具调用流程

> AI 请求调用工具（如 camera、dance 等），`BackgroundToolManager` 异步执行后回送结果。

```mermaid
sequenceDiagram
    participant WS as AI Realtime API
    participant BRH as BaseRealtimeHandler
    participant BTM as BackgroundToolManager
    participant TOOL as Tool 函数
    participant MM as MovementManager

    WS-->>BRH: response.function_call_arguments.done
    BRH->>BTM: submit(tool_name, args, call_id)
    BTM->>TOOL: 调用工具函数 (async)

    alt 涉及动作 (dance/goto 等)
        TOOL->>MM: queue_move(move)
        MM-->>TOOL: OK
    end

    TOOL-->>BTM: result dict
    BTM-->>BRH: ToolNotification(id, result)
    BRH->>WS: conversation.item.create(function_call_output)
    BRH->>WS: response.create() [让 AI 根据工具结果继续回应]
```

---

### 3.4 人脸检测 → 头部跟踪流程

> `CameraWorker` 独立线程持续抓帧、检测人脸，更新偏移量；MovementManager 在控制循环中消费偏移量。

```mermaid
sequenceDiagram
    participant CAM as ReachyMini.media
    participant CW as CameraWorker (线程)
    participant HT as HeadTracker
    participant MM as MovementManager (控制循环)
    participant SDK as ReachyMini.set_target

    loop 每帧 (~30 Hz)
        CW->>CAM: get_frame()
        CAM-->>CW: frame (BGR NDArray)

        alt 人脸检测开启
            CW->>HT: get_head_position(frame)
            HT-->>CW: eye_center (normalized [-1,1])

            alt 检测到人脸
                CW->>SDK: look_at_image(px, py, perform_movement=False)
                SDK-->>CW: target_pose (4×4 矩阵)
                CW->>CW: 提取 translation + rotation (×0.6 FOV 修正)
                CW->>CW: 更新 face_tracking_offsets [x,y,z,roll,pitch,yaw]
            else 人脸丢失 (>2s)
                CW->>CW: 线性插值回中立姿态 (1s 过渡)
                CW->>CW: face_tracking_offsets → [0,0,0,0,0,0]
            end
        end
    end

    loop 控制循环 (60 Hz)
        MM->>CW: get_face_tracking_offsets()
        CW-->>MM: (x,y,z,roll,pitch,yaw)
        MM->>MM: _apply_pending_offsets() 写入 state.face_tracking_offsets
        MM->>MM: _get_secondary_pose() → secondary FullBodyPose
        MM->>MM: _compose_full_body_pose() = primary ⊕ secondary
        MM->>SDK: set_target(head, antennas, body_yaw)
    end
```

---

### 3.5 声源方向估计 → 头部偏转流程

> 多通道音频帧进入 `BaseRealtimeHandler.receive()`，根据各通道能量估计偏航角，驱动 MovementManager 转头。

```mermaid
sequenceDiagram
    participant MIC as 麦克风 (多通道)
    participant BRH as BaseRealtimeHandler
    participant MM as MovementManager
    participant STATE as MovementState
    participant SDK as ReachyMini.set_target

    MIC->>BRH: receive(frame) [shape: (N_samples, N_channels)]

    BRH->>BRH: 计算各通道 RMS 能量
    alt 立体声 (2通道)
        BRH->>BRH: yaw = clip((right-left)/(left+right) × 0.6, ±0.6 rad)
    else 多通道 (>2)
        BRH->>BRH: energy centroid → 归一化 [-1,1] → yaw (±0.6 rad)
    end

    BRH->>MM: set_sound_direction(yaw)
    MM->>MM: _command_queue.put("sound_direction", yaw)
    BRH->>BRH: 折叠为单声道 (取第0通道)
    BRH->>BRH: 继续正常语音流程 (转发到 AI API)

    Note over MM: 控制循环下一个 tick
    MM->>MM: _apply_pending_offsets()
    MM->>STATE: face_tracking_offsets[5] = yaw (覆盖 yaw 分量)
    MM->>MM: _get_secondary_pose() 使用新 yaw
    MM->>SDK: set_target(head, antennas, body_yaw)

    Note over BRH,MM: 语音结束时清除方向偏移
    BRH->>MM: set_sound_direction(None)
    MM->>STATE: face_tracking_offsets[5] 恢复 (不再由声音覆盖)
```

---

### 3.6 MovementManager 控制循环（60 Hz 主循环）

> 展示每个 tick 内主姿态、次级偏移、天线融合的完整计算路径。

```mermaid
sequenceDiagram
    participant LOOP as 控制循环 (60 Hz)
    participant CMD as _command_queue
    participant CW as CameraWorker
    participant STATE as MovementState
    participant SDK as ReachyMini.set_target

    loop 每 ~16.7 ms
        LOOP->>CMD: 消费所有待处理命令 (set_listening / queue_move / sound_direction)
        LOOP->>CW: get_face_tracking_offsets()
        CW-->>LOOP: 最新人脸偏移
        LOOP->>STATE: _apply_pending_offsets() [face + sound_direction yaw]

        LOOP->>LOOP: _manage_move_queue(t) [切换主动作]
        LOOP->>LOOP: _manage_breathing(t)
        Note right of LOOP: 无活动 >0.3s → 自动启动 BreathingMove

        LOOP->>LOOP: _get_primary_pose(t) [当前主动作.evaluate(t)]
        LOOP->>LOOP: _get_secondary_pose() [face_tracking_offsets → 4×4]
        LOOP->>LOOP: combine_full_body(primary, secondary)
        Note right of LOOP: compose_world_offset 做世界系叠加

        LOOP->>LOOP: _calculate_blended_antennas(target_antennas)
        Note right of LOOP: 聆听模式 → 天线冻结; 解除后 0.4s 平滑混合

        LOOP->>SDK: set_target(head=..., antennas=..., body_yaw=...)
    end
```

---

### 3.7 空闲呼吸动作流程

> 当无对话活动时，MovementManager 自动启动 `BreathingMove` 保持机器人"存在感"。

```mermaid
sequenceDiagram
    participant MM as MovementManager
    participant BM as BreathingMove
    participant SDK as ReachyMini.set_target

    Note over MM: 无活动 > 0.3s，未在聆听，队列空
    MM->>SDK: get_current_head_pose()
    SDK-->>MM: current_pose
    MM->>SDK: get_current_joint_positions()
    SDK-->>MM: current_antennas
    MM->>BM: BreathingMove(start_pose, start_antennas, interp_duration=1.0)
    MM->>MM: move_queue.append(breathing_move)

    loop 呼吸中 (无限 duration)
        MM->>BM: evaluate(t)
        alt t < 1.0s (过渡阶段)
            BM-->>MM: 线性插值 → 中立姿态
        else t >= 1.0s (呼吸阶段)
            BM-->>MM: z轴 ±5mm sin波 + 天线对向摇摆
        end
        MM->>SDK: set_target(head, antennas, body_yaw)
    end

    Note over MM: 新的主动作入队 → 立即打断呼吸
    MM->>MM: current_move = None; _breathing_active = False
```

---

## 4. 线程模型总览

```mermaid
graph LR
    subgraph Main Thread
        MAIN[main.py / Gradio 事件循环]
    end

    subgraph AsyncIO Event Loop
        BRH[BaseRealtimeHandler\nreceive / emit / event loop]
        BTMGR[BackgroundToolManager\n工具异步执行]
    end

    subgraph Camera Thread
        CW[CameraWorker.working_loop\n~30 Hz 人脸检测]
    end

    subgraph Movement Thread
        MM[MovementManager._control_loop\n60 Hz 控制循环]
    end

    subgraph Robot Daemon
        SDK[ReachyMini SDK\nset_target / get_frame / push_audio]
    end

    MAIN -->|创建并启动| CW
    MAIN -->|创建并启动| MM
    BRH -->|set_sound_direction via queue| MM
    BRH -->|submit tool call| BTMGR
    BTMGR -->|queue_move via queue| MM
    CW -->|face_tracking_offsets (lock)| MM
    MM -->|set_target| SDK
    CW -->|get_frame| SDK
    BRH -->|push_audio_sample| SDK
```

> **并发安全机制**：
> - `CameraWorker` 通过 `face_tracking_lock` 保护 `face_tracking_offsets`；
> - `MovementManager` 通过 `_face_offsets_lock`、`_sound_direction_lock` 暂存待处理偏移；
> - 所有主动作变更均通过 `_command_queue` 由控制线程独占执行，避免竞态。

---

## 5. 关键数据结构

| 类型 | 定义 | 说明 |
|------|------|------|
| `FullBodyPose` | `(NDArray[4×4], (float,float), float)` | 头部姿态矩阵、天线角度对、躯干偏航角 |
| `face_tracking_offsets` | `tuple[float×6]` | (x,y,z,roll,pitch,yaw) 世界系偏移，单位：米/弧度 |
| `AudioFrame` | `(int, NDArray[int16])` | (采样率, PCM 数据) |
| `FullBodyPose` 融合 | `compose_world_offset(primary, secondary)` | 次级偏移在世界系中叠加到主姿态 |

---

*文件路径参考：*
- [src/reachy_mini_conversation_app/main.py](src/reachy_mini_conversation_app/main.py)
- [src/reachy_mini_conversation_app/base_realtime.py](src/reachy_mini_conversation_app/base_realtime.py)
- [src/reachy_mini_conversation_app/moves.py](src/reachy_mini_conversation_app/moves.py)
- [src/reachy_mini_conversation_app/camera_worker.py](src/reachy_mini_conversation_app/camera_worker.py)
- [src/reachy_mini_conversation_app/conversation_handler.py](src/reachy_mini_conversation_app/conversation_handler.py)
- [src/reachy_mini_conversation_app/tools/background_tool_manager.py](src/reachy_mini_conversation_app/tools/background_tool_manager.py)
