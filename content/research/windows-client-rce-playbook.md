---
title: "Windows 客户端 RCE 挖掘方法全景手册"
date: 2026-08-17
tags: ["windows", "rce", "fuzzing", "vulnerability-research", "binary-security"]
summary: "从攻击面测绘、偏门入口（剪贴板/OLE/Deep Link/文件语义）、组件信任边界（Electron/WebView2/COM/RPC），到 Fuzzing、崩溃研判与无破坏 PoC 的完整方法论，覆盖 Windows 桌面客户端 RCE 挖掘的全流程。"
toc: true
draft: false
---

> **适用范围：** 获得授权的安全研究、企业内审、漏洞赏金与隔离实验环境。

> 本文聚焦发现、定位、风险判断与最小化验证，不包含武器化、持久化、业务破坏或未经授权的测试流程。

## 内容提要

Windows 桌面客户端 RCE
的核心，不是“程序是否开放端口”，而是攻击者可控内容能否经由文件、网页、URI、服务端响应、更新包、剪贴板、IPC
或缓存，最终进入进程启动、模块加载、任意文件写入、危险对象构造或可控内存破坏。

本手册将常见和偏门攻击面统一到同一套研究框架中，重点关注低交互入口、跨权限
Broker、Web/Native 桥、更新链、Windows
路径与文件系统语义、自动预览、二次触发状态以及多个中低危原语形成的组合链。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>攻击者可控输入<br />
↓<br />
解码 / 规范化 / 解析 / 反序列化 / 状态恢复<br />
↓<br />
跨越 Web—Native、低权—高权、用户目录—程序目录或沙箱—Broker 边界<br />
↓<br />
进程启动 / 模块加载 / 任意文件写 / 危险对象构造 / 可控内存破坏<br />
↓<br />
目标进程权限下代码执行</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>使用边界</strong></p>
<ul>
<li><p>仅在明确授权资产、企业内审环境、漏洞赏金范围或隔离实验室中开展研究。</p></li>
<li><p>验证以证明边界与影响为目的，不修改、删除、覆盖业务数据，不植入持久化。</p></li>
<li><p>对内存破坏无需强行完成武器化；稳定可控写、对象重占位、间接调用污染等证据已足以支持高质量研判。</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 第一部分　研究框架与优先级

### 1. 适用范围、RCE 定义与研究边界

本文讨论 Win32/C/C++、.NET、Qt、Electron、CEF、WebView2、UWP/WinUI
辅助组件、Shell 扩展、更新器和后台服务等 Windows
客户端。内核、驱动与浏览器内核本身不是主体，但当客户端把这些组件纳入自动解析链时，仍应纳入风险图。

- 远程内容型
  RCE：攻击者控制服务器返回值、消息、网页、局域网广播或同步内容，客户端自动或低交互处理后执行代码。

- 文件型
  RCE：恶意文件、项目、导入包、主题、模板或附件被打开、预览、索引、缩略图化或恢复时执行代码。

- 浏览器到本机型 RCE：网页经 Deep Link、localhost、WebSocket、Native
  Messaging 或外部协议调用本地客户端危险能力。

- 跨权限执行：普通用户或低完整性渲染进程通过 IPC/Broker 调用管理员或
  SYSTEM 组件；在报告中通常同时涉及 RCE 与本地提权。

- 仅本地、需先拥有目标目录写权限、只造成空指针崩溃的现象，通常不能单独称为高质量
  RCE。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>结论用词建议</strong></p>
<p>在证据不足时使用“可形成代码执行原语”“高概率可利用的内存破坏”“可跨越预期信任边界”，不要仅凭一次异常访问直接宣称稳定
RCE。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 2. 统一攻击链模型与危险原语

每个入口都应沿 Source—Transform—Boundary—Sink 四层追踪。不要只搜索危险
API，也不要只观察崩溃点；真正的根因往往出现在第一次错误规范化、第一次错误信任或第一次对象生命周期破坏。

| **层级** | **需要回答的问题** | **典型对象** |
|:--:|----|----|
| Source | 谁控制输入？远程、网页、低权限用户还是本地文件？ | 文件、URI、IPC、服务端响应、剪贴板、缓存 |
| Transform | 经历了几次解码、参数拆分、路径规范化、解压或反序列化？ | URL 解码、命令行解析、JSON/二进制解析、对象恢复 |
| Boundary | 在哪一步跨越了权限、进程、Origin、沙箱或文件系统边界？ | Renderer→Main、用户→SYSTEM、下载目录→插件目录 |
| Sink | 最终获得了什么可用原语？ | CreateProcess、LoadLibrary、任意写、危险对象构造、函数指针污染 |

高价值危险原语通常包括：

- 攻击者可控字符串进入进程创建、Shell、脚本解释器、外部工具或插件安装接口。

- 任意路径文件写入，且可落入可靠加载点、启动点、脚本目录、Manifest、插件目录或高权限消费目录。

- 不可信网页、iframe 或本地 HTML 获得 Native Bridge、Host Object、IPC
  或通用文件/进程能力。

- 普通用户可调用高权限服务的通用文件、下载、解压、诊断、安装、回滚或启动功能。

- 反序列化、XAML/QML 对象构造、动态程序集或脚本加载。

- 内存破坏中的可控写、UAF 可重占位、虚表/函数指针/间接调用目标污染。

### 3. 高价值目标的优先级与评分

建议先扫逻辑型与跨边界漏洞，再投入复杂内存利用。原因是逻辑型 RCE
通常更稳定、根因更清晰、复现成本更低，且更容易证明真实影响。

| **优先级** | **方向** | **原因** |
|:--:|----|----|
| S | Web/Native Bridge、高权限 Broker、更新器写入、自动预览、远程响应自动解析 | 低交互、跨边界、稳定且影响明确 |
| A | Deep Link 参数链、不安全反序列化、项目自动执行、DLL/插件可靠加载 | 通常可直接或通过一段短链形成执行 |
| B | 任意路径写但无加载点、XSS 无 Bridge、可控越界读、有限 IPC 越权 | 适合作为组合链零件 |
| C | 空指针、单纯 DoS、只能在调试版触发、需先有同等写权限 | 单独价值有限 |

可以使用以下内部评分模型进行队列排序：

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>总分 = 远程可控性(0–3) + 低交互程度(0–3) + 权限收益(0–3)<br />
+ 稳定性(0–2) + 可组合性(0–2) − 强前置条件(0–3)</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- 第一优先：Electron/WebView2/CEF/Qt Bridge、自动更新与高权限服务 IPC。

- 第二优先：Preview/Thumbnail/Property/IFilter、消息附件自动解析、局域网自动发现。

- 第三优先：项目/工作区自动任务、压缩导入链、插件与 DLL 加载。

- 第四优先：自定义原生解析器的函数级 Fuzz 与 Patch Diff。

### 4. 进程、权限、输入与信任边界测绘

正式审计前，先建立“进程图”和“输入矩阵”。这一步常常比直接阅读数百万行反编译代码更能缩小范围。

- 进程：主
  GUI、Renderer、GPU、插件宿主、更新器、崩溃处理器、后台服务、Shell
  扩展、预览宿主、本地 Web 服务。

- 权限：用户、管理员、SYSTEM；完整性级别；AppContainer/LowBox；令牌、Session、桌面与
  UIAccess。

- 接口：Named Pipe、RPC、COM、localhost、WebSocket、共享内存、Windows
  Message、临时文件、注册表和自定义 Socket。

- 加载：DLL、插件、Manifest、QML、脚本、语言包、主题、Codec、字体、COM
  类和子进程。

- 落盘：下载、缓存、更新、会话恢复、崩溃恢复、日志、诊断包、离线消息和项目状态。

| **字段** | **建议记录** |
|:--:|----|
| 输入来源 | 文件、文件名、URI、网页、服务端、剪贴板、IPC、局域网、缓存、更新包 |
| 触发条件 | 客户端关闭/运行/后台；是否登录；是否点击；是否自动恢复 |
| 处理进程 | 进程名、父进程、命令行、当前目录、完整性级别、沙箱 |
| 数据流 | 原始值、每次解码/规范化结果、最终 API 参数或文件句柄真实路径 |
| 危险结果 | 子进程、模块加载、文件写入、对象构造、内存异常、权限跨越 |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>基线建议</strong></p>
<p>对每个入口至少执行一次合法输入和一次异常输入，比较文件、注册表、模块、子进程、网络、IPC
和 COM 激活差异。行为差分经常能直接暴露隐藏解析器和二次加载点。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 第二部分　外部输入与被动触发面

### 5. Explorer、Shell 扩展与搜索索引

这类入口的价值在于：用户可能只需浏览目录、切换视图、选中文件、查看属性或让系统后台索引，客户端组件就开始解析。

- Preview Handler、Thumbnail Handler、Property Handler、Icon
  Handler、Overlay Handler。

- Context Menu Handler、Drop Handler、Copy Hook、Namespace Extension。

- Windows Search IFilter、搜索协议处理器、文件属性写回和元数据枚举。

- 文件关联、默认打开程序、右键命令、ShellExecute、预览窗格与详细信息窗格。

建议分别测试以下触发动作：

- 仅把文件放入目录；打开目录；切换大图标/缩略图；启用预览窗格。

- 查看属性；按作者、尺寸、时长等元数据字段排序；使用 Windows Search
  搜索。

- 鼠标悬停、单击选中、右键菜单、拖放、复制、移动和重命名。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>小众但高价值</strong></p>
<ul>
<li><p>同一种格式可能被主程序、Explorer 缩略图、Property
Handler、IFilter、预览器和云盘客户端分别实现解析；对同一样本做多解析器差分测试。</p></li>
<li><p>Property Handler
不仅读取，也可能写回元数据；写回路径、临时文件和对象生命周期是独立攻击面。</p></li>
<li><p>延迟触发需要覆盖
SearchIndexer、SearchProtocolHost、prevhost、dllhost
等不同宿主及其权限。</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 6. 剪贴板、OLE、虚拟文件与拖放

Windows 剪贴板和拖放通过 IDataObject/STGMEDIUM
传递多种格式。同一个对象可同时声明文本、HTML、RTF、位图、文件、流和嵌入对象，而客户端可能对不同阶段使用不同解析器。

- CF_HDROP、CF_UNICODETEXT、HTML Clipboard Format、RTF、DIB/PNG。

- FileGroupDescriptor/FileContents 虚拟文件、Outlook
  附件拖放、延迟渲染数据。

- Embedded Object、Link Source、OLE 嵌入/链接对象、自定义注册 Clipboard
  Format。

- IDataObject::GetData 返回的
  TYMED、IStream、HGLOBAL、文件句柄与生命周期管理。

偏门测试思路：

- 同一对象声明多种互相矛盾的格式，观察客户端优先级和回退路径。

- 文件描述符中的名称、长度、扩展名与真实流不一致。

- DragEnter、DragOver、Drop、Paste
  各阶段分别变异，并在处理中关闭窗口或取消操作。

- 延迟渲染期间数据源退出、断开或返回不同尺寸，检查
  UAF、双重释放和状态错乱。

- 剪贴板内容先由远程桌面、浏览器或聊天软件产生，再被高权限客户端粘贴消费。

### 7. URI、Deep Link、命令行与单实例转发

Deep Link 常形成“网页 → Windows 协议激活 → 第二实例 → 单实例 IPC →
第一实例内部路由 →
文件/插件/子进程”的多层解析链。每一层可能使用不同编码和引用规则。

- 注册位置：HKCU/HKLM\Software\Classes、HKCR、URL
  Protocol、shell\open\command、App Paths。

- 测试
  Query、Path、Fragment、重复参数、大小写、空值、超长值、Unicode、换行、引号、反斜杠和嵌套
  URL。

- 比较客户端未运行、已运行、初始化中、登录前、管理员实例和多用户 Session
  下的行为。

- 确认参数经过 URL 解码、Windows 命令行拆分、IPC
  序列化和内部路由后是否仍保持预期边界。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>参数记录器</strong></p>
<p>在实验室中使用无害的参数记录程序，记录最终可执行路径、argc/argv、工作目录、环境变量、父进程、完整性级别和继承句柄。比直接尝试命令更安全，也更能证明参数边界被突破。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- 重点
  Sink：CreateProcess、ShellExecute、Process.Start、QProcess、child_process、脚本解释器、压缩工具、编译器和插件安装器。

- 重点差异：程序路径是否完整；程序与参数是否分离；工作目录是否可控；是否使用相对
  Helper；是否继承 PATH/PATHEXT/TEMP。

- 小众点：@response-file、扩展名省略、SearchPath、PATHEXT、当前驱动器相对路径
  C:folder、引号前反斜杠规则。

### 8. 文件、项目、工作区、压缩与导入包

传统文件解析仍是原生客户端 RCE
的核心，但项目、工作区、导入包和恢复包往往比普通媒体更容易形成稳定的逻辑执行链。

- 普通格式：图片、音视频、字体、PDF、文档、电子书、CAD、3D、地图、字幕、播放列表、数据库、备份、抓包和日志。

- 复合格式：项目、工作区、主题、模板、模型、规则、工作流、插件包、更新包、恢复包、诊断包。

- 高风险字段：长度、数量、偏移、索引、压缩字典、嵌套层级、对象引用、元数据、嵌入字体、ICC、EXIF/XMP、外部资源
  URL。

项目和工作区重点检查：

- 打开后自动运行构建任务、终端、Hook、语言服务器、脚本、宏、模板转换器或外部工具。

- 自动加载项目目录
  DLL、QML、JavaScript、Python/Lua、插件、主题或自定义协议。

- 项目配置能否改变 PATH、工作目录、环境变量、插件路径、QML Import Path
  或解释器路径。

- “信任项目”机制是否覆盖所有入口；预览、最近项目、崩溃恢复和命令行打开是否绕过确认。

压缩与导入链不仅要测试经典目录穿越，还要覆盖：

- 绝对路径、UNC、设备路径、重复分隔符、多次解码、尾部点/空格、8.3
  短名、ADS。

- Junction、Symbolic Link、Hard Link、目标根目录本身为重解析点。

- 校验后替换、解压后自动打开、安装 Hook、回滚路径、ZIP Extra Field、TAR
  扩展属性。

- 文件数量、深度、总解压尺寸、压缩炸弹、重复条目、同名大小写冲突和顺序依赖。

### 9. Windows 路径语义、MOTW 与文件系统竞态

Windows 同时存在 DOS、UNC、设备、卷 GUID、驱动器相对路径和 NT
对象命名空间。不同语言运行时、Win32 API、Shell API
和自定义规范化函数对同一字符串可能得出不同结果。

- C:\path 与 C:path；\path；UNC；\\\\\\\\卷 GUID；旧式设备名。

- 混合 / 与 \\重复分隔符、尾部点和空格、大小写、Unicode
  正规化、长路径、8.3 Short Name。

- Alternate Data Stream、目录命名流、保留设备名、不同 API
  对冒号和扩展名的解释。

- WOW64 文件/注册表重定向、32/64 位路径差异、用户配置目录与程序目录 ACL
  继承。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>原始输入字符串<br />
→ 第一次 URL / JSON / IPC 解码结果<br />
→ 应用自己的规范化结果<br />
→ 传给 Win32 / Shell / .NET API 的路径<br />
→ 最终句柄解析到的真实对象路径</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

文件系统竞态重点覆盖“先检查、后按路径重新打开”的逻辑：

- 检查安全后释放句柄，随后目录被替换为 Junction 或 Reparse Point。

- 低权限进程创建临时文件，高权限进程稍后按可预测名称打开、移动、签名或执行。

- 扩展名检查后文件被替换；符号路径用于授权，文件句柄用于实际操作，两者不一致。

- 清理、权限修复、备份恢复、日志收集、崩溃转储和更新服务递归跟随链接。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Mark-of-the-Web（MOTW）</strong></p>
<ul>
<li><p>检查客户端下载、复制、解压、导入、同步和恢复时是否保留
Zone.Identifier。</p></li>
<li><p>检查压缩包内文件、项目目录、脚本、HTML、快捷方式、插件和安装包是否因
MOTW 丢失而被当作本地可信内容。</p></li>
<li><p>关注“远程内容落盘后变成本地内容”的边界：WebView、本地脚本、ShellExecute、SmartScreen、Protected
View 和插件信任策略可能因此改变。</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 10. 恶意服务端响应与局域网自动发现

客户端安全测试必须反向考虑“恶意服务器能返回什么”。登录、配置、通知、消息、文件名、缩略图、富文本、更新清单、WebSocket
和二进制协议都可能进入本地解析器。

- 长度与真实帧尺寸不一致；类型字段与 Payload
  不一致；压缩标志和内容不一致。

- 重定向、分块传输、重复 Header、异常编码、超大/超小对象、深层嵌套。

- 消息顺序变化、重复、丢失、迟到响应、请求 A 对应响应
  B、断线恢复和重连。

- 远程文件名、保存路径、头像/贴纸/SVG/字体、富文本链接、自动下载和缓存落盘。

- Protobuf、FlatBuffers、MessagePack、自定义二进制、压缩字典、加密封装与增量同步包。

局域网自动发现是偏门但低交互的入口：

- mDNS、SSDP/UPnP、UDP
  Broadcast、自定义设备发现、打印/扫描、投屏、游戏房间、远程控制、代理发现。

- 发现包中提供名称、图标、XML/JSON、控制 URL
  或后续资源地址；第一阶段只需把客户端引向第二阶段恶意内容。

- 覆盖客户端在前台、后台、锁屏、多网卡、VPN、IPv4/IPv6
  和网络切换时的状态差异。

### 11. localhost、WebSocket、OAuth 与浏览器到本机边界

大量客户端通过本地 HTTP/WebSocket/gRPC/自定义端口连接浏览器扩展、OAuth
回调、下载助手、游戏启动器、IDE、代理/VPN 控制面板或本地 Web UI。

- 绑定地址：仅 Loopback、所有网卡、IPv4/IPv6
  差异；固定端口、随机端口和端口复用。

- 认证：一次性 Token、长期 Token、可预测 Token、Cookie、客户端证书和
  Windows 身份。

- 浏览器边界：Origin、Host、CORS、CSRF、WebSocket Origin、DNS rebinding
  防护。

- 危险参数：URL、目标路径、下载位置、程序、命令行、插件、项目、诊断、更新与文件打开。

- 服务权限：普通用户、管理员、SYSTEM；多用户 Session
  是否共享同一个实例。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>OAuth Loopback</strong></p>
<p>回调监听器应只在发起认证时开放、绑定明确的回环地址、验证
state/PKCE，并在响应后关闭。长期开放的通用回调接口需要按普通 localhost
API 进行完整攻击面审计。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

还要覆盖浏览器扩展 Native Messaging Host、企业 SSO
Helper、协议代理和本地调试接口；它们经常拥有比网页更多的文件和进程能力。

### 12. 通知、Jump List、会话恢复、缓存与诊断包

很多高价值问题不是即时触发，而是“远程内容先落盘，应用重启、崩溃恢复、升级迁移或高权限诊断时由另一套代码再次解释”。

- Toast Notification 点击参数、系统托盘菜单、Jump List
  自定义任务、最近打开文件。

- 启动恢复上次会话、崩溃恢复工作区、自动保存、下载缓存、缩略图缓存、离线消息库。

- 版本升级迁移、配置兼容层、插件状态恢复、旧版序列化对象、更新状态和回滚状态。

- 日志查看器、富文本日志、崩溃转储上传器、支持包导入、诊断包解压、管理员支持工具。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>二次触发链</strong></p>
<p>远程输入被普通进程保存 → 当前会话无异常 → 应用重启/升级/崩溃恢复 →
旧版或高权限组件解析缓存 →
执行。测试计划必须包含“写入—退出—重启—恢复—升级/降级”的完整状态序列。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- 普通权限客户端生成的数据是否会被管理员/SYSTEM 诊断组件再次解释。

- 日志内容是否进入格式化、超链接、Shell、脚本或 HTML 渲染。

- 崩溃报告中的文件路径、模块名、命令行和环境是否被辅助工具重新使用。

### 13. 媒体、字体、图形、打印与设备数据

现代客户端常把一个输入送入多层解析流水线：容器 → 解压 → 图片/视频 →
色彩配置 → 字体 → GPU/图形。任一层都可能由第三方组件实现。

- WIC 图片 Codec、Media Foundation、第三方音视频解码器、SVG、PDF
  内嵌媒体。

- 字体、嵌入字体、OpenType 表、字幕、播放列表、波形、封面、缩略图。

- ICC 色彩配置、EXIF/XMP、3D
  模型纹理、地图瓦片、打印预览、模板和渲染缓存。

- 摄像头、扫描仪、打印机、USB 设备描述、蓝牙设备名称、厂商扩展数据和能力
  XML。

偏门触发动作包括：仅显示消息列表预览、鼠标悬停读取时长、后台媒体库扫描、云同步后索引、项目预加载全部资源、打印预览和崩溃恢复重建缓存。

## 第三部分　组件与本地信任边界

### 14. Electron、WebView2、CEF 与 Qt WebChannel

这类客户端的核心链是“可控网页内容 → XSS/导航/Frame → Native
Bridge/IPC/Host Object → 文件、进程、Shell、更新或插件能力”。单独的 XSS
未必高危，但 XSS 与 Native 能力结合时往往直接升级。

- Electron：preload、contextBridge、ipcMain.handle/on、shell.openExternal、protocol、webContents、webview、autoUpdater、child_process。

- WebView2：AddHostObjectToScript、WebMessageReceived、导航、新窗口、虚拟主机映射、本地目录、下载处理。

- CEF：Message Router、JavaScript
  Binding、命令行开关、远程调试、下载/导航回调。

- Qt：QWebChannel、QDesktopServices、QProcess、QML、QPluginLoader、QQmlEngine
  Import Path。

高价值与偏门检查项：

- 主页面可信，但恶意 iframe、子 Frame 或弹窗能否发送 IPC；是否校验
  senderFrame 和实际 Origin。

- 初始页面可信，导航/重定向到攻击者页面后 Bridge
  是否仍存在；注入脚本是否对后续所有导航持续有效。

- URL allowlist 是否用字符串前缀而不是真正 URL
  解析；file://、自定义协议和本地 HTML 是否获得同等权限。

- 下载的 HTML/脚本、本地缓存、Service Worker、localStorage/IndexedDB
  是否持久保留高权限 Origin。

- openExternal 是否接受任意协议；Native 接口是否提供通用
  read/write/exec/install/download/diagnose。

- 正式版本是否暴露 DevTools、远程调试、开发开关、宽松 Electron Fuse
  或不安全命令行参数。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>WebView2 虚拟主机映射</strong></p>
<ul>
<li><p>映射目录是否包含用户可写的缓存、下载、日志、主题、插件或项目内容。</p></li>
<li><p>映射 Origin 是否拥有 Host Object、Web Message
或跨源访问权限。</p></li>
<li><p>路径规范化是否允许越出映射根目录；更新后旧缓存页面是否继续持有
Bridge。</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 15. Named Pipe、RPC、COM 与高权限 Broker

典型架构是普通权限 GUI/Renderer 通过 IPC 调用管理员或 SYSTEM
服务。接口存在本身不是漏洞，关键是连接权限、调用者身份、消息完整性和服务对参数的信任。

- Named Pipe、RPC、COM Local Server/In-Proc Server、DDE、localhost
  Socket、WM_COPYDATA。

- Shared Memory/Memory-Mapped File、命名
  Event/Mutex/Section、临时文件或注册表消息队列。

- AppContainer/LowBox Renderer 到 Broker 的
  Capability、句柄传递和路径代理。

| **检查层** | **问题** |
|:--:|----|
| 发现与连接 | 谁能枚举接口？ACL、SDDL、Session、命名空间和 Pipe 实例是否一致？ |
| 身份 | 是否验证 SID、Token、完整性级别、Session、包身份、签名或进程来源？ |
| 消息 | 长度、类型、序列号、请求 ID、对象生命周期、粘包/截断、重放与并发。 |
| 权限使用 | 是否正确模拟客户端；敏感操作前是否过早 Revert；是否错误使用服务自身令牌。 |
| 危险能力 | 程序/参数、任意路径、下载、解压、复制、删除、安装、诊断、驱动/服务操作。 |

偏门问题包括：

- Pipe ACL 正确，但服务只信任客户端传入的“已验证”标志。

- 第一个 Pipe 实例和后续实例权限不同；命名对象可被低权限进程抢占。

- RPC/COM 只检查进程名或窗口标题，不验证真实调用者。

- 服务接收低权限进程传入的句柄后，未重新验证对象类型、路径和访问权。

- 请求取消、断开、服务停止、对象销毁与迟到回调之间产生 UAF。

- 多用户 Session 共用同一 Broker，用户 A 可影响用户 B
  的任务、缓存或文件。

### 16. 更新器、安装器、Repair、Rollback 与卸载

更新器常拥有最高权限、最复杂的网络和文件写入逻辑，也是最容易被忽略的独立产品。主程序安全不代表更新、修复、回滚和卸载路径安全。

- 更新清单、完整包、Delta
  Patch、组件文件是否签名；是否固定预期发布者；清单与包是否绑定。

- 是否只依赖 HTTPS；是否允许降级；是否接受已签名但存在已知漏洞的旧版本。

- 校验发生在下载、解压、移动还是执行前；校验后是否存在 TOCTOU。

- 更新缓存、临时目录、回滚目录、日志目录、安装脚本和重启标记的 ACL。

- 普通用户是否可直接调用高权限 Helper/Service 并指定
  URL、路径、包、参数或完成后程序。

MSI 与安装流程的偏门方向：

- Install、Repair、Advertised Shortcut、Rollback、Uninstall 使用不同
  Custom Action。

- 临时脚本、CustomActionData、Transform、Response
  File、安装日志和缓存是否可被低权限影响。

- 卸载器/修复器从用户可写目录加载配置、DLL、Manifest 或 Helper。

- 外层包签名正确，但解压后新增文件、Delta
  输出和第三方组件未单独绑定验证。

- 更新完成前自动重启客户端，可能加载尚未完成校验或权限修复的组件。

### 17. DLL、插件、Qt 加载路径、SxS 与免注册 COM

DLL 搜索问题的证据门槛不是出现 NAME NOT
FOUND，而是攻击者能在现实威胁模型下控制某个被搜索目录或依赖，并能触发可靠加载。

- LoadLibrary/LoadLibraryEx、SetDllDirectory、AddDllDirectory、SearchPath、当前目录、PATH。

- 主 DLL
  在安全目录，但其二级依赖从项目、下载、缓存、临时或插件目录解析。

- Helper、崩溃处理器、语言包、Codec、数据库驱动、TLS
  插件和主题组件的缺失依赖。

- Qt libraryPaths、QT_PLUGIN_PATH、QML Import
  Path、platforms/imageformats/sqldrivers/styles/tls 等插件类别。

插件生态重点检查签名、发布者绑定、更新渠道、包解压、依赖加载、热更新、卸载残留和项目级插件目录。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>SxS 与免注册 COM</strong></p>
<ul>
<li><p>应用目录或组件目录中的 Manifest 可决定 COM 类和 Assembly
绑定，不一定出现在注册表。</p></li>
<li><p>检查 Manifest、相对组件路径、SxS 目录、语言/版本选择、正常 COM
激活失败后的本地回退。</p></li>
<li><p>更新、复制到其他目录运行、便携模式和项目工作目录可能改变绑定结果。</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 18. 反序列化、XAML/QML、脚本与对象构造

不安全反序列化不只存在于网络协议；缓存、剪贴板、项目、云同步、会话恢复、升级迁移和插件状态都可能把远程数据延迟送入危险对象构造。

- .NET：BinaryFormatter、NetDataContractSerializer、LosFormatter、ObjectStateFormatter、SoapFormatter。

- Json.NET TypeNameHandling、自定义
  SerializationBinder、反射创建类型、Assembly.Load、Activator.CreateInstance。

- XamlReader.Load/Parse、Loose XAML、动态
  ResourceDictionary、MarkupExtension、TypeConverter。

- QML 动态加载、Import
  Path、JavaScript、Lua、Python、PowerShell、宏、模板表达式和规则引擎。

重点确认数据来源：网络响应、消息数据库、剪贴板、项目文件、备份、恢复、配置、插件清单、IPC、升级迁移和下载缓存。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>安全边界判断</strong></p>
<p>“产品支持脚本”不等于漏洞；需要判断不可信内容是否在无充分提示、无信任确认或超出承诺权限的情况下进入拥有系统能力的解释器、对象图或插件环境。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 19. 多进程沙箱、Broker 能力与跨边界链

Electron、WebView、媒体框架和现代 UI 常采用 Renderer/GPU/Utility/Broker
多进程模型。单个渲染进程代码执行的价值取决于沙箱、令牌、文件系统访问和可调用
Broker 能力。

- 记录每个进程的令牌、完整性级别、AppContainer
  SID、Capabilities、Job、Mitigation 和可访问对象。

- 列出 Broker IPC
  方法：文件选择、下载、保存、打开外部协议、打印、插件、更新、诊断、证书、凭据和网络。

- 检查 Renderer 是否可伪造主 Frame、Origin、用户手势、窗口 ID、请求 ID
  或对象句柄。

- 检查 GPU/Utility/Crashpad
  等辅助进程是否加载用户可控文件或继承高权限句柄。

- 评估从 XSS、Renderer RCE 到 Broker
  逻辑越权、任意文件写、外部协议和更新器的组合链。

### 20. 并发、重入、异步回调与状态机漏洞

GUI 客户端的大量 UAF、Double Free
和逻辑绕过来自操作序列，而不是单个畸形字段。应把连接、导航、取消、关闭、重试、恢复和热加载作为
Fuzz 输入的一部分。

- 打开文件过程中关闭窗口；解析时取消；下载完成与控件销毁同时发生。

- WebView 导航、Frame 销毁、IPC 回调和弹窗生命周期交叉。

- 插件卸载后仍有异步任务；更新器重启时后台回调未停止。

- 单实例激活发生在初始化期间；服务断开后旧响应迟到；请求 ID 被重用。

- 多个标签页共享 Native
  对象；文件监控器重复加载；对象删除后缓存仍保留指针。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>打开 → 取消 → 重开<br />
打开 → 关闭窗口 → 迟到回调<br />
连接 → 断开 → 重连 → 旧响应到达<br />
安装插件 → 卸载 → 立即重装<br />
导航 → 弹窗 → 返回 → 销毁 Frame</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 第四部分　挖掘、验证与研判方法

### 21. 静态审计：Source—Transform—Sink

静态审计应以输入来源和边界为起点，结合危险 Sink 反向追踪。搜索到 API
只是线索，必须证明攻击者可控数据如何到达，并识别中间所有规范化和授权判断。

| **技术栈** | **重点 Sink / 关键对象** |
|:--:|----|
| C/C++ | CreateProcess、ShellExecute、LoadLibrary、CoCreateInstance、解析器、解压、临时文件、COM/OLE、Named Pipe/RPC |
| .NET | Process.Start、反序列化、XamlReader、Assembly.Load、Activator、NamedPipeServerStream、HttpListener、WebView2 Host Object |
| Electron | preload、ipcMain、contextBridge、openExternal、protocol、webContents、autoUpdater、child_process |
| Qt | QProcess、QDesktopServices、QWebChannel、QPluginLoader、QLibrary、libraryPaths、QQmlEngine Import Path |

- 标记所有外部输入函数、解析入口、文件映射、网络回调、IPC Handler、Shell
  激活和恢复入口。

- 识别多次解码：URL → JSON → IPC → 命令行；路径 → Shell → Win32；压缩包
  → Manifest → DLL。

- 对权限判断建立支配关系：校验是否发生在所有危险分支之前，是否只校验了一个入口。

- 对内存解析器关注整数宽度、符号扩展、对象计数、偏移相加、递归、引用计数和异常清理。

### 22. 动态观察、行为差分与运行时验证

动态方法的目标是确认真实数据流和第一现场，而不是只收集最终崩溃。

- ProcMon：文件、注册表、进程/线程；Process
  Explorer：模块、句柄、令牌；WinObj/PipeList：对象与 Pipe。

- TCPView/Wireshark/代理：本地端口、服务端响应、重定向、WebSocket
  和局域网协议。

- WinDbg/x64dbg：First-Chance
  Exception、断点、堆、调用栈；ProcDump：自动收集转储。

- Application
  Verifier/PageHeap：更早捕获堆破坏、句柄和锁问题；TTD：向前/向后定位第一次错误写入。

行为差分应比较合法与异常输入的：模块加载、文件路径、子进程、COM
类、Pipe/RPC、网络连接、工作目录、环境变量、权限和崩溃前最后一次系统调用。

### 23. 函数级 Harness 与可重复执行环境

对原生解析器，最有效的方式通常是拆出 DLL/函数并建立最小
Harness，而不是让 Fuzzer 反复启动完整 GUI。

> **1.** 定位真正处理输入的 DLL、导出函数、COM 方法、消息 Handler
> 或网络解析函数。
>
> **2.** 复制最小初始化：COM Apartment、Qt
> Application、媒体/图形环境、插件注册和必要全局状态。
>
> **3.**
> 固定工作目录、区域设置、时间、网络和文件系统布局，消除非确定性。
>
> **4.** 把输入直接送入目标函数；每轮清理对象、缓存、线程和异常状态。
>
> **5.** 优先使用持久循环；对必须在 GUI 线程运行的组件保留最小消息循环。
>
> **6.** 用同一输入重复执行并比较覆盖，确认无隐式状态漂移。

难以拆出的目标可采用：Attach 后投递样本、COM Harness、Named Pipe
客户端、假服务端、进程快照、持久实例和文件监控驱动的样本队列。

### 24. 覆盖率、结构感知、差分、状态机与环境 Fuzz

单一字节变异不足以覆盖复杂客户端。应根据入口组合多种 Fuzz 模型。

| **方法** | **适合目标** | **关注点** |
|:--:|----|----|
| 覆盖率引导 | 原生解析器、DLL、COM、IPC Handler | 稳定 Harness、模块范围、持久模式、Crash 去重 |
| 结构感知 | 容器、项目、协议、压缩包 | 长度/偏移/校验和/对象关系/嵌套/压缩 |
| 差分 | 新旧版、主程序与预览器、x86/x64、跨平台 | 解析结果、规范化、对象数量、崩溃与安全判断差异 |
| 状态机 | 网络、WebView、插件、下载、单实例 | 消息顺序、重复、取消、超时、重连、恢复 |
| 并发 | 异步 GUI、WebView、文件监控、更新器 | 销毁、回调、热加载、请求 ID、共享对象 |
| 环境 | 启动、插件、路径、Helper | PATH/TEMP/当前目录/长路径/非 ASCII/网络共享/磁盘不足 |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Pipeline Fuzz</strong></p>
<p>把 URI 解码 → 参数解析 → 路径规范化 → 下载 → 解压 → 插件发现 → DLL
加载视为一条流水线。各组件单独安全，组合后的语义差异常常才是漏洞根因。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- 有源码：ASan、libFuzzer、单元级 Harness、自定义结构化 Mutator。

- 无源码：WinAFL、Jackalope、TinyInst、动态插桩、硬件跟踪、PageHeap/Verifier。

- 语料：小型合法样本、多版本样本、边界字段、真实项目、异常状态序列和补丁前后差异样本。

### 25. Patch Diff、版本考古与兄弟函数审计

Patch Diff
的价值不止是复现旧漏洞，而是找出供应商修复的安全假设，并搜索同模块、其他入口和其他产品中未同步修复的代码。

- 比较当前版/上一版、安全公告前后、稳定/测试、x86/x64、企业/个人、Windows/其他平台。

- 优先关注新增长度上限、整数检查、引用计数、锁、Origin/Sender/SID
  校验、路径规范化、签名和降级限制。

- 搜索同模块中的兄弟解析函数、复制代码、回滚/预览/导入旁路、第三方库旧副本。

- 比较更新器、安装器、主程序、Shell 扩展和插件宿主中的同名 DLL。

常用工具包括 BinDiff、Diaphora、Ghidra/IDA/Binary Ninja
的函数相似度与脚本化对比。

### 26. Crash 去重、根因定位与可利用性研判

Crash
应按根因而不是最终异常地址去重。空指针、资源耗尽和不可控读通常价值较低；可控写、UAF
重占位和间接调用污染优先。

|            **表现**            | **初步判断**                               |
|:------------------------------:|--------------------------------------------|
|   访问 0x0 附近、稳定空指针    | 通常是 DoS；检查是否可通过对象布局转化     |
|           可控越界读           | 信息泄露或 ASLR 绕过零件                   |
|     可控地址/内容的越界写      | 高价值；继续确认第一次错误写入与稳定性     |
|     UAF 且释放对象可重占位     | 高价值；关注虚表、回调、引用计数和线程时序 |
| 函数指针/虚表/间接调用目标污染 | 非常接近代码执行证据                       |
|         任意路径文件写         | 寻找可靠加载点、权限消费者和自然触发       |
| 用户输入进入进程/脚本/插件 API | 可能是直接逻辑 RCE                         |

每个 Crash
至少记录：入口、可控字段、模块、第一错误位置、读/写/执行、地址和内容控制、复现率、线程、目标权限、沙箱、CFG/DEP/ASLR/ACG/CIG
等缓解。

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>第一现场原则</strong></p>
<p>最终崩溃可能距离根因很远。优先使用 PageHeap、First-Chance
Exception、硬件断点和 TTD
追踪第一次错误写入、第一次释放或第一次错误状态转移。</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 27. 漏洞链组合、证据门槛与优先级管理

中低危问题只有在具备清晰、现实、稳定的组合关系时才应升级。不要把理论上可串联的任意两点强行称为
RCE。

- 任意文件写 + 可预测模块/插件/Manifest/脚本加载。

- XSS/任意导航 + Native Bridge/IPC/Host Object。

- Deep Link 参数混淆 + 子进程/外部工具/插件安装。

- 路径穿越/重解析点 + 高权限更新、修复、诊断或清理服务。

- 签名降级 + 旧版已知解析器漏洞。

- 低权限 IPC + SYSTEM Broker 的通用文件/进程能力。

- 远程内容落盘 + 会话恢复/升级迁移/高权限诊断二次解析。

组合链证据应回答：攻击者是否真实控制每一步；中间是否需要额外权限；触发是否自然；路径/进程/Origin
是否确定；每个原语是否稳定；是否有产品设计提示或信任确认。

### 28. 无破坏 PoC、报告结构与修复建议

验证目标是证明安全边界，而不是展示破坏能力。建议使用隔离副本、一次性目录、无联网的标记程序和最小输入。

| **发现类型** | **最小证明方式** |
|:--:|----|
| 逻辑执行 | 启动实验室专用无害程序，仅记录进程身份并在临时目录写随机标记后退出 |
| 任意文件写 | 写入隔离目录中的哨兵文件，证明路径越界与可靠加载关系，不覆盖真实组件 |
| 高权限 IPC | 普通用户调用一次无害操作，证明身份校验缺失与高权限 Sink 可达 |
| 内存破坏 | 证明可控地址/内容、对象重占位、间接调用污染和稳定复现，不要求武器化 |
| 更新链 | 使用本地测试仓库/签名测试包，证明验证、降级、解压或写入边界 |

报告建议结构：

> **1.** 标题：入口 + 根因 + 最终影响，避免泛化。
>
> **2.** 环境：产品、精确版本、安装方式、Windows 版本、组件和权限。
>
> **3.** 威胁模型：攻击者控制什么、需要什么交互、客户端处于何种状态。
>
> **4.** 数据流：Source → Transform → Boundary → Sink。
>
> **5.**
> 根因：缺失的边界检查、身份验证、路径绑定、对象生命周期或整数检查。
>
> **6.** 复现：最小、稳定、无破坏步骤与附件。
>
> **7.** 影响：目标进程权限、沙箱、可组合原语、真实场景。
>
> **8.** 修复：固定路径/程序与参数分离、强身份校验、Origin/Sender
> 校验、签名绑定、句柄式安全路径、禁用危险反序列化、最小 Broker
> 能力、解析器沙箱化。

## 附录 A　高价值组合链清单

> **1.** 远程网页 → Deep Link → 单实例 IPC → 内部路由 →
> 子进程参数边界突破。
>
> **2.** 恶意 iframe/子 Frame → 未校验 senderFrame 的 IPC → Native
> 文件或进程接口。
>
> **3.** WebView 虚拟主机目录可写 → 本地脚本替换 → 可信 Origin Bridge。
>
> **4.** 攻击者网页 → localhost/WebSocket → 缺少 Origin/认证 →
> 危险客户端操作。
>
> **5.** 远程消息附件 → 自动缩略图/预览 → 第三方图片、字体或媒体解码器。
>
> **6.** 远程文件同步 → 恶意文件名/路径语义差异 → 插件或脚本目录写入。
>
> **7.** 导入包 → Junction/Reparse Point →
> 高权限更新/修复服务写入其他位置。
>
> **8.** 更新器允许降级 → 合法签名旧版本 → 旧解析器漏洞重新暴露。
>
> **9.** 剪贴板自定义格式 → 旧式 .NET 反序列化或对象构造。
>
> **10.** OLE 虚拟文件 → 描述符与实际流不一致 → Native Parser。
>
> **11.** 项目文件 → 修改 Qt Plugin/QML Path → 自动加载用户可控组件。
>
> **12.** 可写 Manifest → 免注册 COM/SxS 绑定到非预期组件。
>
> **13.** 项目打开 → 自动恢复终端、任务、Hook 或脚本 → 缺少信任确认。
>
> **14.** 服务端返回文件名 → 缓存落盘 → Helper 从相对路径/当前目录启动。
>
> **15.** 更新包外层签名正确 → 解压路径/链接控制 →
> 验证范围之外的文件写入。
>
> **16.** 崩溃恢复/升级迁移文件 → 下次启动自动加载 →
> 对象构造或脚本执行。
>
> **17.** 普通用户可控日志 → 管理员诊断程序按富格式、HTML 或 Shell
> 再解释。
>
> **18.** 第二实例参数安全 → 转发给第一实例后被另一套解析器重新拆分。
>
> **19.** 自定义 URI → openExternal → 另一个本地协议处理器 →
> 高权限客户端。
>
> **20.** 文件预览安全 → Property Handler
> 写回元数据时发生文件覆盖或内存破坏。
>
> **21.** 下载内容本身安全 → 远程文件名进入 Shell、脚本或 Helper 参数。
>
> **22.** 插件主 DLL 位于安全目录 → 二级依赖从用户可写目录加载。
>
> **23.** 主更新流程安全 → Repair/Rollback/Uninstall
> 仍使用旧的不安全路径。
>
> **24.** 远程响应 → 异步回调命中已销毁 GUI/Frame 对象 → UAF。
>
> **25.** 普通用户 IPC → SYSTEM “诊断/维护”接口 → 任意路径或程序参数。
>
> **26.** 远程文件解压/复制时丢失 MOTW → 本地
> HTML/脚本/快捷方式被当作可信内容。
>
> **27.** Renderer 仅有受限权限 → Broker 接口可伪造用户手势/主 Frame →
> 高权限文件或 Shell 能力。
>
> **28.** 局域网发现包 → 二级控制 URL/图标资源 → Web/媒体解析器。
>
> **29.** 插件市场清单 → 发布者/包绑定不足 → 合法插件 ID 对应错误组件。
>
> **30.** 临时文件可预测 → 低权创建/替换 →
> 高权限组件重新打开、签名、移动或执行。

## 附录 B　Windows 客户端 RCE 攻击面检查清单

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr>
<th style="text-align: center;"><strong>类别</strong></th>
<th style="text-align: center;"><strong>检查项</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;">被动入口</td>
<td>□ Preview/Thumbnail/Property/Icon/IFilter<br />
□ 目录浏览、排序、搜索、属性、悬停<br />
□ 消息列表预览、缩略图、后台索引</td>
</tr>
<tr>
<td style="text-align: center;">文件与项目</td>
<td>□ 自定义格式、容器、元数据、嵌套资源<br />
□ 项目自动任务、脚本、插件、语言服务器<br />
□ 压缩、导入、恢复、诊断、更新包</td>
</tr>
<tr>
<td style="text-align: center;">激活与网页</td>
<td>□ URI/Deep Link、文件关联、Jump List<br />
□ 单实例转发、命令行、外部协议<br />
□ localhost/WebSocket/OAuth/Native Messaging</td>
</tr>
<tr>
<td style="text-align: center;">Web/Native</td>
<td>□ preload/IPC/Host Object/QWebChannel<br />
□ Frame/Origin/导航/弹窗/本地 HTML<br />
□ 虚拟主机映射、缓存、Service Worker</td>
</tr>
<tr>
<td style="text-align: center;">本地边界</td>
<td>□ Named Pipe/RPC/COM/DDE/WM_COPYDATA<br />
□ 共享内存、命名对象、Session<br />
□ 高权限 Broker/服务身份验证</td>
</tr>
<tr>
<td style="text-align: center;">更新与安装</td>
<td>□ 签名、发布者绑定、降级、Delta<br />
□ 解压、临时目录、TOCTOU、回滚<br />
□ MSI Custom Action、Repair、Uninstall</td>
</tr>
<tr>
<td style="text-align: center;">加载链</td>
<td>□ DLL 搜索、二级依赖、Helper<br />
□ Qt 插件/QML/语言包/Codec<br />
□ SxS、Manifest、免注册 COM</td>
</tr>
<tr>
<td style="text-align: center;">对象与脚本</td>
<td>□ 反序列化、TypeNameHandling<br />
□ XAML/QML/ResourceDictionary<br />
□ Python/Lua/PowerShell/宏/模板</td>
</tr>
<tr>
<td style="text-align: center;">状态与并发</td>
<td>□ 缓存、会话/崩溃恢复、迁移<br />
□ 取消、关闭、重连、迟到回调<br />
□ 插件热加载、请求 ID、共享对象</td>
</tr>
<tr>
<td style="text-align: center;">研判</td>
<td>□ Source—Transform—Boundary—Sink<br />
□ 第一次错误位置与稳定复现<br />
□ 权限、沙箱、缓解、组合链和无破坏 PoC</td>
</tr>
</tbody>
</table>

## 附录 C　单个发现的记录模板

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>产品和精确版本：<br />
安装方式 / 架构 / Windows 版本：<br />
组件和模块：<br />
输入入口：<br />
攻击者前置条件：<br />
用户交互与客户端状态：<br />
实际处理进程与完整性级别：<br />
沙箱 / AppContainer / Broker：<br />
攻击者可控字段：<br />
Source → Transform → Boundary → Sink：<br />
第一次错误或错误信任位置：<br />
根因：<br />
读 / 写 / 执行原语：<br />
地址控制程度：<br />
内容控制程度：<br />
稳定复现率：<br />
CFG / DEP / ASLR / ACG / CIG 等缓解：<br />
跨权限或跨 Origin 情况：<br />
可组成的漏洞链：<br />
无破坏 PoC：<br />
当前结论：已确认 / 高概率 / 待确认 / 不成立<br />
修复建议：</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## 附录 D　参考资料与官方文档

以下资料用于进一步核对 Windows
行为、组件边界与工具用法。文档内容以官方安全模型和 API 语义为主。

1\. Microsoft: Shell Handlers —
[<u>https://learn.microsoft.com/en-us/windows/win32/shell/handlers</u>](https://learn.microsoft.com/en-us/windows/win32/shell/handlers)

2\. Microsoft: Preview Handlers —
[<u>https://learn.microsoft.com/en-us/windows/win32/shell/preview-handlers</u>](https://learn.microsoft.com/en-us/windows/win32/shell/preview-handlers)

3\. Microsoft: Shell Drag-and-Drop —
[<u>https://learn.microsoft.com/en-us/windows/win32/shell/dragdrop</u>](https://learn.microsoft.com/en-us/windows/win32/shell/dragdrop)

4\. Microsoft: URI Activation —
[<u>https://learn.microsoft.com/en-us/windows/apps/develop/launch/handle-uri-activation</u>](https://learn.microsoft.com/en-us/windows/apps/develop/launch/handle-uri-activation)

5\. Microsoft: CommandLineToArgvW —
[<u>https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-commandlinetoargvw</u>](https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-commandlinetoargvw)

6\. Microsoft: CreateProcessW —
[<u>https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw</u>](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw)

7\. Microsoft: Windows File Path Formats —
[<u>https://learn.microsoft.com/en-us/dotnet/standard/io/file-path-formats</u>](https://learn.microsoft.com/en-us/dotnet/standard/io/file-path-formats)

8\. Microsoft: Hard Links and Junctions —
[<u>https://learn.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions</u>](https://learn.microsoft.com/en-us/windows/win32/fileio/hard-links-and-junctions)

9\. Microsoft: ZIP and TAR Archive Best Practices —
[<u>https://learn.microsoft.com/en-us/dotnet/standard/io/zip-tar-best-practices</u>](https://learn.microsoft.com/en-us/dotnet/standard/io/zip-tar-best-practices)

10\. Microsoft: Named Pipe Security and Access Rights —
[<u>https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights</u>](https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights)

11\. Microsoft: RPC Security —
[<u>https://learn.microsoft.com/en-us/windows/win32/rpc/security</u>](https://learn.microsoft.com/en-us/windows/win32/rpc/security)

12\. Microsoft: Dynamic-Link Library Search Order —
[<u>https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order</u>](https://learn.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order)

13\. Microsoft: WebView2 Security Best Practices —
[<u>https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/security</u>](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/security)

14\. Microsoft: WebView2 Local Content and Virtual Host Mapping —
[<u>https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/working-with-local-content</u>](https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/working-with-local-content)

15\. Electron: Security, Native Capabilities, and IPC —
[<u>https://www.electronjs.org/docs/latest/tutorial/security</u>](https://www.electronjs.org/docs/latest/tutorial/security)

16\. Qt: QWebChannel —
[<u>https://doc.qt.io/qt-6/qwebchannel.html</u>](https://doc.qt.io/qt-6/qwebchannel.html)

17\. Qt: QPluginLoader —
[<u>https://doc.qt.io/qt-6/qpluginloader.html</u>](https://doc.qt.io/qt-6/qpluginloader.html)

18\. Microsoft: BinaryFormatter Security Guide —
[<u>https://learn.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide</u>](https://learn.microsoft.com/en-us/dotnet/standard/serialization/binaryformatter-security-guide)

19\. Microsoft: XAML Security Considerations —
[<u>https://learn.microsoft.com/en-us/dotnet/desktop/xaml-services/security-considerations</u>](https://learn.microsoft.com/en-us/dotnet/desktop/xaml-services/security-considerations)

20\. Microsoft: Custom Action Security —
[<u>https://learn.microsoft.com/en-us/windows/win32/msi/custom-action-security</u>](https://learn.microsoft.com/en-us/windows/win32/msi/custom-action-security)

21\. RFC 8252: OAuth 2.0 for Native Apps —
[<u>https://datatracker.ietf.org/doc/html/rfc8252</u>](https://datatracker.ietf.org/doc/html/rfc8252)

22\. Microsoft: Sysinternals Suite —
[<u>https://learn.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite</u>](https://learn.microsoft.com/en-us/sysinternals/downloads/sysinternals-suite)

23\. Microsoft: Application Verifier —
[<u>https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/application-verifier-testing-applications</u>](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/application-verifier-testing-applications)

24\. Microsoft: Time Travel Debugging Overview —
[<u>https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/time-travel-debugging-overview</u>](https://learn.microsoft.com/en-us/windows-hardware/drivers/debuggercmds/time-travel-debugging-overview)

25\. Microsoft: Exploit Protection Reference —
[<u>https://learn.microsoft.com/en-us/defender-endpoint/exploit-protection-reference</u>](https://learn.microsoft.com/en-us/defender-endpoint/exploit-protection-reference)

26\. Microsoft: MSVC AddressSanitizer —
[<u>https://learn.microsoft.com/en-us/cpp/sanitizers/asan?view=msvc-170</u>](https://learn.microsoft.com/en-us/cpp/sanitizers/asan?view=msvc-170)

27\. Google Project Zero: WinAFL —
[<u>https://github.com/googleprojectzero/winafl</u>](https://github.com/googleprojectzero/winafl)

28\. Google: BinDiff —
[<u>https://github.com/google/bindiff</u>](https://github.com/google/bindiff)

**— 文档结束 —**

------------------------------------------------------------------------
