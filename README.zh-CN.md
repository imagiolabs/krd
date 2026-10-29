# krd — 跑在终端里的远程桌面

[English](README.md) | 简体中文

[![CI](https://github.com/zzjcool/krd/actions/workflows/ci.yml/badge.svg)](https://github.com/zzjcool/krd/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/zzjcool/krd?sort=semver&label=release)](https://github.com/zzjcool/krd/releases)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](#license)
[![macOS](https://img.shields.io/badge/macOS-14%2B-black)](#环境要求)

**把一台 Mac 的桌面放进一个终端 pane 里。** ScreenCaptureKit 采集画面，编码成 kitty 图形协议的
转义序列写到 stdout；鼠标和键盘事件从 stdin 读回来，用 `CGEvent` 注入回桌面。

没有 VNC，没有编解码器，也不多开一个端口 —— 你手上这个终端本身就是客户端。

## 演示

<!-- 录制文件放在 docs/media/demo.gif；文件还没到位时下面的图片位置会显示 alt 文本，不会撑坏排版 -->
<p align="center">
  <img src="docs/media/demo.gif" alt="krd：SSH 终端 pane 里的一块 macOS 桌面" width="640">
</p>

<p align="center"><sub>一条 SSH 会话里的桌面。只有像素变了的瓦片会重发 —— 画面完全静止时一个字节都不发。</sub></p>

## 为什么

| | 传统远程桌面（VNC / RDP / 各家 App） | krd |
|---|---|---|
| 客户端 | 每台要看的机器上都得装一个 | 你已经在用的终端，任意 SSH 会话 |
| 网络 | 要监听端口，要端口转发或 VPN | 不用：转义序列走 stdout，按键走 stdin |
| 认证 | 自己一套密码 / TLS | 交给 SSH（密钥、2FA、跳板机、`ProxyJump` 全都能用） |
| 压缩 | 视频编解码器 | 只对真正变化的屏幕区域做 zlib |
| 输入 | 自己的协议 | SGR 鼠标上报 + 按键 → `CGEvent` |

具体一点：一次普通的界面变化（多一个字、按钮高亮、光标移动）只碰到 **1~2 块瓦片 = 1~5 KB**；
桌面静止时是 **0 字节/秒**，因为和上一帧逐字节相同的帧在进入编码器之前就被丢掉了。

## 环境要求

- 被看的那台 Mac 需要 **macOS 14 (Sonoma) 或更新版本**（`Package.swift` 里写死 `.macOS(.v14)`）。
- 看的那一侧需要一个实现了 **kitty 图形协议**的终端 —— 见[兼容矩阵](#终端兼容矩阵)。Apple
  Terminal.app 不行。
- 编译需要 Xcode command line tools。如果只部署二进制过去，目标 Mac 上不需要 Swift 工具链。
- 被看的那台 Mac 需要「屏幕录制」和「辅助功能」两项权限，一次性，见[部署到另一台 Mac](#部署到另一台-mac)。

## 安装

### 从源码编译

```bash
git clone https://github.com/zzjcool/krd && cd krd
swift build -c release                     # → .build/release/krd
.build/release/krd --selftest --no-input   # 合成画面，完全不需要任何权限
```

### 装成签名过的二进制（推荐）

```bash
./install.sh          # 编译 → 装到 ~/bin/krd → 用稳定身份重新签名
```

`install.sh` 把二进制拷到一个固定路径（`~/bin/krd`），用机器上找到的第一个
`Apple Development: …` 身份签名，找不到就退回 ad-hoc 并给出警告。签名不是为了好看：macOS 把
**屏幕录制授权记在代码签名身份上**，ad-hoc 签名每重编一次 cdhash 就变一次，授权随之失效
（`-3801 The user declined TCCs`）。一次签名，一次授权。要指定身份：
`KRD_SIGN_IDENTITY="Apple Development: …" ./install.sh`。

### 部署到另一台 Mac

```bash
./deploy.sh user@host              # → 那台机器的 ~/bin/krd
./deploy.sh user@host /usr/local/bin/krd
```

脚本编译 universal 二进制（arm64 + x86_64），在本机用同一个身份签名，再用 `cat | ssh`
流式拷过去，签名原样保留。完整步骤（含两次 TCC 授权）：[docs/DEPLOY.zh-CN.md](docs/DEPLOY.zh-CN.md)。

每个 [Release](https://github.com/zzjcool/krd/releases) 也附了现成的
`krd-vX.Y.Z-macos-universal.tar.gz` 和对应的 `.sha256`。那些二进制**没有**用你的身份签名，第一次跑之前
先本地签一下（`codesign --force --sign "…" --identifier com.kkrt.macdesk krd`），否则下一个版本一换，
TCC 授权就没了。

## 用法

```bash
krd --list                        # 列出显示器与窗口后退出
krd                               # 指针所在那块屏，20 fps
krd --display 1 --fps 30          # 指定显示器（序号来自 --list）
krd --window "Mail"               # 只镜像一个窗口（id 或标题子串）
krd --no-input                    # 只看不动
krd --debug                       # 统计 + 几何 + 输入轨迹，写入文件
```

走 SSH 的时候：

```bash
ssh -t user@host ~/bin/krd --fps 20 --debug
```

`-t` 不是可选项。stdin 不是 tty 时输入会被禁用（krd 会明确说明，并且不去打开鼠标上报，否则
终端的鼠标上报会当成文字糊在画面上），同时 pane 会退回 80×24 格，比例就错了。

**退出按 Ctrl-\\**。Ctrl-C 会原样转发给远端桌面，和在真机上按一样。

### 参数

下表每一行都对应 `Sources/krd/main.swift` 里的解析代码，默认值和取值范围按代码实际行为写。

| 参数 | 默认 | 说明 |
|---|---|---|
| `--fps <n>` | `20` | 采集帧率上限；最小 1，实际生效为 `minimumFrameInterval = 1/n` |
| `--display <i>` | 指针所在的屏 | `--list` 打印的显示器序号；越界时退回指针所在屏 → 主屏 → 第一个可用的 |
| `--window <q>` | — | 只采集一个窗口：窗口 id 数字，或标题的不区分大小写子串；候选窗口需大于 40×40 pt |
| `--tile <c>x<r>` | `16x8` | 脏瓦片尺寸，单位是终端单元格；`c` 最小 2，`r` 最小 1 |
| `--cell <w>x<h>` | `8x20` | 终端不汇报像素尺寸时，兜底的单元格像素尺寸 |
| `--log <path>` | `$TMPDIR/krd-<pid>.log` | 诊断日志写到这个文件（默认绝不写屏幕） |
| `--full-frame` | 关 | 关掉瓦片，每次都整帧重传（对比 / 排障用） |
| `--no-input` | 关 | 不打开 stdin，不注入任何事件 |
| `--dry-run` | 关 | 只解析输入并记日志，绝不发 `CGEvent`；建议配 `--debug` 或 `--log`，否则日志会落到 stderr 糊在画面上 |
| `--debug` | 关 | 文件日志 + 每秒统计 + 几何信息 + 输入轨迹 + 辅助功能权限状态 |
| `--selftest` | 关 | 用合成画面走完整的瓦片管线；不碰 ScreenCaptureKit，不需要任何权限 |
| `--list` | 关 | 打印显示器与窗口后退出（需要屏幕录制权限） |
| `-h`、`--help` | — | 打印用法后退出 |
| `-v`、`--version` | — | 打印版本号（`krdVersion`，与 `VERSION` 文件同步）后退出 |

`--tile 8x4` 是更小更碎的瓦片（每次更新更小、次数更多）；`--tile 32x16` 是更大更少的瓦片
（单次更新的协议开销更小、字节更多）。默认 16×8 就是下面实测数据用的配置。

### 环境变量

都是启动时读一次的诊断开关，逐个解释见 [docs/TESTING.zh-CN.md](docs/TESTING.zh-CN.md)。

| 变量 | 作用 |
|---|---|
| `KRD_DUMP=<path>` | 把 krd 发出的每一个字节存到文件（离线比对协议） |
| `KRD_GRID=1` | 仅 selftest：每块瓦片一个颜色 + 白边，验瓦片定位 |
| `KRD_QUADRANTS=1` | 仅 selftest：四种发图方式各画一个象限 |
| `KRD_Q=0` | 不再吞掉协议回复（能看到 `OK` / `EINVAL` / `ENOENT`） |
| `KRD_NOSYNC=1` | 去掉每块瓦片外面的 `?2026` 同步块 |
| `KRD_DELTEST=1` | 仅 selftest + quadrants：单独发一次 `a=d,i=901`，看它到底删掉了什么 |
| `KRD_NO_NSAPP=1` | 不创建 `NSApplication` 上下文（复现 ScreenCaptureKit 的一种失效） |
| `KRD_SIGN_IDENTITY` | `install.sh` / `deploy.sh` 使用的签名身份 |

## 工作原理

```
 你的机器                                  被看的 Mac
┌──────────────────────────┐              ┌──────────────────────────────────────────────┐
│ 终端 pane                │              │  sshd ── pty ── stdout ──▶ krd               │
│  rex / Ghostty / kitty   │              │                  stdin  ◀──                 │
│                          │              │                                              │
│  画 APC 图像      ◀──────┼──── ssh ─────┼── kitty 图形协议转义序列                     │
│  上报鼠标/键盘    ───────┼──── ssh ─────┼──▶ SGR 上报 ──▶ CGEvent ──▶ 桌面             │
└──────────────────────────┘              │                                              │
                                          │  ScreenCaptureKit ──▶ BGRA ──▶ RGBA ──▶ 瓦片 │
                                          └──────────────────────────────────────────────┘
```

1. **采集。** `SCStream`（整屏，或 `SCContentFilter` 指定的单个窗口）按 `--fps` 送 BGRA 帧，
   `vImagePermuteChannels_ARGB8888` 一趟转成 RGBA。和上一帧逐字节相同的帧在这里就被丢掉 —— 这就是
   桌面静止时零开销的原因。`queueDepth = 3`；流意外停止会由 delegate 记下来。
2. **布局。** 把采集区域的长宽比按**整格**贴进 pane（不做亚格偏移，不给终端留重采样的机会）：
   比如 137×73 格、单元格 17×33 px 的 pane 显示 1920×1200 pt 的屏，会得到 137×44 格、
   2329×1452 px 的画面，也就是 9×6 = 54 块 16×8 格的瓦片。
3. **脏瓦片。** 每帧逐瓦片 `memcmp`（每行像素一次 `memcmp`），只有变的瓦片才压缩：
   系统 `libz` 的 `compress2()`（RFC 1950，正是协议 `o=z` 要的东西）→ base64 → 按 4096 字符
   分块写给终端。
4. **渲染。** 每块瓦片一个 image id + 一个 placement id，都按瓦片序号写死，所以新一帧只是覆盖
   同一块瓦片的上一个 placement：**永远不发删除**，也就没有哪个终端能因为删除把画面搞没。
   完整序列见 [docs/PROTOCOL.zh-CN.md](docs/PROTOCOL.zh-CN.md)。
5. **输入。** 打开 `?1003h`（所有移动都要报）+ `?1002h`（按下/拖动/释放）+ `?1006h`（SGR 编码）+
   `?1016h`（**像素**坐标），按同一套布局反算回屏幕坐标，再发成 `CGEvent`。能不能点中 12 点的
   红绿灯按钮，全靠像素坐标；不按键也能让指针跟着动，全靠 any-motion 跟踪。

### 几何换算的三层

```
终端单元格  ──▶  图像像素  ──▶  屏幕点（CGEvent 坐标）
     ▲                  ▲
     │                  └─ TileGrid：整格贴，边缘瓦片更窄/更矮
     └─ CSI 16t / CSI 14t  →  TIOCGWINSZ  →  --cell（谁先答应用谁）
```

- `TermGeometry.query()` 直接问终端（`CSI 16 t` 问单元格尺寸、`CSI 14 t` 问窗口尺寸，250 ms 超时）。
  sshd 起的 pty 里 `ws_xpixel`/`ws_ypixel` 通常是 0，所以这是 SSH 场景下唯一能拿到真实像素尺寸的路子。
- 问不到就退回 `TIOCGWINSZ`（stdout → stdin → `/dev/tty`），最后退回 `--cell`（默认 `8x20`）。
- `TileGrid.make()` 用整格把采集区域的长宽比贴进 pane；`TileGrid.rect()` 对每块瓦片的像素矩形做
  四舍五入，保证瓦片正好铺满画面，边缘瓦片自然更窄/更矮。
- `Viewport.point(forCell:)` 把单元格映射到该格的**中心**；`Viewport.point(forPixel:)` 在 placement
  范围内做线性映射。落在 placement 之外的事件直接丢掉。

### 输入

- 鼠标：`?1003h`（每次移动都报）+ `?1002h`（按下 / 拖动 / 释放）+ `?1006h`（SGR 编码）+
  `?1016h`（像素坐标）。只合成左键：按下 → `leftMouseDown`，按着键移动 → `leftMouseDragged`，
  没按键的移动（`b & 3 == 3`）→ `mouseMoved`，结束符 `m` → `leftMouseUp`。滚轮（`b & 64`）变成
  按行的 `scrollWheelEvent`，上滚 +1、下滚 -1。
- **降级判定。** 单元格坐标不可能超过终端格数，所以任何 x 或 y 超过 `cols`/`rows` 的上报一定是像素；
  此时粘住一个 `pixelMouse` 标志。不认 `?1016h` 的终端继续按单元格发，就继续走单元格映射。
  两条路都对，只是像素坐标精确 17×33 倍。
- 键盘：CSI 方向键 / Home / End / `~` 系列（带修饰键位）、SS3 方向键与 F1–F4、
  Backspace/Tab/Enter、Ctrl-A … Ctrl-Z、可打印字符走 US/ANSI 虚拟键码表（需要 Shift 的符号补
  `maskShift`），其余字符走 Unicode 字符串按键事件。Alt+键 以 `ESC`+键 到达，映射成 `maskAlternate`。
- tty 会进入 raw 模式（关 `ECHO|ICANON|ISIG|IEXTEN`、关 `IXON|ICRNL|BRKINT|INPCK|ISTRIP`、关
  `OPOST`、`CS8`、`VMIN=1`、`VTIME=0`），退出时恢复，所以远端桌面能收到每一个键，也不会回显两次。

## 实测数据

把 1920×1200 pt 的屏镜像进 137×73 格的 pane（单元格 17×33 px → 2329×1452 px 画面，
9×6 = 54 块瓦片），取自 `--debug` 的 `stats:` 行：

| 场景 | tiles/s | 脏瓦片 | 最近一帧 | 每秒字节 | zlib |
|---|---|---|---|---|---|
| 静止 | 0 | 0 | 0 KB | 0.00 MB/s | — |
| 打字 / 滚屏 | 113 | 1 | 8 KB | 3.88 MB/s | 1 ms |
| 拖动窗口 | 157 | 43 | 2395 KB | 9.18 MB/s | 128 ms |

`tiles/s` 是每秒发出的瓦片数（不是帧数），`最近一帧` 是该帧压缩后的字节数，`每秒字节` 是最近一秒
写出的总字节数。

上瓦片之前（每次更新重传整帧），同一个 pane 是 430 KB/帧、12.6 MB/s、每帧 zlib 48 ms。瓦片把它
降了一个数量级，而且压缩成本只和真正变化的面积有关。

协议层的测试数据（终端管道吞吐、单块瓦片大小、端到端带宽预算）在原调研报告里：
[docs/research-terminal-remote-desktop.md](docs/research-terminal-remote-desktop.md)。

## 终端兼容矩阵

| 终端 | 结果 | 说明 |
|---|---|---|
| **rex** | ✅ | 端到端验证：画面、点击（含窗口红绿灯）、拖动、键盘 |
| **Ghostty** | ✅ | 画面与输入 |
| kitty、WezTerm | ✅ 预期可用 | 协议的标准实现，本项目没有逐个回归 |
| **Herdr 0.9.5** | ⚠️ | 它的 `a=d` 会清空整批图像；krd 从不对单块瓦片发删除，理论上可用，但不在回归名单里 |
| Apple **Terminal.app** | ❌ | 不支持 kitty 图形协议，会把转义序列当文本打出来 |

换一个没测过的终端时，用 [docs/TESTING.zh-CN.md](docs/TESTING.zh-CN.md) 里的标定模式（`--selftest`、
`KRD_GRID=1`、`KRD_QUADRANTS=1`）验一遍：不需要权限，也不需要第二台机器。

## 已知限制

- **没有 kitty 键盘协议**（`CSI > 9 u`）。裸修饰键（单独按住 Shift）、按键释放事件、Cmd 组合都拿不到；
  Alt+键 靠 ESC 前缀能用。
- **不保证非美式键盘布局。** 字符走 US/ANSI 虚拟键码表，表里没有的按 Unicode 字符串发，所以
  **中文/日文等输入法输入不支持**，死键也不支持。
- **鼠标左中右三个键都支持**，双击/三击也支持（`clickState` 按「半秒内同一位置」推导，和真机行为一致），
  终端在按钮码里带上的修饰键会一起转发（`& 4`/`& 8`/`& 16` → Shift / Option / Control）。
  指针移动同样转发（`?1003h` → `mouseMoved`），所以 hover 和指针跟随能用 —— 前提是终端支持 any-motion 跟踪。
  不支持跨应用拖拽、不支持力度/压感事件。
- **没有剪贴板**，没有拖拽传文件，没有文件传输。
- **窗口模式只作用于采集。** `--window` 只过滤*采集*，点击仍然按全局坐标注入，被别的窗口盖住的窗口
  收不到（`CGEventPostToPid` 在 Roadmap 里）。
- **显示器和窗口在启动时选定。** 窗口尺寸变化会处理（`SIGWINCH` → 重新测量、重排布局、重启流），
  但热插拔显示器、切换采集窗口需要重启。
- **只有绝对坐标指针。** 终端给的是绝对坐标，所以没有指针锁定、没有 FPS 式的相对位移 —— 这是终端
  形态的天花板，不是实现缺陷。
- **滚轮是离散的**（按钮 64/65 → 行单位），没有像素级增量，没有惯性。
- **没有声音**（`capturesAudio = false`），指针是合成进画面里的（`showsCursor = true`）。
- **锁屏是黑的。** 会话锁屏后 ScreenCaptureKit 不给像素，画面会冻住/变黑，直到解锁。
- 性能取决于链路：一次普通变化 1~5 KB，任何 SSH 会话都够；但全屏动态（视频、图片区平滑滚动）是每次
  更新 400 KB 以上，需要更好的编码器 —— 那是 Roadmap 里的事。

## 部署到另一台 Mac

完整步骤与所有失败模式：[docs/DEPLOY.zh-CN.md](docs/DEPLOY.zh-CN.md)。简版：

```bash
# 1. 目标 Mac：系统设置 → 通用 → 共享 → 远程登录：打开
# 2. 在你的机器上（同一份源码、同一个签名身份）：
./deploy.sh user@host              # universal 二进制 → 目标机的 ~/bin/krd

# 3. 目标 Mac 上，第一次要在它自己的图形界面终端里跑（不是 SSH 里）：
~/bin/krd --list                   # macOS 弹「屏幕录制」时点允许
~/bin/krd --fps 20                 # macOS 弹「辅助功能」时点允许
#    如果 macOS 压根没弹（有可能，因为进程是 SSH 拉起来的），就在
#    系统设置 → 隐私与安全性 → 屏幕录制 里用「+」手动添加这个二进制，辅助功能同样再加一次。

# 4. 之后在任何一台有 kitty 图形终端的机器上：
ssh -t user@host ~/bin/krd --fps 20 --debug
```

**最容易翻车的是 TCC。** macOS 把「屏幕录制」和「辅助功能」记在*代码签名*上，所以：

- 二进制必须用**稳定身份签名**，并且待在**稳定路径**上。ad-hoc 签名（`codesign -s -`）每重编一次
  cdhash 就变一次，下一次运行就像一个全新 App，授权没了 —— ScreenCaptureKit 报 `-3801`，或者画面
  全黑但屏幕上一条错误都看不到。
- `install.sh` / `deploy.sh` 故意使用同一个 identifier（`com.kkrt.macdesk`，见 `install.sh` 里的注释）：
  TCC 记录挂在它上面，改掉它等于强制重新授权一次。
- 不要 `strip`、不要重签、不要 `mv` 已安装的二进制；顺序永远是「重编 → 重签 → 再拷」，路径保持不变。
- `--list` 本身就需要屏幕录制权限（它通过 ScreenCaptureKit 枚举窗口），所以它是安全的第一跑：要么打印
  出你的显示器和窗口，要么告诉你授权没给 —— 都在碰到任何像素之前。

**按需启动，用完就走。** krd 由你在 SSH 会话里启动，按 Ctrl-\\ 或断开会话就结束。没有东西留在后台，
也不会自己启动；想要第二个会话，就再跑一次。

## Roadmap

1. **kitty 键盘协议**（`CSI > 9 u`）：裸修饰键、按键释放、Cmd 组合。
2. **`CGEventPostToPid`** 让窗口模式下的点击能落到被遮挡的窗口。
3. 剪贴板同步（OSC 52）、多显示器热切换。
4. 全屏动态场景的编码器分支（例如 VideoToolbox HEVC），开关后面；UI 场景仍然默认走瓦片。

## 文档

| 文档 | 内容 |
|---|---|
| [docs/ARCHITECTURE.zh-CN.md](docs/ARCHITECTURE.zh-CN.md) | 模块划分、数据流、线程模型、布局算法、必须守住的不变量 |
| [docs/PROTOCOL.zh-CN.md](docs/PROTOCOL.zh-CN.md) | krd 发出和解析的每一条转义序列，逐字节 |
| [docs/TROUBLESHOOTING.zh-CN.md](docs/TROUBLESHOOTING.zh-CN.md) | 这个项目踩过的 36 个坑，按「现象 → 根因 → 对策」整理 |
| [docs/DEPLOY.zh-CN.md](docs/DEPLOY.zh-CN.md) | 单二进制部署、TCC、签名稳定性 |
| [docs/TESTING.zh-CN.md](docs/TESTING.zh-CN.md) | 每个诊断开关到底在验什么 |
| [docs/research-terminal-remote-desktop.md](docs/research-terminal-remote-desktop.md) | 最初的可行性调研与测量 |

## 参与贡献

欢迎提 issue 和 PR。动手前请先读 [CONTRIBUTING.md](CONTRIBUTING.md)（英文）：里面有编译/自检命令、
「绝不往 pty 写文字」这条铁律，以及新功能的终端标定流程。

## 安全

krd 的设计就是把被看的那台 Mac 的完整控制权交给能运行它的人。它不开任何端口、也不做任何认证 ——
请放在 SSH 后面。详见 [SECURITY.md](SECURITY.md)（英文）。

## License

MIT，见 [LICENSE](LICENSE)。
