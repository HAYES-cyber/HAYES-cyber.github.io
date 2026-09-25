---
title: "Windows Client RCE Hunting Methodology: A Full-Spectrum Playbook"
date: 2026-08-17
tags: ["windows", "rce", "fuzzing", "vulnerability-research", "binary-security"]
summary: "A complete methodology covering attack-surface mapping, unconventional entry points (clipboard/OLE/Deep Link/file semantics), component trust boundaries (Electron/WebView2/COM/RPC), fuzzing, crash triage, and non-destructive PoC — spanning the full workflow for hunting RCE in Windows desktop clients."
toc: true
draft: false
---

> **Scope:** authorized security research, internal enterprise audits, bug bounty programs, and isolated lab environments.

> This article focuses on discovery, localization, risk assessment, and minimal verification. It does not cover weaponization, persistence, business disruption, or unauthorized testing procedures.

## Overview

The core of Windows desktop client RCE isn't "does the program open a port" — it's whether attacker-controlled content can, via files, web pages, URIs, server responses, update packages, the clipboard, or caches, ultimately reach process creation, module loading, arbitrary file writes, dangerous object construction, or controllable memory corruption.

This playbook unifies common and obscure attack surfaces into a single research framework, with a focus on low-interaction entry points, cross-privilege brokers, Web/Native bridges, update chains, Windows path and filesystem semantics, automatic previews, second-trigger states, and composite chains built from multiple medium/low-severity primitives.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>Attacker-controlled input<br />
↓<br />
Decode / normalize / parse / deserialize / restore state<br />
↓<br />
Cross a Web—Native, low-priv—high-priv, user-dir—program-dir, or sandbox—Broker boundary<br />
↓<br />
Process creation / module loading / arbitrary file write / dangerous object construction / controllable memory corruption<br />
↓<br />
Code execution under the target process's privileges</th>
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
<th><p><strong>Scope of use</strong></p>
<ul>
<li><p>Only conduct research against explicitly authorized assets, internal enterprise audit environments, bug bounty scope, or isolated labs.</p></li>
<li><p>Verification exists to prove boundaries and impact — do not modify, delete, or overwrite business data, and do not implant persistence.</p></li>
<li><p>Memory corruption doesn't need to be forced into full weaponization; evidence such as a stable controllable write, object re-occupation, or indirect-call-target poisoning is already sufficient to support a high-quality assessment.</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Part One — Research Framework and Prioritization

### 1. Scope, RCE Definition, and Research Boundaries

This article discusses Windows clients built on Win32/C/C++, .NET, Qt, Electron, CEF, WebView2, UWP/WinUI auxiliary components, shell extensions, updaters, and background services. The kernel, drivers, and the browser engine itself are not the primary subject, but when a client folds these components into an automatic parsing chain, they should still be included in the risk map.

- Remote-content RCE: the attacker controls a server response, message, web page, LAN broadcast, or sync content, which the client processes automatically or with low interaction and then executes code.

- File-based RCE: code executes when a malicious file, project, imported package, theme, template, or attachment is opened, previewed, indexed, thumbnailed, or restored.

- Browser-to-native RCE: a web page invokes dangerous local-client capabilities via a Deep Link, localhost, WebSocket, Native Messaging, or an external protocol.

- Cross-privilege execution: a normal-user or low-integrity renderer process calls an admin or SYSTEM component via IPC/Broker; reports on this usually involve both RCE and local privilege escalation at once.

- A local-only issue that requires write access to the target directory beforehand, and only causes a null-pointer crash, generally doesn't qualify on its own as a high-quality RCE.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Suggested wording for conclusions</strong></p>
<p>When evidence is insufficient, use phrasing like "can form a code-execution primitive" or "high-probability exploitable memory corruption" or "can cross the intended trust boundary" — don't claim a stable RCE outright based on a single anomalous access.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 2. A Unified Attack-Chain Model and Dangerous Primitives

Every entry point should be traced through four layers: Source—Transform—Boundary—Sink. Don't just search for dangerous APIs, and don't just observe the crash point — the real root cause is often the first incorrect normalization, the first misplaced trust, or the first broken object lifetime.

| **Layer** | **Question to answer** | **Typical objects** |
|:--:|----|----|
| Source | Who controls the input? Remote, a web page, a low-privilege user, or a local file? | Files, URIs, IPC, server responses, clipboard, cache |
| Transform | How many rounds of decoding, argument splitting, path normalization, decompression, or deserialization does it go through? | URL decoding, command-line parsing, JSON/binary parsing, object restoration |
| Boundary | At which step does it cross a privilege, process, origin, sandbox, or filesystem boundary? | Renderer→Main, user→SYSTEM, downloads directory→plugin directory |
| Sink | What usable primitive is ultimately obtained? | CreateProcess, LoadLibrary, arbitrary write, dangerous object construction, function-pointer poisoning |

High-value dangerous primitives typically include:

- Attacker-controlled strings reaching process creation, a shell, a script interpreter, an external tool, or a plugin-install interface.

- Arbitrary-path file writes that can land in a reliable load point, startup point, script directory, manifest, plugin directory, or high-privilege consumer directory.

- An untrusted web page, iframe, or local HTML gaining a Native Bridge, Host Object, IPC, or general file/process capability.

- A normal user being able to invoke a high-privilege service's general file, download, extraction, diagnostic, install, rollback, or launch functionality.

- Deserialization, XAML/QML object construction, or dynamic assembly/script loading.

- In memory corruption: a controllable write, a UAF whose freed object can be re-occupied, or vtable/function-pointer/indirect-call-target poisoning.

### 3. Prioritizing and Scoring High-Value Targets

It's best to scan logic-type and cross-boundary vulnerabilities first, and only invest in complex memory exploitation afterward. The reason is that logic-type RCE is usually more stable, has a clearer root cause, is cheaper to reproduce, and is easier to demonstrate real impact for.

| **Priority** | **Direction** | **Reason** |
|:--:|----|----|
| S | Web/Native bridge, high-privilege broker, updater writes, auto-preview, automatic parsing of remote responses | Low interaction, cross-boundary, stable, and clearly impactful |
| A | Deep Link parameter chains, insecure deserialization, project auto-execution, reliable DLL/plugin loading | Usually leads to execution directly or via a short chain |
| B | Arbitrary-path write with no load point, XSS with no bridge, controllable out-of-bounds read, limited IPC privilege escalation | Good as a piece of a composite chain |
| C | Null pointer, plain DoS, only triggers in debug builds, requires equivalent write access as a precondition | Limited standalone value |

The following internal scoring model can be used to rank the queue:

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>Total score = remote controllability(0–3) + degree of low interaction(0–3) + privilege gain(0–3)<br />
+ stability(0–2) + composability(0–2) − strong preconditions(0–3)</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- First priority: Electron/WebView2/CEF/Qt bridges, auto-update, and IPC to high-privilege services.

- Second priority: Preview/Thumbnail/Property/IFilter, automatic parsing of message attachments, LAN auto-discovery.

- Third priority: project/workspace auto-tasks, archive import chains, plugin and DLL loading.

- Fourth priority: function-level fuzzing and patch diffing of custom native parsers.

### 4. Mapping Processes, Privileges, Inputs, and Trust Boundaries

Before formal auditing, build a "process graph" and an "input matrix" first. This step often narrows the scope more effectively than reading millions of lines of decompiled code directly.

- Processes: main GUI, renderer, GPU, plugin host, updater, crash handler, background service, shell extension, preview host, local web server.

- Privileges: user, admin, SYSTEM; integrity level; AppContainer/LowBox; token, session, desktop, and UIAccess.

- Interfaces: named pipe, RPC, COM, localhost, WebSocket, shared memory, Windows message, temp files, registry, and custom sockets.

- Loading: DLLs, plugins, manifests, QML, scripts, language packs, themes, codecs, fonts, COM classes, and child processes.

- On-disk artifacts: downloads, cache, updates, session recovery, crash recovery, logs, diagnostic packages, offline messages, and project state.

| **Field** | **What to record** |
|:--:|----|
| Input source | File, filename, URI, web page, server, clipboard, IPC, LAN, cache, update package |
| Trigger condition | Client closed/running/backgrounded; logged in or not; clicked or not; auto-recovered or not |
| Handling process | Process name, parent process, command line, current directory, integrity level, sandbox |
| Data flow | Raw value, result after each decode/normalization step, final API argument or the file handle's real path |
| Dangerous outcome | Child process, module load, file write, object construction, memory anomaly, privilege crossing |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Baseline recommendation</strong></p>
<p>For every entry point, run at least one legitimate input and one anomalous input, and compare the differences in files, registry, modules, child processes, network activity, IPC, and COM activation. Behavioral diffing frequently exposes hidden parsers and secondary load points directly.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Part Two — External Input and Passive Trigger Surfaces

### 5. Explorer, Shell Extensions, and Search Indexing

The value of this class of entry point is that a user may only need to browse a directory, switch views, select a file, view properties, or let the system index in the background, and a client component starts parsing already.

- Preview Handler, Thumbnail Handler, Property Handler, Icon Handler, Overlay Handler.

- Context Menu Handler, Drop Handler, Copy Hook, Namespace Extension.

- Windows Search IFilter, the search protocol handler, file-attribute write-back, and metadata enumeration.

- File associations, default-open programs, right-click commands, ShellExecute, the preview pane, and the details pane.

Recommended trigger actions to test separately:

- Just place a file in a directory; open the directory; switch between large icons/thumbnails; enable the preview pane.

- View properties; sort by metadata fields such as author, dimensions, or duration; use Windows Search.

- Hover, single-click select, right-click menu, drag-and-drop, copy, move, and rename.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Niche but high value</strong></p>
<ul>
<li><p>The same format may be parsed independently by the main program, the Explorer thumbnail, a Property Handler, an IFilter, a previewer, and a cloud-drive client; differentially test multiple parsers against the same sample.</p></li>
<li><p>A Property Handler doesn't just read — it may also write metadata back; the write-back path, temp files, and object lifetime are an independent attack surface.</p></li>
<li><p>Delayed triggers need to cover different hosts such as SearchIndexer, SearchProtocolHost, prevhost, and dllhost, and each host's own privilege level.</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 6. Clipboard, OLE, Virtual Files, and Drag-and-Drop

The Windows clipboard and drag-and-drop deliver multiple formats via IDataObject/STGMEDIUM. The same object can simultaneously declare text, HTML, RTF, bitmap, files, streams, and embedded objects, and a client may use a different parser for each stage.

- CF_HDROP, CF_UNICODETEXT, HTML Clipboard Format, RTF, DIB/PNG.

- FileGroupDescriptor/FileContents virtual files, Outlook attachment drag-and-drop, delayed rendering data.

- Embedded objects, link sources, OLE embedded/linked objects, custom registered clipboard formats.

- The TYMED, IStream, HGLOBAL, file handle, and lifetime management returned by IDataObject::GetData.

Unconventional testing ideas:

- Have the same object declare multiple mutually contradictory formats, and observe the client's priority and fallback path.

- Make the name, length, and extension in the file descriptor inconsistent with the real stream.

- Mutate each stage — DragEnter, DragOver, Drop, Paste — separately, and close the window or cancel the operation mid-processing.

- Have the data source exit, disconnect, or return a different size during delayed rendering; check for UAF, double-free, and corrupted state.

- Have clipboard content originate from a remote desktop, browser, or chat app, then get pasted and consumed by a higher-privilege client.

### 7. URIs, Deep Links, Command Lines, and Single-Instance Forwarding

A Deep Link often forms a multi-layer parsing chain: "web page → Windows protocol activation → second instance → single-instance IPC → internal routing in the first instance → file/plugin/child process." Each layer may use different encoding and quoting rules.

- Registration locations: HKCU/HKLM\Software\Classes, HKCR, URL Protocol, shell\open\command, App Paths.

- Test the query, path, and fragment; repeated parameters; case sensitivity; empty values; overlong values; Unicode; newlines; quotes; backslashes; and nested URLs.

- Compare behavior when the client is not running, already running, initializing, before login, running as an admin instance, or under a multi-user session.

- Confirm whether the arguments still stay within the expected boundary after URL decoding, Windows command-line splitting, IPC serialization, and internal routing.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Argument logger</strong></p>
<p>In the lab, use a harmless argument-logging program that records the final executable path, argc/argv, working directory, environment variables, parent process, integrity level, and inherited handles. This is safer than directly trying a command, and better demonstrates that the argument boundary was actually broken.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- Key sinks: CreateProcess, ShellExecute, Process.Start, QProcess, child_process, script interpreters, archive tools, compilers, and plugin installers.

- Key differences to check: whether the program path is complete; whether the program and arguments are separated; whether the working directory is controllable; whether a relative helper is used; whether PATH/PATHEXT/TEMP are inherited.

- Niche points: @response-file, extension omission, SearchPath, PATHEXT, current-drive-relative paths like C:folder, and the backslash-before-quote rule.

### 8. Files, Projects, Workspaces, Archives, and Import Packages

Traditional file parsing is still the core of native-client RCE, but projects, workspaces, import packages, and recovery packages often form a stable logic-execution chain more easily than ordinary media.

- Common formats: images, audio/video, fonts, PDF, documents, ebooks, CAD, 3D, maps, subtitles, playlists, databases, backups, packet captures, and logs.

- Composite formats: projects, workspaces, themes, templates, models, rules, workflows, plugin packages, update packages, recovery packages, diagnostic packages.

- High-risk fields: length, count, offset, index, compression dictionary, nesting depth, object references, metadata, embedded fonts, ICC, EXIF/XMP, external resource URLs.

Key checks for projects and workspaces:

- Whether opening it auto-runs a build task, terminal, hook, language server, script, macro, template converter, or external tool.

- Whether it auto-loads DLLs, QML, JavaScript, Python/Lua, plugins, themes, or a custom protocol from the project directory.

- Whether the project configuration can change PATH, the working directory, environment variables, plugin paths, the QML import path, or the interpreter path.

- Whether the "trust this project" mechanism covers every entry point; whether preview, recent projects, crash recovery, or command-line opening can bypass confirmation.

For archive and import chains, don't just test classic directory traversal — also cover:

- Absolute paths, UNC paths, device paths, repeated separators, multiple rounds of decoding, trailing dots/spaces, 8.3 short names, ADS.

- Junctions, symbolic links, hard links, and the target root directory itself being a reparse point.

- Replacement after verification, auto-open after extraction, install hooks, rollback paths, ZIP extra fields, TAR extended attributes.

- File count, depth, total decompressed size, compression bombs, duplicate entries, same-name case conflicts, and order dependencies.

### 9. Windows Path Semantics, MOTW, and Filesystem Races

Windows has DOS paths, UNC paths, device paths, volume GUID paths, drive-relative paths, and the NT object namespace all at once. Different language runtimes, Win32 APIs, Shell APIs, and custom normalization functions can reach different conclusions for the same string.

- C:\path vs. C:path; \path; UNC; \\\\\\\\volume GUID; legacy device names.

- Mixing / and \\; repeated separators; trailing dots and spaces; case; Unicode normalization; long paths; 8.3 short names.

- Alternate Data Streams, directory-named streams, reserved device names, and different APIs' interpretation of colons and extensions.

- WOW64 file/registry redirection, 32/64-bit path differences, and ACL inheritance between the user profile directory and the program directory.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>Raw input string<br />
→ result after the first URL / JSON / IPC decode<br />
→ result after the app's own normalization<br />
→ the path passed to the Win32 / Shell / .NET API<br />
→ the real object path the final handle resolves to</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Filesystem races should focus on the "check first, then reopen by path" pattern:

- A handle is released after a security check, and the directory is then replaced with a junction or reparse point.

- A low-privilege process creates a temp file, and a high-privilege process later opens, moves, signs, or executes it by a predictable name.

- The file is replaced after an extension check; a symbolic path is used for authorization while the file handle is used for the actual operation, and the two are inconsistent.

- Cleanup, permission repair, backup restore, log collection, crash-dump handling, and the update service all recursively follow links.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Mark-of-the-Web (MOTW)</strong></p>
<ul>
<li><p>Check whether the client preserves the Zone.Identifier when downloading, copying, extracting, importing, syncing, or restoring.</p></li>
<li><p>Check whether files inside an archive, a project directory, a script, HTML, a shortcut, a plugin, or an installer get treated as trusted local content because MOTW was lost.</p></li>
<li><p>Pay attention to the boundary where "remote content, once written to disk, becomes local content" — WebView, local scripts, ShellExecute, SmartScreen, Protected View, and plugin trust policy can all change because of this.</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 10. Malicious Server Responses and LAN Auto-Discovery

Client-side security testing must also think in reverse about "what can a malicious server return." Logins, configuration, notifications, messages, filenames, thumbnails, rich text, update manifests, WebSocket traffic, and binary protocols can all end up in a local parser.

- Length inconsistent with the real frame size; type field inconsistent with the payload; compression flag inconsistent with the content.

- Redirects, chunked transfer, duplicate headers, unusual encodings, oversized/undersized objects, deeply nested structures.

- Changed message order, duplication, loss, late responses, request A matched to response B, disconnect-recovery, and reconnection.

- Remote filenames, save paths, avatars/stickers/SVG/fonts, rich-text links, auto-download, and cache writes.

- Protobuf, FlatBuffers, MessagePack, custom binary formats, compression dictionaries, encryption wrappers, and incremental sync packages.

LAN auto-discovery is an obscure but low-interaction entry point:

- mDNS, SSDP/UPnP, UDP broadcast, custom device discovery, print/scan, screen casting, game rooms, remote control, proxy discovery.

- A discovery packet provides a name, icon, XML/JSON, a control URL, or a follow-up resource address; the first stage only needs to steer the client toward second-stage malicious content.

- Cover behavioral differences with the client in the foreground, background, locked screen, multiple NICs, VPN, IPv4/IPv6, and network switching.

### 11. localhost, WebSocket, OAuth, and the Browser-to-Native Boundary

Many clients connect to browser extensions, OAuth callbacks, download helpers, game launchers, IDEs, proxy/VPN control panels, or a local web UI over local HTTP/WebSocket/gRPC/custom ports.

- Bind address: loopback-only vs. all interfaces; IPv4/IPv6 differences; fixed port, random port, and port reuse.

- Authentication: one-time token, long-lived token, predictable token, cookie, client certificate, and Windows identity.

- Browser boundaries: Origin, Host, CORS, CSRF, WebSocket Origin, and DNS rebinding protection.

- Dangerous parameters: URL, target path, download location, program, command line, plugin, project, diagnostics, update, and file-open.

- Service privilege: normal user, admin, SYSTEM; whether multiple user sessions share the same instance.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>OAuth loopback</strong></p>
<p>The callback listener should only be open while authentication is in progress, bind to an explicit loopback address, validate state/PKCE, and close after the response. A general-purpose callback interface that stays open long-term needs a full attack-surface audit as if it were an ordinary localhost API.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

Also cover browser-extension Native Messaging hosts, enterprise SSO helpers, protocol proxies, and local debugging interfaces — they often have more file and process capability than a web page does.

### 12. Notifications, Jump Lists, Session Recovery, Cache, and Diagnostic Packages

Many high-value issues aren't triggered instantly — instead, "remote content is written to disk first, and then re-interpreted by a different piece of code when the app restarts, recovers from a crash, migrates during an upgrade, or a high-privilege diagnostic runs."

- Toast notification click parameters, the system-tray menu, Jump List custom tasks, recently opened files.

- Restoring the last session on launch, crash-recovered workspaces, autosave, download cache, thumbnail cache, offline message stores.

- Version-upgrade migration, the configuration compatibility layer, plugin-state restoration, legacy serialized objects, update state, and rollback state.

- Log viewers, rich-text logs, the crash-dump uploader, support-package import, diagnostic-package extraction, and admin support tools.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Second-trigger chain</strong></p>
<p>Remote input is saved by a normal-privilege process → nothing anomalous happens in the current session → the app restarts/upgrades/recovers from a crash → a legacy or high-privilege component parses the cache → execution. The test plan must include the full state sequence "write — exit — restart — recover — upgrade/downgrade."</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- Whether data produced by a normal-privilege client gets re-interpreted by an admin/SYSTEM diagnostic component.

- Whether log content flows into formatting, hyperlinks, a shell, a script, or HTML rendering.

- Whether file paths, module names, command lines, and environment data inside a crash report get reused by an auxiliary tool.

### 13. Media, Fonts, Graphics, Printing, and Device Data

Modern clients often push one input through a multi-layer parsing pipeline: container → decompress → image/video → color profile → font → GPU/graphics. Any layer may be implemented by a third-party component.

- WIC image codecs, Media Foundation, third-party audio/video decoders, SVG, PDF-embedded media.

- Fonts, embedded fonts, OpenType tables, subtitles, playlists, waveforms, cover art, thumbnails.

- ICC color profiles, EXIF/XMP, 3D model textures, map tiles, print preview, templates, and render caches.

- Camera, scanner, printer, USB device descriptors, Bluetooth device names, vendor extension data, and capability XML.

Unconventional trigger actions include: just showing a message-list preview, hovering to read a duration, a background media-library scan, indexing after cloud sync, a project preloading all its resources, print preview, and cache rebuilding during crash recovery.

## Part Three — Components and Local Trust Boundaries

### 14. Electron, WebView2, CEF, and Qt WebChannel

The core chain for this class of client is "controllable web content → XSS/navigation/frame → Native Bridge/IPC/Host Object → file, process, shell, update, or plugin capability." An XSS alone isn't necessarily high severity, but combining XSS with native capability often escalates it directly.

- Electron: preload, contextBridge, ipcMain.handle/on, shell.openExternal, protocol, webContents, webview, autoUpdater, child_process.

- WebView2: AddHostObjectToScript, WebMessageReceived, navigation, new-window handling, virtual host mapping, local directories, download handling.

- CEF: message router, JavaScript binding, command-line switches, remote debugging, download/navigation callbacks.

- Qt: QWebChannel, QDesktopServices, QProcess, QML, QPluginLoader, QQmlEngine import path.

High-value and unconventional checks:

- The main page is trusted, but can a malicious iframe, child frame, or popup send IPC? Is senderFrame and the real origin validated?

- The initial page is trusted — does the bridge still exist after navigating/redirecting to an attacker page? Does an injected script stay effective across all subsequent navigations?

- Does the URL allowlist use string-prefix matching instead of real URL parsing? Do file://, custom protocols, and local HTML get the same privilege?

- Do downloaded HTML/scripts, local cache, service workers, or localStorage/IndexedDB persistently retain a high-privilege origin?

- Does openExternal accept an arbitrary protocol? Does the native interface offer general read/write/exec/install/download/diagnose capability?

- Does the release build expose DevTools, remote debugging, dev switches, loose Electron fuses, or insecure command-line arguments?

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>WebView2 virtual host mapping</strong></p>
<ul>
<li><p>Does the mapped directory contain user-writable cache, downloads, logs, themes, plugins, or project content?</p></li>
<li><p>Does the mapped origin have Host Object, Web Message, or cross-origin access privileges?</p></li>
<li><p>Does path normalization allow escaping the mapping root? Does an old cached page keep holding onto the bridge after an update?</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 15. Named Pipes, RPC, COM, and High-Privilege Brokers

A typical architecture is a normal-privilege GUI/renderer calling an admin or SYSTEM service over IPC. The interface existing at all isn't a vulnerability by itself — what matters is the connection privilege, caller identity, message integrity, and how much the service trusts the arguments.

- Named pipes, RPC, COM local server/in-proc server, DDE, localhost sockets, WM_COPYDATA.

- Shared memory/memory-mapped files, named events/mutexes/sections, temp files, or a registry-based message queue.

- AppContainer/LowBox renderer-to-broker capability, handle passing, and path proxying.

| **Check layer** | **Question** |
|:--:|----|
| Discovery & connection | Who can enumerate the interface? Are the ACL, SDDL, session, namespace, and pipe-instance behavior consistent? |
| Identity | Is the SID, token, integrity level, session, package identity, signature, or process origin verified? |
| Message | Length, type, sequence number, request ID, object lifetime, coalescing/truncation, replay, and concurrency. |
| Privilege use | Is client impersonation done correctly; is Revert called too early before a sensitive operation; is the service's own token misused? |
| Dangerous capability | Program/arguments, arbitrary paths, download, extraction, copy, delete, install, diagnostics, driver/service operations. |

Unconventional issues include:

- The pipe ACL is correct, but the service only trusts an "already verified" flag passed in by the client.

- The first pipe instance and later instances have different privileges; a named object can be squatted by a lower-privilege process.

- RPC/COM only checks the process name or window title instead of validating the real caller.

- After the service receives a handle passed in from a low-privilege process, it fails to re-verify the object type, path, and access rights.

- A UAF arises between request cancellation, disconnection, service stop, object destruction, and a late-arriving callback.

- Multiple user sessions share the same broker, so user A can affect user B's tasks, cache, or files.

### 16. Updaters, Installers, Repair, Rollback, and Uninstall

Updaters often run with the highest privilege and the most complex network and file-write logic, and they're also the most easily overlooked as their own product. The main program being secure doesn't mean the update, repair, rollback, and uninstall paths are.

- Are the update manifest, full package, delta patch, and component files signed? Is the expected publisher pinned? Are the manifest and package bound together?

- Does it rely on HTTPS only? Does it allow downgrades? Does it accept a signed-but-known-vulnerable older version?

- Does verification happen at download, extraction, move, or right before execution — and is there a TOCTOU gap after verification?

- The ACLs of the update cache, temp directory, rollback directory, log directory, install scripts, and restart marker.

- Whether a normal user can directly invoke a high-privilege helper/service and specify a URL, path, package, arguments, or a post-completion program.

Unconventional angles for MSI and the install flow:

- Install, Repair, Advertised Shortcut, Rollback, and Uninstall each use different custom actions.

- Whether temp scripts, CustomActionData, transforms, response files, install logs, and caches can be influenced by a low-privilege actor.

- Whether the uninstaller/repair tool loads configuration, DLLs, manifests, or helpers from a user-writable directory.

- The outer package is signed correctly, but newly added files, delta output, and third-party components aren't individually bound to verification after extraction.

- The client auto-restarts before the update finishes, potentially loading a component whose verification or permission repair isn't complete yet.

### 17. DLLs, Plugins, Qt Load Paths, SxS, and Registration-Free COM

The evidence bar for a DLL search issue isn't just seeing NAME NOT FOUND — it's whether the attacker can, under a realistic threat model, control some searched directory or dependency and reliably trigger the load.

- LoadLibrary/LoadLibraryEx, SetDllDirectory, AddDllDirectory, SearchPath, the current directory, and PATH.

- The main DLL sits in a safe directory, but a secondary dependency of it resolves from a project, downloads, cache, temp, or plugin directory.

- Missing dependencies for helpers, the crash handler, language packs, codecs, database drivers, TLS plugins, and theme components.

- Qt libraryPaths, QT_PLUGIN_PATH, the QML import path, and plugin categories such as platforms/imageformats/sqldrivers/styles/tls.

For the plugin ecosystem, focus on signing, publisher binding, the update channel, package extraction, dependency loading, hot updates, leftover files after uninstall, and per-project plugin directories.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>SxS and registration-free COM</strong></p>
<ul>
<li><p>A manifest in the app directory or a component directory can determine COM class and assembly binding — it doesn't have to appear in the registry.</p></li>
<li><p>Check the manifest, relative component paths, the SxS directory, language/version selection, and the local fallback after normal COM activation fails.</p></li>
<li><p>Updates, copying to a different directory to run, portable mode, and the project's working directory can all change the binding result.</p></li>
</ul></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 18. Deserialization, XAML/QML, Scripts, and Object Construction

Insecure deserialization isn't limited to network protocols — the cache, clipboard, projects, cloud sync, session recovery, upgrade migration, and plugin state can all feed remote data, with a delay, into dangerous object construction.

- .NET: BinaryFormatter, NetDataContractSerializer, LosFormatter, ObjectStateFormatter, SoapFormatter.

- Json.NET TypeNameHandling, a custom SerializationBinder, reflection-based type creation, Assembly.Load, Activator.CreateInstance.

- XamlReader.Load/Parse, loose XAML, a dynamic ResourceDictionary, MarkupExtension, TypeConverter.

- QML dynamic loading, the import path, JavaScript, Lua, Python, PowerShell, macros, template expressions, and rule engines.

Confirm the data source specifically: network responses, a message database, the clipboard, project files, backups, recovery, configuration, plugin manifests, IPC, upgrade migration, and download cache.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Judging the security boundary</strong></p>
<p>"The product supports scripting" is not the same as "this is a vulnerability." What matters is whether untrusted content reaches an interpreter, object graph, or plugin environment with system-level capability, without adequate prompting, without a trust confirmation, or beyond the privileges the product promises.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 19. Multi-Process Sandboxes, Broker Capability, and Cross-Boundary Chains

Electron, WebView, media frameworks, and modern UI stacks commonly use a Renderer/GPU/Utility/Broker multi-process model. The value of code execution in a single renderer process depends on the sandbox, token, filesystem access, and what broker capability it can call.

- Record each process's token, integrity level, AppContainer SID, capabilities, job object, mitigations, and accessible objects.

- List the broker's IPC methods: file picker, download, save, open-external-protocol, print, plugins, update, diagnostics, certificates, credentials, and networking.

- Check whether the renderer can forge the main frame, origin, user gesture, window ID, request ID, or an object handle.

- Check whether auxiliary processes like GPU/Utility/Crashpad load user-controllable files or inherit high-privilege handles.

- Assess the composite chain from XSS or renderer RCE through to broker logic privilege escalation, arbitrary file write, an external protocol, and the updater.

### 20. Concurrency, Reentrancy, Async Callbacks, and State-Machine Bugs

The bulk of UAF, double-free, and logic-bypass issues in GUI clients come from an operation sequence, not a single malformed field. Connect, navigate, cancel, close, retry, recover, and hot-reload should all be part of the fuzzing input space.

- Closing the window while a file is being opened; canceling mid-parse; a download completing at the same moment its control is destroyed.

- WebView navigation, frame destruction, IPC callbacks, and popup lifetimes crossing each other.

- An async task still pending after a plugin unloads; the updater's background callbacks not stopping when it restarts.

- Single-instance activation happening during initialization; an old response arriving late after the service disconnects; a request ID being reused.

- Multiple tabs sharing a native object; a file watcher loading duplicates; a cache still holding a pointer after the object was deleted.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>Open → cancel → reopen<br />
Open → close window → late callback<br />
Connect → disconnect → reconnect → stale response arrives<br />
Install plugin → uninstall → reinstall immediately<br />
Navigate → popup → back → destroy frame</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Part Four — Hunting, Verification, and Assessment Methodology

### 21. Static Auditing: Source—Transform—Sink

Static auditing should start from input sources and boundaries, combined with tracing backward from dangerous sinks. Finding the API is just a lead — you have to prove how attacker-controlled data actually gets there, and identify every normalization and authorization check along the way.

| **Stack** | **Key sinks / objects** |
|:--:|----|
| C/C++ | CreateProcess, ShellExecute, LoadLibrary, CoCreateInstance, parsers, decompression, temp files, COM/OLE, named pipes/RPC |
| .NET | Process.Start, deserialization, XamlReader, Assembly.Load, Activator, NamedPipeServerStream, HttpListener, WebView2 Host Object |
| Electron | preload, ipcMain, contextBridge, openExternal, protocol, webContents, autoUpdater, child_process |
| Qt | QProcess, QDesktopServices, QWebChannel, QPluginLoader, QLibrary, libraryPaths, QQmlEngine import path |

- Mark every external-input function, parsing entry point, file mapping, network callback, IPC handler, shell activation, and recovery entry point.

- Identify multi-round decoding: URL → JSON → IPC → command line; path → shell → Win32; archive → manifest → DLL.

- Build a dominance relationship for privilege checks: does the check happen before every dangerous branch, or does it only cover one entry point?

- For native parsers, pay attention to integer width, sign extension, object counts, offset addition, recursion, reference counting, and exception cleanup.

### 22. Dynamic Observation, Behavioral Diffing, and Runtime Verification

The goal of dynamic methods is to confirm the real data flow and the first-fault site, not just to collect the final crash.

- ProcMon: files, registry, process/thread; Process Explorer: modules, handles, tokens; WinObj/PipeList: objects and pipes.

- TCPView/Wireshark/a proxy: local ports, server responses, redirects, WebSocket, and LAN protocols.

- WinDbg/x64dbg: first-chance exceptions, breakpoints, heap, and call stacks; ProcDump: automatic dump collection.

- Application Verifier/PageHeap: catch heap corruption, handle, and lock issues earlier; TTD: locate the first bad write forward/backward.

Behavioral diffing should compare legitimate vs. anomalous input across: module loads, file paths, child processes, COM classes, pipes/RPC, network connections, working directory, environment variables, privileges, and the last syscall before the crash.

### 23. Function-Level Harnesses and Repeatable Execution Environments

For native parsers, the most effective approach is usually to pull out the DLL/function and build a minimal harness, rather than repeatedly launching the full GUI under a fuzzer.

> **1.** Locate the DLL, exported function, COM method, message handler, or network-parsing function that actually processes the input.
>
> **2.** Replicate minimal initialization: the COM apartment, the Qt application object, the media/graphics environment, plugin registration, and required global state.
>
> **3.** Pin down the working directory, locale, time, network, and filesystem layout to eliminate non-determinism.
>
> **4.** Feed input directly into the target function; clean up objects, caches, threads, and exception state after each round.
>
> **5.** Prefer a persistent loop; keep a minimal message loop for components that must run on the GUI thread.
>
> **6.** Re-run the same input repeatedly and compare coverage to confirm there's no implicit state drift.

For targets that are hard to pull out, options include: attaching and delivering a sample, a COM harness, a named-pipe client, a fake server, process snapshotting, a persistent instance, and a file-watcher-driven sample queue.

### 24. Coverage-Guided, Structure-Aware, Differential, State-Machine, and Environment Fuzzing

Single-byte mutation isn't enough to cover a complex client. Combine multiple fuzzing models based on the entry point.

| **Method** | **Good fit** | **Focus** |
|:--:|----|----|
| Coverage-guided | Native parsers, DLLs, COM, IPC handlers | Stable harness, module scope, persistent mode, crash dedup |
| Structure-aware | Containers, projects, protocols, archives | Length/offset/checksum/object relationships/nesting/compression |
| Differential | Old vs. new version, main program vs. previewer, x86/x64, cross-platform | Parse results, normalization, object counts, differences in crash/safety judgment |
| State-machine | Network, WebView, plugins, downloads, single-instance | Message order, duplication, cancellation, timeout, reconnect, recovery |
| Concurrency | Async GUI, WebView, file watcher, updater | Destruction, callbacks, hot-reload, request ID, shared objects |
| Environment | Startup, plugins, paths, helpers | PATH/TEMP/current directory/long paths/non-ASCII/network shares/low disk space |

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>Pipeline fuzzing</strong></p>
<p>Treat URI decode → argument parsing → path normalization → download → extraction → plugin discovery → DLL load as a single pipeline. Each component may be safe on its own; the semantic mismatch that shows up once they're composed is often the actual root cause.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

- With source: ASan, libFuzzer, unit-level harnesses, custom structured mutators.

- Without source: WinAFL, Jackalope, TinyInst, dynamic instrumentation, hardware tracing, PageHeap/Verifier.

- Corpus: small legitimate samples, samples from multiple versions, boundary-value fields, real-world projects, anomalous state sequences, and pre-/post-patch diff samples.

### 25. Patch Diffing, Version Archaeology, and Sibling-Function Auditing

The value of patch diffing isn't just reproducing an old vulnerability — it's finding the security assumption behind the vendor's fix, and then searching the same module, other entry points, and other products for code that wasn't fixed in sync.

- Compare the current version against the previous one, before/after a security advisory, stable/beta, x86/x64, enterprise/personal, and Windows/other platforms.

- Focus first on newly added length caps, integer checks, reference counting, locks, Origin/Sender/SID validation, path normalization, signing, and downgrade restrictions.

- Search for sibling parsing functions in the same module, copy-pasted code, rollback/preview/import bypasses, and old bundled copies of third-party libraries.

- Compare identically named DLLs across the updater, installer, main program, shell extension, and plugin host.

Common tools include BinDiff, Diaphora, and the function-similarity and scripted comparison features in Ghidra/IDA/Binary Ninja.

### 26. Crash Deduplication, Root-Cause Localization, and Exploitability Assessment

Crashes should be deduplicated by root cause, not by the final faulting address. Null pointers, resource exhaustion, and uncontrollable reads are usually lower value; prioritize controllable writes, a UAF whose freed object can be re-occupied, and indirect-call-target poisoning.

|            **Symptom**            | **Initial assessment**                               |
|:------------------------------:|--------------------------------------------|
|   Access near 0x0, stable null pointer    | Usually DoS; check whether it can be converted via object layout     |
|           Controllable out-of-bounds read           | Info leak or an ASLR-bypass building block                   |
|     Out-of-bounds write with controllable address/content      | High value; keep confirming the first bad write and stability     |
|     UAF where the freed object can be re-occupied     | High value; focus on vtables, callbacks, ref-counting, and thread timing |
| Function-pointer/vtable/indirect-call-target poisoning | Very close to code-execution evidence                       |
|         Arbitrary-path file write         | Look for a reliable load point, a privileged consumer, and a natural trigger       |
| User input reaching a process/script/plugin API | Potentially a direct logic RCE                         |

For every crash, record at minimum: entry point, controllable fields, module, first-fault location, read/write/execute, degree of address and content control, reproduction rate, thread, target privilege, sandbox, and mitigations such as CFG/DEP/ASLR/ACG/CIG.

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th><p><strong>The first-fault-site principle</strong></p>
<p>The final crash can be far from the root cause. Prefer PageHeap, first-chance exceptions, hardware breakpoints, and TTD to trace the first bad write, the first bad free, or the first bad state transition.</p></th>
</tr>
</thead>
<tbody>
</tbody>
</table>

### 27. Composing Vulnerability Chains, the Evidence Bar, and Priority Management

Medium/low-severity issues should only be escalated when they have a clear, realistic, stable compositional relationship. Don't force-call any two theoretically chainable points an RCE.

- Arbitrary file write + a predictable module/plugin/manifest/script load point.

- XSS/arbitrary navigation + a Native Bridge/IPC/Host Object.

- Deep Link argument confusion + a child process/external tool/plugin install.

- Path traversal/a reparse point + a high-privilege update, repair, diagnostic, or cleanup service.

- Signature downgrade + a known vulnerability in an old parser.

- Low-privilege IPC + a SYSTEM broker's general file/process capability.

- Remote content written to disk + second-parse during session recovery/upgrade migration/high-privilege diagnostics.

The evidence for a composite chain should answer: does the attacker genuinely control every step; is any additional privilege needed in between; does the trigger happen naturally; are the path/process/origin fixed and known; is each primitive stable; and is there a product-design hint or trust confirmation involved.

### 28. Non-Destructive PoC, Report Structure, and Remediation Recommendations

The goal of verification is to prove the security boundary, not to demonstrate destructive capability. Use an isolated copy, a disposable directory, an offline marker program, and minimal input.

| **Finding type** | **Minimal proof method** |
|:--:|----|
| Logic execution | Launch a lab-only harmless program that only records the process identity and writes a random marker to a temp directory before exiting |
| Arbitrary file write | Write a sentinel file into an isolated directory to prove the path-traversal-to-reliable-load relationship, without overwriting a real component |
| High-privilege IPC | Have a normal user invoke one harmless operation to prove the identity check is missing and the high-privilege sink is reachable |
| Memory corruption | Prove controllable address/content, object re-occupation, indirect-call-target poisoning, and stable reproduction — weaponization is not required |
| Update chain | Use a local test repo/signed test package to prove a verification, downgrade, extraction, or write-path boundary |

Recommended report structure:

> **1.** Title: entry point + root cause + final impact, avoiding vague generalities.
>
> **2.** Environment: product, exact version, install method, Windows version, components, and privilege.
>
> **3.** Threat model: what the attacker controls, what interaction is required, and what state the client is in.
>
> **4.** Data flow: Source → Transform → Boundary → Sink.
>
> **5.** Root cause: the missing boundary check, identity verification, path binding, object lifetime handling, or integer check.
>
> **6.** Reproduction: minimal, stable, non-destructive steps and attachments.
>
> **7.** Impact: the target process's privilege, sandbox, composable primitives, and a real-world scenario.
>
> **8.** Fix: separate the fixed path/program from arguments, strong identity verification, Origin/Sender validation, signature binding, handle-based safe paths, disabling dangerous deserialization, minimal broker capability, and sandboxing the parser.

## Appendix A — High-Value Composite-Chain Checklist

> **1.** Remote web page → Deep Link → single-instance IPC → internal routing → child-process argument-boundary break.
>
> **2.** Malicious iframe/child frame → IPC with no senderFrame validation → a native file or process interface.
>
> **3.** A writable WebView virtual-host directory → local script replacement → a trusted origin's bridge.
>
> **4.** Attacker web page → localhost/WebSocket → missing Origin/auth → a dangerous client operation.
>
> **5.** Remote message attachment → auto thumbnail/preview → a third-party image, font, or media decoder.
>
> **6.** Remote file sync → malicious filename/path-semantics mismatch → a write into a plugin or script directory.
>
> **7.** Import package → junction/reparse point → a high-privilege update/repair service writes to a different location.
>
> **8.** Updater allows downgrade → a legitimately signed old version → an old parser vulnerability re-exposed.
>
> **9.** Custom clipboard format → legacy .NET deserialization or object construction.
>
> **10.** OLE virtual file → descriptor inconsistent with the real stream → the native parser.
>
> **11.** Project file → modifies the Qt plugin/QML path → auto-loads a user-controllable component.
>
> **12.** Writable manifest → registration-free COM/SxS binds to an unintended component.
>
> **13.** Opening a project → auto-restores a terminal, task, hook, or script → no trust confirmation.
>
> **14.** Server-returned filename → written to cache → a helper launches from a relative path/current directory.
>
> **15.** Update package's outer signature is correct → extraction-path/link control → a file write outside the verified scope.
>
> **16.** A crash-recovery/upgrade-migration file → auto-loaded on next launch → object construction or script execution.
>
> **17.** A log a normal user can control → an admin diagnostic tool re-interprets it as rich format, HTML, or shell.
>
> **18.** Second-instance arguments are safe → forwarded to the first instance, then re-split by a different parser.
>
> **19.** Custom URI → openExternal → another local protocol handler → a high-privilege client.
>
> **20.** File preview is safe → the Property Handler writing metadata back causes a file overwrite or memory corruption.
>
> **21.** The download content itself is safe → the remote filename flows into a shell, script, or helper argument.
>
> **22.** The plugin's main DLL sits in a safe directory → a secondary dependency loads from a user-writable directory.
>
> **23.** The main update flow is safe → Repair/Rollback/Uninstall still use an old, unsafe path.
>
> **24.** A remote response → an async callback hits an already-destroyed GUI/frame object → UAF.
>
> **25.** Normal-user IPC → a SYSTEM "diagnostics/maintenance" interface → an arbitrary path or program argument.
>
> **26.** MOTW is lost when a remote file is extracted/copied → local HTML/script/shortcut treated as trusted content.
>
> **27.** The renderer only has restricted privilege → the broker interface can be tricked into forging a user gesture/main frame → high-privilege file or shell capability.
>
> **28.** LAN discovery packet → second-stage control URL/icon resource → a web/media parser.
>
> **29.** A plugin-marketplace manifest → insufficient publisher/package binding → a legitimate plugin ID mapped to the wrong component.
>
> **30.** A predictable temp file → a low-privilege create/replace → a high-privilege component reopens, signs, moves, or executes it.

## Appendix B — Windows Client RCE Attack-Surface Checklist

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr>
<th style="text-align: center;"><strong>Category</strong></th>
<th style="text-align: center;"><strong>Checklist item</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align: center;">Passive entry points</td>
<td>□ Preview/Thumbnail/Property/Icon/IFilter<br />
□ Directory browsing, sorting, search, properties, hover<br />
□ Message-list preview, thumbnails, background indexing</td>
</tr>
<tr>
<td style="text-align: center;">Files &amp; projects</td>
<td>□ Custom formats, containers, metadata, nested resources<br />
□ Project auto-tasks, scripts, plugins, language servers<br />
□ Archives, imports, recovery, diagnostics, update packages</td>
</tr>
<tr>
<td style="text-align: center;">Activation &amp; web</td>
<td>□ URI/Deep Link, file associations, Jump List<br />
□ Single-instance forwarding, command line, external protocols<br />
□ localhost/WebSocket/OAuth/Native Messaging</td>
</tr>
<tr>
<td style="text-align: center;">Web/Native</td>
<td>□ preload/IPC/Host Object/QWebChannel<br />
□ Frame/Origin/navigation/popups/local HTML<br />
□ Virtual host mapping, cache, service worker</td>
</tr>
<tr>
<td style="text-align: center;">Local boundaries</td>
<td>□ Named pipe/RPC/COM/DDE/WM_COPYDATA<br />
□ Shared memory, named objects, sessions<br />
□ High-privilege broker/service identity verification</td>
</tr>
<tr>
<td style="text-align: center;">Update &amp; install</td>
<td>□ Signing, publisher binding, downgrade, delta<br />
□ Extraction, temp directory, TOCTOU, rollback<br />
□ MSI custom actions, Repair, Uninstall</td>
</tr>
<tr>
<td style="text-align: center;">Load chain</td>
<td>□ DLL search, secondary dependencies, helpers<br />
□ Qt plugins/QML/language packs/codecs<br />
□ SxS, manifests, registration-free COM</td>
</tr>
<tr>
<td style="text-align: center;">Objects &amp; scripts</td>
<td>□ Deserialization, TypeNameHandling<br />
□ XAML/QML/ResourceDictionary<br />
□ Python/Lua/PowerShell/macros/templates</td>
</tr>
<tr>
<td style="text-align: center;">State &amp; concurrency</td>
<td>□ Cache, session/crash recovery, migration<br />
□ Cancel, close, reconnect, late callbacks<br />
□ Plugin hot-load, request ID, shared objects</td>
</tr>
<tr>
<td style="text-align: center;">Assessment</td>
<td>□ Source—Transform—Boundary—Sink<br />
□ First-fault location and stable reproduction<br />
□ Privilege, sandbox, mitigations, composite chains, and non-destructive PoC</td>
</tr>
</tbody>
</table>

## Appendix C — Per-Finding Record Template

<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr>
<th>Product and exact version:<br />
Install method / architecture / Windows version:<br />
Component and module:<br />
Input entry point:<br />
Attacker preconditions:<br />
User interaction and client state:<br />
Actual handling process and integrity level:<br />
Sandbox / AppContainer / Broker:<br />
Attacker-controllable fields:<br />
Source → Transform → Boundary → Sink:<br />
First bad location or misplaced trust:<br />
Root cause:<br />
Read / write / execute primitive:<br />
Degree of address control:<br />
Degree of content control:<br />
Stable reproduction rate:<br />
Mitigations such as CFG / DEP / ASLR / ACG / CIG:<br />
Cross-privilege or cross-origin situation:<br />
Composable vulnerability chain:<br />
Non-destructive PoC:<br />
Current conclusion: confirmed / high probability / unconfirmed / not valid<br />
Remediation recommendation:</th>
</tr>
</thead>
<tbody>
</tbody>
</table>

## Appendix D — References and Official Documentation

The following resources are for further verification of Windows behavior, component boundaries, and tool usage. The content relies primarily on official security models and API semantics.

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

**— End of document —**

------------------------------------------------------------------------
