# CameraViewer：View 驱动的 Pull 模型

> **核心结论：** 首帧由 `CameraPanel` 进入 `RUNNING` 时请求；后续成功帧只有在
> `ImageViewport.paintEvent()` 真正完成绘制后才会请求。采集循环不依赖固定定时器。

## 线程模型：实际有多少条线程？

时序图中的参与者表示职责，不代表每个参与者各占一条线程。项目显式定义的业务执行上下文如下：

| 执行上下文 | 数量 | 负责内容 |
| --- | --- | --- |
| GUI 主线程 | 固定 1 条 | 窗口、面板、状态汇总、单路调度和所有界面更新 |
| `QThreadPool` 工作线程 | 同时最多 3 条 | 文件读取、JPEG 解码和 BGR→RGB 转换 |
| `_FrameWorker` | 任务对象，不是线程 | 描述一次“读取并处理一帧”的工作，由线程池调度 |

`CaptureCoordinator` 将线程池上限设置为三，是因为三路相机各自最多只有一个在途任务。
因此高峰时最多并行执行三个单帧任务，加上 GUI 主线程，业务代码最多同时在四条线程上执行。

线程池按需创建并复用工作线程。任务结束后，线程可能在池中空闲一段时间再退出，所以操作系统中
看到的线程数不一定等于当前在途任务数。这里描述的是项目主动建立的线程模型，不包含 Qt、图形
驱动或 OpenCV 可能使用的内部辅助线程。

## 主时序图：启动与逐帧采集

图中每条消息先说明动作含义，括号内才是对应函数或信号。

```mermaid
sequenceDiagram
    autonumber

    actor User as 用户

    box GUI 主线程
        participant Toolbar as CaptureToolbar
        participant MW as MainWindow
        participant Coord as CaptureCoordinator
        participant Panel as CameraPanel
        participant View as ImageViewport
        participant Ctrl as CameraController<br/>代表任意一路
        participant Pool as QThreadPool<br/>调度入口
    end

    box 线程池工作线程（按需复用，并发上限 3）
        participant Worker as _FrameWorker<br/>单帧任务
    end

    participant Source as ImageSequenceSource<br/>共享数据源

    User->>Toolbar: 表达开始采集意图（点击“启动采集”）
    Toolbar->>MW: 通知采集按钮被触发（capture_requested）
    MW->>Coord: 请求启动全部相机（start(source)）
    Coord->>Source: 只验证目录和 500 个文件是否存在（validate()）
    Source-->>Coord: 返回序列验证成功

    loop 为三路相机分别建立运行代次
        Coord->>Ctrl: 注入共享数据源并进入运行态（start(source)）
        Ctrl-->>Coord: 汇报单路状态，供全局状态汇总（state_changed）
        Ctrl-->>Panel: 通知对应面板进入采集中（state_changed → apply_state()）
        Panel->>Ctrl: 请求该路首帧（frame_requested → request_frame()）
    end

    loop 每路相机独立重复以下逐帧流程
        Ctrl->>Ctrl: 检查运行态和在途任务（request_frame()）

        alt 当前已有在途任务
            Ctrl->>Ctrl: 将重复需求合并为一个待提交标记（_pending = True）
        else 当前没有在途任务
            Ctrl->>Ctrl: 锁定帧号与启停代次，并占用在途名额（_submit()）
            Ctrl->>Pool: 提交一次单帧任务（start(worker)）
            Pool->>Worker: 在可用工作线程执行任务（run()）
            Worker->>Source: 读取文件、解码并转换为 RGB（load_rgb(frame_number)）
            Source-->>Worker: 返回 RGB NumPy 数组或抛出单帧异常
            Worker->>Worker: 将成功数据或异常统一封装为结果（FrameResult）
            Worker-->>Ctrl: 将帧结果跨线程排队投递回 GUI（finished → _on_worker_finished()）

            Ctrl->>Ctrl: 释放在途名额并确认结果属于当前代次（_release_worker() / _is_current_result()）

            alt 结果来自旧代次或相机已停止
                Ctrl->>Ctrl: 丢弃迟到结果，不显示也不推进游标
            else 结果属于当前运行代次
                Ctrl->>Ctrl: 推进游标并更新连续失败计数（_settle_current_result()）
                Ctrl-->>Panel: 交付成功帧或错误信息（result_ready → show_result()）

                alt 成功帧
                    Panel->>View: 保存新图像并请求 Qt 重绘（set_frame()）
                    View->>View: 将新图像实际绘制到界面（paintEvent()）
                    View-->>Panel: 通知本帧已经绘制完成（frame_painted）
                    Panel->>Ctrl: 请求下一帧（frame_requested → request_frame()）
                else 失败结果
                    Panel->>Panel: 显示错误并保留上一张成功图像
                    Panel->>Ctrl: 不等待绘制，立即请求下一帧（frame_requested → request_frame()）
                end
            end
        end
    end
```

这张图中的 `QThreadPool` 是调度入口，`_FrameWorker` 是被调度的任务。只有
`_FrameWorker.run()` 及其调用的 `load_rgb()` 在线程池工作线程执行；结果结算、状态信号、
面板更新和绘制都回到 GUI 主线程。

## 停止后立即重启：为什么旧结果不会串入新会话？

```mermaid
sequenceDiagram
    autonumber

    actor User as 用户

    box GUI 主线程
        participant Toolbar as CaptureToolbar
        participant MW as MainWindow
        participant Coord as CaptureCoordinator
        participant Panel as CameraPanel
        participant Ctrl as CameraController
        participant Pool as QThreadPool<br/>调度入口
    end

    box 线程池工作线程
        participant OldWorker as 旧代次 _FrameWorker
        participant NewWorker as 新代次 _FrameWorker
    end

    User->>Toolbar: 表达停止采集意图（点击“停止采集”）
    Toolbar->>MW: 通知采集按钮被触发（capture_requested）
    MW->>Coord: 停止全部相机（stop()）
    Coord->>Ctrl: 增加代次、清空待提交需求并进入停止态（stop()）
    Note over Ctrl,OldWorker: 已经运行的旧任务无法强制取消，只允许它自然结束

    User->>Toolbar: 立即表达重新启动意图（再次点击“启动采集”）
    Toolbar->>MW: 再次通知采集按钮被触发（capture_requested）
    MW->>Coord: 启动全部相机（start(source)）
    Note over MW,Coord: 数据源验证与主时序图相同，此处省略
    Coord->>Ctrl: 再次增加代次并进入运行态（start(source)）
    Ctrl-->>Panel: 通知面板进入采集中（state_changed → apply_state()）
    Panel->>Ctrl: 请求新代次首帧（frame_requested → request_frame()）
    Ctrl->>Ctrl: 旧任务仍在途，因此保留一个新代次需求（_pending = True）

    OldWorker-->>Ctrl: 旧任务完成并投递结果（finished → _on_worker_finished()）
    Ctrl->>Ctrl: 发现 generation 不匹配，丢弃旧结果
    Ctrl->>Pool: 提交之前保留的新代次需求（_submit_pending_if_needed() → _submit()）
    Pool->>NewWorker: 在可用工作线程执行新代次任务（run()）
```

## 关键类图（UML）

```mermaid
classDiagram
    direction LR

    class MainWindow {
        <<GUI线程>>
        +start_all()
        +stop_all()
        -_connect_signals()
    }

    class CaptureToolbar {
        <<GUI线程>>
        +browse_requested
        +capture_requested
        +image_directory
        +apply_state(state)
    }

    class CaptureStatusBar {
        <<GUI线程>>
        +apply_state(state)
        +set_activity(active, capacity)
    }

    class CaptureCoordinator {
        <<GUI线程>>
        +controllers
        +state
        +start(source)
        +stop()
        +shutdown(timeout_ms)
        +state_changed
        +activity_changed
    }

    class CameraController {
        <<GUI线程>>
        +state
        +cursor
        +start(source)
        +stop()
        +request_frame()
        -_submit()
        -_on_worker_finished(result)
    }

    class QThreadPool {
        <<Qt调度器>>
        +setMaxThreadCount(3)
        +start(runnable)
        +clear()
        +waitForDone(timeout_ms)
    }

    class _FrameWorker {
        <<线程池任务>>
        +run()
    }

    class FrameSource {
        <<abstract>>
        +frame_count
        +validate()
        +load_rgb(frame_number)
    }

    class ImageSequenceSource {
        +directory
        +frame_path(frame_number)
        +validate()
        +load_rgb(frame_number)
    }

    class CameraPanel {
        <<GUI线程>>
        +frame_requested
        +apply_state(camera_id, state, message)
        +show_result(result)
    }

    class ImageViewport {
        <<GUI线程>>
        +frame_painted
        +set_frame(rgb)
        +paintEvent(event)
    }

    class FrameResult {
        <<不可变数据>>
        +camera_id
        +generation
        +frame_number
        +rgb
        +error
        +ok
    }

    class CameraState {
        <<enum>>
        IDLE
        RUNNING
        STOPPED
        ERROR
    }

    class GlobalCaptureState {
        <<enum>>
        IDLE
        RUNNING
        STOPPED
        PARTIAL_ERROR
        ERROR
    }

    MainWindow *-- CaptureToolbar : 组合
    MainWindow *-- CaptureStatusBar : 组合
    MainWindow *-- CaptureCoordinator : 组合
    MainWindow *-- "3" CameraPanel : 组合

    CaptureCoordinator *-- "1" QThreadPool : 管理
    CaptureCoordinator *-- "3" CameraController : 管理
    CaptureCoordinator --> GlobalCaptureState : 汇总

    CameraPanel *-- ImageViewport : 组合
    CameraPanel ..> CameraController : 请求帧
    CameraController ..> CameraPanel : 状态与结果信号

    CameraController --> FrameSource : 保存共享引用
    CameraController --> CameraState : 维护
    CameraController ..> _FrameWorker : 每帧创建
    QThreadPool ..> _FrameWorker : 调度执行
    _FrameWorker --> FrameSource : 读取一帧
    _FrameWorker ..> FrameResult : 产生
    CameraController ..> FrameResult : 结算并发送
    CameraPanel ..> FrameResult : 展示

    FrameSource <|-- ImageSequenceSource : 实现
    MainWindow ..> ImageSequenceSource : 启动时创建
    CaptureToolbar --> GlobalCaptureState : 展示
    CaptureStatusBar --> GlobalCaptureState : 展示

    note for _FrameWorker "实例代表一次单帧任务，不代表一条新线程"
    note for CameraController "控制器始终在 GUI 线程结算结果和改变状态"
```

类之间有两条主要链路：

- **控制链路：** `MainWindow → CaptureCoordinator → CameraController`，负责验证、启停、
  代次隔离和状态汇总。
- **帧链路：** `CameraPanel → CameraController → _FrameWorker → FrameSource`，结果再由
  `CameraController → CameraPanel → ImageViewport` 返回界面。

## Pull 模型的关键约束

| 约束 | 含义 | 对应代码 |
| --- | --- | --- |
| View 决定何时继续 | Controller 不主动循环；首帧和下一帧需求都来自 View | `CameraPanel.apply_state()`、`ImageViewport.paintEvent()` |
| 每路最多一个在途任务 | 重复请求只设置一个 `_pending` 标记，不形成帧队列 | `CameraController.request_frame()`、`_submit()` |
| 绘制完成才继续 | `set_frame()` 只请求重绘，真正的下一次 Pull 来自 `paintEvent()` | `ImageViewport.set_frame()`、`paintEvent()` |
| 失败也推进游标 | 单帧损坏不会卡死在同一文件，失败结果交付后立即继续 | `CameraController._settle_current_result()`、`CameraPanel.show_result()` |
| 旧代次结果必须丢弃 | 停止或重启前创建的任务不能改变新会话状态 | `CameraController._is_current_result()` |

这些约束形成天然背压：读取、解码或绘制变慢时，下一次需求也会相应推迟，不会无限预取或积压帧。

## 主要异常与边界场景

| 场景 | 处理方式 |
| --- | --- |
| 目录不存在或 500 帧序列不完整 | 协调器捕获验证异常并将三路 Controller 设为 `ERROR`，不发起首帧 Pull |
| 已有在途 Worker 时再次收到请求 | 不创建第二个任务，只设置 `_pending=True`；多个重复请求合并为一个 |
| 单帧读取、解码或处理失败 | 推进 cursor、显示错误、保留上一张成功图像并立即 Pull 下一帧 |
| 连续 500 帧失败 | 仅对应 Controller 进入 `ERROR`，其他两路继续运行 |
| 采集中点击停止 | 增加 generation、清除 pending 并进入 `STOPPED`；最后一张成功图像保留 |
| Worker 在停止后迟到返回 | generation 不匹配，结果不显示且 cursor 不推进 |
| 停止后立即重启，旧 Worker 尚未结束 | 新需求保存为 pending；旧结果被丢弃后提交新代次任务 |
| 普通窗口重绘 | 不触发 Pull；只有 `set_frame()` 标记的新帧会在绘制后发出 `frame_painted` |

## 核心代码入口

| 职责 | 类与方法 | 文件 |
| --- | --- | --- |
| 全局启动/停止 | `MainWindow.start_all()` / `stop_all()` | [`main_window.py`](../cameraviewer/main_window.py) |
| 数据源验证与三路协调 | `CaptureCoordinator.start()` / `stop()` | [`capture.py`](../cameraviewer/capture.py) |
| 首帧请求 | `CameraPanel.apply_state()` | [`widgets.py`](../cameraviewer/widgets.py) |
| 绘制完成后的下一帧请求 | `ImageViewport.paintEvent()` | [`widgets.py`](../cameraviewer/widgets.py) |
| 单路 Pull 调度 | `CameraController.request_frame()` / `_submit()` | [`capture.py`](../cameraviewer/capture.py) |
| 结果结算与迟到过滤 | `CameraController._on_worker_finished()` | [`capture.py`](../cameraviewer/capture.py) |
| 后台单帧任务 | `_FrameWorker.run()` | [`capture.py`](../cameraviewer/capture.py) |

> **判断实现是否正确：** 首帧由 View 发起；成功帧只有实际绘制后才继续；失败结果交付后
> 立即继续；旧 generation 的结果永远不能进入 View。
