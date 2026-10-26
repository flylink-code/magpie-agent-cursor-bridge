# Cursor · Magpie Bridge

让 Cursor 编辑器直接使用 Magpie 中配置的全部模型与路由组。模型菜单返回 Magpie 模型目录，隐藏 Cursor 默认 Auto 入口，保留 Cursor 原有登录。

当前版本全面支持 **Windows (x64)** 与 **Apple Silicon / Intel macOS**。

---

## 特性

- **全平台支持**：针对 Windows 与 macOS 深度适配原生协议桥接核心，开箱即用。
- **动态模型同步**：在 Magpie 中添加、切换或分组的模型，每 15 秒自动热同步至 Cursor 模型列表。
- **无感代理接管**：连接时自动配置 Cursor 代理，断开连接或卸载时完整自动还原 Cursor 原有配置。
- **安全与隐私**：所有 TLS 证书与回环代理均在本机生成，网络流量仅流经本机回环地址（`127.0.0.1`）至 Magpie 网关，不接触任何外部中转服务。

---

### 安装方式

在 Magpie → **Plugins** 添加仓库地址安装：
```text
github:flylink-code/magpie-agent-cursor-bridge
```

---

## 安装与快速上手

### 前置条件
- 已在本机安装 **Cursor** 与 **Magpie**。
- 已在 Magpie 中添加并配置好至少一个可用模型或路由组。

---

### Windows 系统使用步骤

1. **运行初始化**：
   在插件目录下打开 PowerShell 执行初始化脚本（或直接双击运行 `setup.cmd`）：
   ```powershell
   powershell -ExecutionPolicy Bypass -File .\setup.ps1
   ```
   *注：首次初始化时会检查并提示信任本机生成的本地根证书（系统标准确认窗口选择“是”即可）。*

2. **在 Magpie 中选择模型**：
   - 打开 Magpie，在 **Agents** 页面找到 **Cursor · Magpie Bridge**。
   - 打开开关，并从下拉列表中挑选您希望在 Cursor 中使用的模型或路由组。

3. **在 Cursor 中刷新模型列表**：
   - 首次开启后，后台桥接需要几十秒启动；等状态变为已连接即可。
   - 打开对话面板，或点一下底部的模型下拉框——Cursor 会重新拉取模型表，Magpie 提供的模型随即出现，点击即可畅快对话。

   > 无需重启 Cursor。模型目录是按请求下发的，只要 Cursor 重新发起一次拉取就会更新；
   > 若列表没变，再点一次模型下拉框即可。重启也只是强制刷新的一种方式。

---

### macOS 系统使用步骤

1. **运行初始化**：
   在终端中运行：
   ```bash
   bash setup.command
   ```
   *根据系统提示完成本地钥匙串证书信任授权。*

2. **在 Magpie 中选择模型**：
   - 在 Magpie → **Agents** 中开启 **Cursor · Magpie Bridge** 并选择模型。
   - 等后台桥接启动完成（几十秒），在 Cursor 中打开对话面板或点一下模型下拉框即可看到 Magpie 的模型；无需重启 Cursor。

---

## 常用管理命令

| 操作 | Windows (PowerShell) | macOS / Linux (Bash) |
| :--- | :--- | :--- |
| **启动/更新服务** | `powershell -ExecutionPolicy Bypass -File setup.ps1` | `bash setup.command` |
| **查看后台运行状态** | `powershell -ExecutionPolicy Bypass -File setup.ps1 status` | `bash setup.command status` |
| **停用并卸载后台服务** | `powershell -ExecutionPolicy Bypass -File setup.ps1 uninstall` | `bash setup.command uninstall` |

- **断开连接**：在 Magpie 的 Agents 页面选择 Disconnect，或在终端执行 `magpie cursor-bridge default`。
- **代理自愈与还原**：当桥接服务退出或在 Magpie 中断开连接时，Cursor 的代理配置会自动恢复为初始状态。

---

## 架构与工作流程

```text
Magpie Agent 插件 → 写入私有连接配置 (connection.json)
                                ↓
                      后台监视器 (Supervisor)
                      ├─ 启停原生桥接引擎 (cursor-magpie-bridge)
                      └─ 自动维护/还原 Cursor 代理设置 (settings.json)
                                ↓
Cursor 编辑器 ──[本地 TLS 回环代理]──> 原生桥接服务 ──[OpenAI 协议兼容]──> 本地 Magpie 网关 ──> 大模型提供商
```

- **配置与数据隔离目录**：
  - Windows: `%USERPROFILE%\.config\magpie-cursor-bridge\`
  - macOS: `~/.config/magpie-cursor-bridge/`
- **运行产物与自启**：
  - Windows: 自启通过当前用户 Startup 目录无窗口后台执行。
  - macOS: 通过 `launchd` 用户 LaunchAgent 守护。

---

## 原生协议回归测试

Windows：`powershell -ExecutionPolicy Bypass -File scripts\test-native-wire.ps1`。
脚本在插件内获取固定版本源码、应用 `source.patch`，运行无需 TLS、证书或真实设置的解码器测试及原生库测试。
源码保留在已忽略的 `.native-source/`，编译产物位于 `.native-target/`。完整构建并更新 Windows 二进制与 SHA-256：`powershell -ExecutionPolicy Bypass -File scripts\build-native.ps1`（需要 Rust/MSVC 工具链）。

### 0.2.8 Windows 兼容性测试更新

已重新构建仓库内 Windows 引擎并更新 `integrity.json`。补丁支持 hex / `data_binary` 两种 BidiAppend 编码，修正 Windows 的 Unix 信号引用，并在 protobuf 解码失败时记录字段编号、wire 类型和长度（不记录字段值）。macOS 二进制保持原样，不包含这些原生变更。插件版本为 0.2.8，原生引擎版本标识仍为 0.2.1，由 SHA-256 区分构建。

Magpie 从 GitHub 仓库默认分支安装文件，不需要 GitHub Release。在 **Plugins** 更新 `github:flylink-code/magpie-agent-cursor-bridge`；若原插件来自 `932633145`，先关闭 Agents 开关，再将插件来源替换为上述 fork。确认版本为 **0.2.8**，在更新后的插件目录运行 `powershell -ExecutionPolicy Bypass -File .\setup.ps1`，再开启 Agents 开关。此操作由用户执行，构建测试不会修改已安装配置或证书。重开 Cursor 对话并发送新消息。0.2.8 的 tracing 诊断未接入实际输出，Supervisor 也会丢弃引擎 stderr，因此无法从桥接日志取得 `wire_structure`。当前源码改为在解码错误响应中返回有界的 `wire structure`（仅字段编号、类型、长度和偏移，不含字段值）；重新构建安装后，可从 Cursor Agent Host 日志取得这些诊断。此源码改动尚未作为新版本发布。
Cursor 3.27.14 的 `conversation_state` wire type 6 原始故障尚未定位：合成合法状态可正常解码，合成非法状态可复现相同错误路径，但日志没有请求字节，不能据此断言字段变更或压缩问题。二进制编码支持不是该故障已修复的证据。

### 0.2.9 Windows：MCP Schema 整数转换修正

已确认 sub2api GPT 可因工具定义的 `minLength: 1.0` 被拒绝：Protobuf `Value` 用 double 表示所有数字，原转换保留浮点形式。当前源码在 MCP 输入转换边界将安全整数范围内的整值恢复为 JSON 整数，保留非整数及超出安全整数范围的浮点值。原生回归测试覆盖整数、零、负数与小数；本机已验证此前失败的 `sub2api/gpt-5.6-luna` 请求完成，HTTP 200、14 个输出 token、15 个流事件且无 provider error。0.2.9 包含更新后的 Windows 引擎和完整性校验；macOS 引擎未重新构建。

## 多供应商兼容边界

桥接通过 Magpie 网关使用统一协议，不按 DeepSeek、GPT、Kimi、GLM 或 Kiro 品牌硬编码转换。供应商名称不能证明兼容：仍需验证网关的协议转换、流式输出、工具调用及模型能力。

- 已实际验证：DeepSeek、sub2api GPT Luna/Terra 的基础对话。
- Antigravity 当前返回 HTTP 403 `permission_error`，说明账号所在地区不符合 Gemini Code Assist 使用资格。这属于上游授权拒绝，不应删改工具 Schema、盲目重试或静默切换模型来掩盖。需先在供应商支持的地区和账号条件下恢复授权，或选择其他获授权来源。
- Kiro 反代、Kimi、GLM 尚未实际联调；不承诺所有模型支持工具、图片、推理参数或同样的上下文长度。
- Schema 回归覆盖嵌套 `anyOf/properties`、整数长度/数量约束、小数 `minimum/multipleOf`、安全整数边界及非有限数处理。不按供应商删除约束，也不把所有小数强制取整。
- 新来源验收顺序：普通流式对话 → 多轮 → 工具调用及结果回传 → 该模型声明支持的图片/推理能力；分别检查授权错误（401/403）、限流（429）、参数错误（400）和上游故障（5xx），不可将所有失败都归因于桥接。

## 开源协议

本项目核心遵循 [MIT License](LICENSE)。
部分原生协议核心来源于开源项目 cursor-byok（MIT License）。
