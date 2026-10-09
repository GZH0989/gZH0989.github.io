# 从零打造一个终端 AI 对话程序：TUI-aichat-terminal 开发记录

> 一个在 TTY 里跑的中文 AI 问答终端，纯键盘操作、无图形界面、最小 6 行 × 80 列。
> 本文记录从需求到落地的全过程，包括几次踩坑和修复。

---

## 一、项目背景

一直以来我都想要一个"足够轻"的 AI 对话工具：

- 不要浏览器，不要 Electron，不要 200 MB 的安装包
- 在服务器 SSH 里也能用
- 键盘操作，不要鼠标
- 对话要能存下来，下次能翻

市面上不是没有类似的东西，但要么是英文为主的 CLI 工具，要么依赖太多、要么界面太丑。于是我决定自己搓一个，主要为了三件事：

1. **终端里的聊天体验要顺**——焦点切换、流式输出、思考过程可视、输入光标。
2. **多 API 可热切换**——今天是 DeepSeek，明天可能是 Moonshot 或 OpenAI 兼容网关。
3. **零污染**——不装进系统 Python，整个文件夹能随便挪。

最终成果就是 **TUI-aichat-terminal**：一个单文件主程序，加上一个自动准备运行环境的一键安装脚本。

> ⚠️ 事先说明：本项目**全部代码由 DeepSeek 生成**，我只是负责整理、测试和发布。使用风险自负，详见仓库 README 的免责声明。

---

## 二、技术选型

选择很简单，因为可选范围就那么几种：

| 模块 | 选择 | 原因 |
| --- | --- | --- |
| UI | `prompt_toolkit` | 纯 Python，全键盘可控，跨平台 |
| HTTP | `aiohttp` | 原生异步流式，SSE 支持好 |
| 配置 | `tomllib` | Python 3.11+ 标准库，无需依赖 |
| 协议 | OpenAI 兼容 Chat Completions | 几乎所有国内厂商都兼容 |

为什么不用 `textual` 或 `rich`？——它们的定位更偏"应用框架"，对我要做的这种小工具来说太重了，而且我不需要那么多 widget。`prompt_toolkit` 提供的东西刚好够：一个布局、几个 Window、一套键绑定。

---

## 三、界面设计

### 三段式布局

```
┌─────────────────────────────────────────────┐
│                                             │
│              输出区（对话内容）             │
│              高度 = 窗口高度 - 3            │
│                                             │
├─────────────────────────────────────────────┤
│  >>> 输入框第 1 行                          │
│      输入框第 2 行                          │
├─────────────────────────────────────────────┤
│  API:model 思考:档位 [导航] [历史]  ↑↓ ...  │
└─────────────────────────────────────────────┘
```

输出区弹性，输入框固定 2 行，状态栏固定 1 行。用 `HSplit` 三兄弟，不复杂。

### 焦点循环

七个可聚焦的控件：

```
输出区 → 输入框 → API → Model → 思考 → 问题导航 → 历史对话 → 输出区
```

`↑` / `↓` 循环切换。**为什么不加 Tab？**——因为在终端里 Tab 经常被 shell 或终端本身吃掉，`↑↓` 最稳定。

### 视觉语言

这是花了最多心思的部分。因为要在一个只有字符的终端里表达"当前焦点""生效值""预览值"，光靠文字说不清。

最终定的规则：

| 视觉 | 含义 |
| --- | --- |
| `[x]` 方括号 | 光标位置（预览光标） |
| 蓝底高亮 | 光标位置 = 当前生效值 |
| 黄底高亮 | 光标位置 ≠ 生效值（预览中） |
| 亮色文字 | 生效值 |
| 左侧 `▌` 竖条 | 输出区获得焦点 |
| 反色方块 | 输入框光标 |

预览机制是这样的：在 API / Model / 思考强度上按 `←→`，只是**预览**新值，不生效。按 `Enter` 才确认。如果按 `↑↓` 离开焦点，预览被丢弃。这样避免了用户误切 API 导致的上下文丢失。

### 输入框光标

这一块最折腾。`prompt_toolkit` 自带一个光标，但它的位置计算在自定义 `FormattedTextControl` 里不太准，尤其在多行输入里容易越界抛异常。

最后决定**自己画**：把光标位置的字符在渲染时换成反色块。代价是光标移动时要重新计算行号和列号，好处是完全可控，不会崩。

---

## 四、主程序架构

主程序就一个文件 `ai_terminal.py`，大约 1000 行，结构很扁平：

```
路径与配置  →  工具函数  →  网络层  →  输入缓冲  →  全局状态
     ↓
对话存储  →  LLM 调用  →  输出渲染  →  输入渲染  →  状态栏渲染
     ↓
按键绑定  →  样式  →  布局  →  主程序
```

### 状态

所有可变状态都在一个 `State` 对象里，很土但很好用。`CONFIG` 是全局的，`Ctrl+R` 热重载时直接替换。

### 网络层

三个要点：

1. **大读取超时**（`sock_read_timeout=600s`）：思考模型可能几分钟不发数据，默认超时会误判。
2. **TCP keepalive**：物理断开能快速感知，不用等读取超时。
3. **SSE 注释行过滤**：`:` 开头的行是服务端心跳，必须跳过。

payload 里带上 `stream_options: {"include_usage": True}`，让 API 在流末尾上报 token 用量。状态栏右侧就能实时显示 `↑12.3k / ↓4.5k / ∑16.8k` 这样的数字。

### 思考强度

四档：`none` / `low` / `high` / `max`，显示为「关/低/高/最高」。

这里有个**关键坑**：`reasoning_effort` 参数**必须始终发送**，值设为 `"none"` 才是真正的关闭。如果按"关的时候不发送这个字段"处理，某些 API 会默认走它自己的推理模式，反而关不掉。

### 对话存储

每个对话是一个 JSON 文件：

```
chat/2026-10-07_14-30-00.json
```

结构：

```json
{
  "created": "2026-10-07T14:30:00",
  "updated": "2026-10-07T14:35:12",
  "api": "deepseek",
  "model": "deepseek-reasoner",
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "...", "reasoning_content": "..."}
  ]
}
```

写入用**临时文件 + rename** 保证原子性——万一写到一半断电，至少不会生成半个 JSON 文件。

启动时自动清理所有 `messages` 为空的对话文件（保留当前那个）。退出时也删一次。这样用户误按 `Ctrl+C` 启动一次不会留下一堆空文件。

### 控制字符过滤

模型偶尔会吐 ANSI 转义序列，比如 `\x1b[31m`。如果直接输出到终端，会改变颜色甚至清屏。所以所有内容（包括本地读回的历史）都过一遍：

```python
_CTRL_RE = re.compile(r'[\x00-\x08\x0b-\x1f\x7f]')
def strip_control_chars(s):
    return _CTRL_RE.sub('', s)
```

保留 `\t`（0x09）和 `\n`（0x0a），其他控制字符全清掉。

---

## 五、安装脚本：不复用系统 Python

这部分是后期改设计最多的。

### 最初的想法

一开始想得很简单：检查系统 Python 有没有 `aiohttp` 和 `prompt_toolkit`，有就用，没有就 `pip install`。

### 现实的打击

用户在自己的 Windows 电脑上跑，系统 Python 是 **3.14.0**，然后 `pip install aiohttp` 直接报错：

```
ERROR: Could not find a version that satisfies the requirement aiohttp
(from versions: none)
ERROR: No matching distribution found for aiohttp
```

原因很清楚：**Python 3.14 刚发布不久，`aiohttp` 及其 C 扩展依赖（`multidict`、`yarl`、`frozenlist`）的 `cp314` 预编译 wheel 尚未发布。** pip 只能尝试源码编译，而用户机器上没有 C 编译工具链，于是只能失败。

这是 Python 生态里一个很常见的窗口期问题。**大版本刚发布后的几个月，很多 C 扩展库都还没跟上。**

### 方案演化

经过几轮讨论，最终方案是"**双轨制**"：

```
系统 Python >= 3.11
├─ 环境健康测试（试装 wcwidth）
│   ├─ 通过 → 用系统 Python 建 venv（约 30 MB）
│   └─ 失败 → 下载独立 Python 3.12.7 运行时（约 150 MB）
└─ 用户可随时主动选择下载独立环境

系统 Python < 3.11
└─ 直接询问是否下载独立环境
```

**为什么不干脆放弃系统 Python，直接一律下载独立运行时？**

因为不是所有用户都想多占 150 MB。如果用户的系统 Python 干净、版本合适，用 venv 只占 30 MB，体验更好。所以脚本默认"能用系统就用系统"，只在必要时才下载。

### 环境健康测试

核心是一个简单的试装：

1. 在临时目录里 `pip install wcwidth --target=...`
2. 用子进程把那个目录加进 `sys.path` 再 `import wcwidth`
3. 检查退出码
4. 无论成败，删掉临时目录

`wcwidth` 是 `prompt_toolkit` 的依赖，纯 Python、很小、很快。它同时验证了：pip 能连源、能下载、能写盘、装出来的东西能 import。

**为什么这一步能识别沙箱版 Python？**——Microsoft Store 版和 pythoncore 精简版的 `site-packages` 被重定向到沙箱位置，pip 装完 import 会失败，或者根本装不进去。路径检测不靠谱，但行为测试很准。

### 关于沙箱 Python 的坑

用户遇到过一个诡异的现象：**双击 `test.py` 时下载 aiohttp 失败，但在 cmd 里跑 `python test.py` 就成功。**

一开始以为是网络问题，后来诊断脚本一比，真相大白：

| 启动方式 | `sys.executable` |
| --- | --- |
| 双击 `.py` | `...\AppData\Local\Python\pythoncore-3.14-64\python.exe` |
| cmd 里 `python` | `...\AppData\Local\Programs\Python\Python314\python.exe` |

**Windows 把 `.py` 文件关联到了一个 Store 版的 pythoncore**，那个版本的 pip 索引里认不出 `cp314-cp314` 的 tag，所以找不到任何 aiohttp 版本。而 cmd 里用的 PATH 指向的是用户手动安装的官方版 Python，索引正常。

这个坑最终得出一条重要结论：**永远不要双击 `.py`，永远用 `.bat` 启动器。** 启动器里用绝对路径调用项目自己的 Python，绕开文件关联和 PATH。

### pip 源策略

最初想的是"先默认源，失败切清华"。后来发现清华镜像偶尔会滞后于 PyPI 主站（特别是新版本的 wheel），所以这个回退策略还是有用的：

```python
# 第一次：不指定 index-url
# 第二次：加 -i 清华源 --trusted-host
```

### 下载 Python 的 URL 编码

python-build-standalone 的文件名格式是：

```
cpython-3.12.7+20241016-x86_64-pc-windows-msvc-install_only.tar.gz
```

注意那个 `+`。某些镜像站在处理 URL 时会把 `+` 当成空格，所以下载前必须替换：

```python
filename_encoded = filename.replace("+", "%2B")
```

### 一个奇怪的 `%` bug

`ai-term.bat` 里有一行：

```bat
set "DIR=%~dp0"
```

安装脚本用 Python 的 `%` 格式化来生成这个字符串时，`~d` 被当成了格式化符，直接抛 `ValueError: unsupported format character '~'`。

**教训：生成启动器脚本时，不要用 `%` 格式化**。用 `+` 字符串拼接或 f-string（注意 f-string 也要转义花括号）。

---

## 六、踩过的坑（汇总）

按时间顺序列一下，都是实际调试出来的：

1. `Dimension(exact=2)` 不是构造参数，得用 `Dimension.exact(2)`。
2. `get_cursor_position` 是 `FormattedTextControl` 的方法，不是 `Window` 的。
3. 多行渲染时，每行末尾必须补 `\n`，否则所有内容挤到一行。
4. 输入框光标自己渲染反色字符块，别用 prompt_toolkit 原生光标——会崩。
5. API key 支持 `api_key` 字段和 `api_key_env` 环境变量双通道，前者优先。
6. `reasoning_effort` 必须始终发送，`"none"` 才是真关。
7. 所有外部文本（包括本地历史）都要过滤 ANSI 转义。
8. 不要用 `%` 格式化生成 `.bat` 文件。
9. Python 大版本刚发布时，C 扩展库的 wheel 会滞后，得准备降级方案。
10. `.py` 文件关联到 Store 版 Python 是 Windows 上非常常见的一个坑。

---

## 七、安装脚本的代码嵌入技巧

`一键安装修复脚本.py` 是一个**自包含**的文件：它既要有安装逻辑，又要把主程序和 README 嵌进去。

用生成器 `fill.py` 处理：

```
src/ai_terminal.py  ─┐
src/README.md       ─┴─→ fill.py ─→ 一键安装修复脚本.py
installer_template.py
```

`installer_template.py` 里有两个占位符：

```python
AI_TERMINAL_PY = r'''@@AI_TERMINAL_PY@@'''
README_MD = r'''@@README_MD@@'''
```

`fill.py` 读源文件，用 `text.replace()` 填进去，生成最终脚本。

**几个要点：**

- **不要用 `.format()` 或 f-string**：主程序里有很多 `{`、`}`、`%`，会炸。
- **检查源文件里没有三单引号**：`ai_terminal.py` 里的 `DEFAULT_CONFIG_TOML` 用 `r"""..."""`，如果它用 `r'''...'''`，嵌入时字符串会提前结束。
- **检查文件末尾没有孤立反斜杠**：raw string 不能以反斜杠结尾。

填完后用 `compile()` 做一次语法校验，能过再写盘。

---

## 八、v0.1.0 的状态

**已实现：**

- 7 焦点 TUI，纯键盘操作
- 流式思考与回答显示
- 思考过程折叠/展开
- 思考强度四档热切换
- 多 API 热切换
- 历史对话浏览
- Token 用量显示
- 双轨安装（venv / runtime）
- 环境健康测试

**已知但未处理：**

- Windows 上首次启动完整流程未系统验证（本地已能跑）
- 部分终端下 `Ctrl+Enter` 被识别为 `Enter`（终端限制，非程序问题）
- free-threaded Python 版本（3.13t）未专门适配

**明确不做：**

- 联网搜索控件（DeepSeek API 不支持，删除后焦点从 8 变 7）
- 英文 README（界面本身全中文，不强求）
- GitHub Actions 自动构建（手动上传 Release 附件更省事）
- `--init-config` 参数（主程序首次启动自动生成 config.toml）

---

## 九、经验总结

写这个项目的过程里，最有价值的几条经验：

**1. 终端 UI 里，视觉语言比功能更重要。**

在字符界面里表达状态（焦点、生效值、预览值），光靠文字堆是堆不出来的。必须先定一套配色和符号规范，再往上做功能。

**2. "兼容所有环境"是不可能的，但"能让用户看到问题出在哪"是可能的。**

我不可能让程序在所有 Python 版本、所有操作系统、所有终端上都跑起来。但可以做的是：**检测环境 → 打印诊断 → 给出降级选项**。用户至少知道为什么失败了，以及怎么绕过。

**3. 不要相信文件关联。**

Windows 上 `.py` 文件关联到哪个 Python 完全是未知的，可能指向 Store 版、可能指向用户十年前装的 2.7。永远用绝对路径调用自己装好的解释器。

**4. 生成物和源码要分离。**

`一键安装修复脚本.py` 是生成物，不该手动改。`src/` 下才是真正的源码。这样升级或修复时，只改源码重跑 `fill.py` 就行，不用担心生成物里混进了什么手改的东西。

**5. 大版本刚发布时不要追新。**

Python 3.14 现在很香，但它的 C 扩展生态还没跟上。独立运行时选 3.12.7 是有意为之——这个版本的 wheel 生态最成熟，所有常用库都有现成的预编译包。

---

## 十、致谢

- [prompt_toolkit](https://github.com/prompt-toolkit/python-prompt-toolkit)
- [aiohttp](https://github.com/aio-libs/aiohttp)
- [python-build-standalone](https://github.com/astral-sh/python-build-standalone)
- [DeepSeek](https://www.deepseek.com/) — 本项目全部代码与文件（包括本篇blog）由 DeepSeek 生成

项目仓库：[GZH0989/TUI-aichat-terminal](https://github.com/GZH0989/TUI-aichat-terminal)
