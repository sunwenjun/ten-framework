---
title: TEN Framework 模块划分与实现逻辑全景分析（含案例推演）
_portal_target: development/project_module_analysis.cn.md
---

本文基于仓库当前代码进行分析，目标回答四个问题：

1. 当前项目有多少个模块；
2. 模块如何划分、目录结构如何组织；
3. 关键运行逻辑如何在代码中实现；
4. 给出一个完整案例并做逐步推理演示（最后附 ANSI Art 时序图）。

---

## 1) 模块数量结论

从根 `BUILD.gn` 的 `group("ten_framework_all")` 来看，项目主构建链路可分为：

- **10 个固定模块（默认参与）**
  - `core/src/ten_runtime`
  - `core/src/ten_runtime/binding`
  - `packages/core_addon_loaders`
  - `packages/core_apps`
  - `packages/core_extensions`
  - `packages/core_protocols`
  - `packages/core_systems`
  - `packages/example_apps`
  - `packages/example_extensions`
  - `third_party`
- **最多 3 个可选模块（按编译开关启用）**
  - `core/src/ten_rust`（`ten_enable_ten_rust`）
  - `core/src/ten_manager`（`ten_enable_ten_manager`）
  - `tests`（`ten_enable_tests`）

> 因此：
>
> - **默认主干 = 10 模块**
> - **全部打开 = 13 模块**

---

## 2) 模块划分与目录结构

### 2.1 顶层职责划分（按工程视角）

```text
ten-framework/
├─ core/                 # 运行时核心能力（app/extension/env/msg 等）
├─ packages/             # 可复用包：core_* + example_*
├─ build/                # GN 构建规则与编译选项
├─ tests/                # runtime / util / manager 等测试集
├─ docs/                 # 文档（多语言）
├─ tools/                # 工具链与开发辅助脚本
├─ ai_agents/            # Agent 示例与服务端/客户端
└─ third_party/          # 第三方依赖
```

### 2.2 `core/src/ten_runtime` 内部子模块（25 个）

`ten_runtime` 是核心中的核心，可进一步拆成 25 个子目录（等价为 25 个子模块）：

- `app`：应用生命周期（configure/init/deinit/run/wait）
- `extension` / `extension_group` / `extension_thread` / `extension_context` / `extension_store`
- `ten_env` / `ten_env_proxy`
- `msg` / `msg_conversion`
- `connection` / `protocol` / `remote`
- `engine` / `graph_proxy` / `path`
- `addon` / `addon_loader`
- `binding`（C/C++/Go/Python/Node 等语言桥接）
- `metadata` / `schema_store` / `timer` / `common` / `global` / `test`

一句话理解：

- **App 是进程级入口**；
- **Extension 是业务单元**；
- **TenEnv 是运行时上下文与消息操作面板**；
- **Graph/Engine 负责把“谁给谁发什么”落地执行**。

---

## 3) 关键逻辑如何实现（结合代码）

下面从接口层到示例层，串起核心机制。

### 3.1 生命周期接口：App 与 Extension

在 C 层 API 中，`app` 和 `extension` 都是“回调驱动”的生命周期对象：

- `ten_app_create / ten_app_run / ten_app_wait` 管应用级生命周期；
- `ten_extension_create` 注入 `on_configure/on_init/on_start/on_stop/on_cmd/on_data/on_audio_frame/on_video_frame` 等回调。

这代表 TEN 的执行模型本质上是：

- **框架驱动生命周期；业务代码实现回调。**

### 3.2 消息发送与应答：TenEnv

`ten_env` 负责运行期通信动作，最关键 API：

- `ten_env_send_cmd`
- `ten_env_send_data`
- `ten_env_send_video_frame`
- `ten_env_send_audio_frame`
- `ten_env_return_result`

这形成了最小闭环：

1. 收到 `cmd`；
2. 业务处理；
3. `return_result` 返回给调用方。

### 3.3 配置阶段与“生命周期完成通知”

TEN 要求每个生命周期回调中显式调用 `on_xxx_done`（例如 `on_configure_done/on_start_done`）。

意义是：

- 允许同步/异步初始化；
- 框架可精确知道状态推进点，避免竞态。

### 3.4 示例扩展：`simple_echo_cpp`

`packages/example_extensions/simple_echo_cpp/src/main.cc` 展示了典型处理：

- `on_cmd`：读取命令名，构造 `cmd_result` 并设置 `detail`，再 `return_result`；
- `on_data`：拷贝 buffer 后 `send_data`；
- `on_video_frame/on_audio_frame`：复制帧数据与元信息后继续向下游发送。

这是“无状态透传 + 最小加工”的标准范式。

---

## 4) 完整案例：从测试代码推演一次命令闭环

这里选 `tests/ten_runtime/smoke/notify_test/normal_func_in_lambda.cc`，它非常适合观察真实运行链路。

### 4.1 案例目标

客户端发送 `hello_world` 到扩展，扩展通过外部线程触发 `ten_env_proxy->notify(...)`，最终返回 `"hello world, too"`。

### 4.2 案例分步推理

#### Step A：启动 App

- 测试创建 `test_app` 线程并执行 `run`。
- 在 `on_configure` 里写入运行配置（包含 `msgpack://127.0.0.1:8001/`）。

推理点：

- App 先起来并监听 URI，客户端后续才可连通。

#### Step B：客户端下发 `start_graph`

- 测试客户端构造 `start_graph_cmd`。
- graph 中注册了一个节点 `test_extension`，addon 指向本测试里注册的扩展。

推理点：

- graph 是“路由拓扑”，决定命令能否抵达目标 extension。

#### Step C：发送业务命令 `hello_world`

- 客户端把 `hello_world` 的目的地设为 `test_extension`。
- 扩展 `on_cmd` 收到后，并不立刻回包，而是：
  - 置位 `trigger=true`；
  - 缓存命令对象。

推理点：

- 该测试故意把“真正回包动作”放到外部线程，验证线程切换场景下 API 的正确性。

#### Step D：外部线程通过 `ten_env_proxy` 回到 TEN 线程

- `outer_thread_main` 轮询到 `trigger=true` 后，调用 `ten_env_proxy->notify(...)`。
- `notify` 回调中执行 `extension_on_notify`：
  - 创建 `cmd_result`；
  - `detail = "hello world, too"`；
  - `ten_env.return_result(...)`。

推理点：

- 这是跨线程安全调用范式：外部线程不直接操作不安全对象，而是借 `ten_env_proxy` 把逻辑切回 TEN 上下文。

#### Step E：客户端收到结果并断言

- 测试断言状态码 OK；
- 断言 `detail == "hello world, too"`。

推理点：

- 至此形成“命令 -> 扩展 -> 结果”的完整闭环，且验证了线程切换后的一致性。

---

## 5) ANSI Art 时序图（案例流程）

```ansi
+-------------+        +-------------------+        +----------------------+        +----------------+
|   Client    |        |      TEN App      |        |    test_extension    |        | outer thread   |
+-------------+        +-------------------+        +----------------------+        +----------------+
       |                          |                             |                             |
       | start_graph_cmd          |                             |                             |
       |------------------------->| build graph & load addon    |                             |
       |                          |---------------------------->| on_start() create proxy      |
       |                          |                             |----------------------------->| thread loop
       |<-------------------------| cmd_result(OK)              |                             |
       |                          |                             |                             |
       | hello_world cmd          |                             |                             |
       |------------------------->| route by graph              |                             |
       |                          |---------------------------->| on_cmd(): trigger=true       |
       |                          |                             | cache cmd                    |
       |                          |                             |                             |
       |                          |                             |<-----------------------------| detect trigger
       |                          |                             |<-----------------------------| ten_env_proxy.notify(...)
       |                          |                             | extension_on_notify()        |
       |                          |                             | return_result(detail=...,OK) |
       |<-------------------------|<----------------------------|                             |
       | assert OK + detail       |                             |                             |
       |                          |                             |                             |
```

---

## 6) 结论（可执行建议）

1. **看“模块数量”时，优先以 `BUILD.gn` 依赖组为准**，不要只看目录个数；
2. **理解 TEN 的关键在三件事**：生命周期回调、TenEnv 消息 API、Graph 路由；
3. **排查复杂问题时建议从 smoke test 入手**（尤其是 `start_graph + send_cmd + return_result` 这条链路）；
4. **跨线程场景务必使用 `ten_env_proxy->notify`**，避免线程上下文不一致导致的问题。

