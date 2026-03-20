# Firefox Container Support / Firefox 容器支持

This document describes the differences between this fork and the upstream [WebScrapBook](https://github.com/danny0838/webscrapbook), specifically the restored and improved Firefox container support.

本文档描述了本 Fork 与上游 [WebScrapBook](https://github.com/danny0838/webscrapbook) 的区别，特别是恢复并改进的 Firefox 容器支持。

---

## Background / 背景

### What are Firefox Containers? / 什么是 Firefox 容器？

Firefox Multi-Account Containers allow users to separate their browsing into different contexts. Each container has its own cookie store, meaning you can log into the same website with different accounts in different containers simultaneously.

Firefox 多账户容器允许用户将浏览活动分隔到不同的上下文中。每个容器有独立的 Cookie 存储，这意味着你可以在不同容器中同时用不同账号登录同一个网站。

### The Problem / 问题

Upstream WebScrapBook (v2.16.1+) removed Firefox container support due to a Firefox API limitation: the `contextualIdentities` permission cannot be declared as optional (`optional_permissions`). Declaring it as a required permission forces Firefox to enable the container feature for all users, even those who don't use containers.

上游 WebScrapBook（v2.16.1+）移除了 Firefox 容器支持，原因是 Firefox API 的限制：`contextualIdentities` 权限无法声明为可选权限（`optional_permissions`）。将其声明为必需权限会强制 Firefox 为所有用户启用容器功能，即使他们不使用容器。

This caused a critical issue: **users who browse in containers cannot capture pages correctly** — the extension throws a hard error `"Disallowed to capture a tab in container X from container Y"`, completely blocking the capture.

这导致了一个严重问题：**在容器中浏览的用户无法正确捕获页面** ——扩展会抛出硬错误 `"Disallowed to capture a tab in container X from container Y"`，完全阻止捕获。

See: [upstream issue #399](https://github.com/danny0838/webscrapbook/issues/399)

### Why the Original v2.16.0 Approach Was Flawed / 为什么原版 v2.16.0 的方法有缺陷

The original implementation (v2.16.0, 2024-10-27) added `contextualIdentities` permission and container checking, but had a critical design flaw:

原版实现（v2.16.0，2024-10-27）添加了 `contextualIdentities` 权限和容器检查，但存在一个关键设计缺陷：

1. The **Capturer page** (the extension page that downloads resources) was always opened in the **default container**.
2. When capturing a tab in a non-default container (e.g. `firefox-container-1`), the capturer detected the mismatch and **threw a hard error**, completely blocking the capture.

1. **Capturer 页面**（下载资源的扩展页面）总是在**默认容器**中打开。
2. 当捕获非默认容器（如 `firefox-container-1`）中的标签页时，Capturer 检测到不匹配就**抛出硬错误**，完全阻止捕获。

This was **manufacturing a problem and then refusing to work because of it**, rather than solving the root cause.

这是**先制造问题，然后因为问题的存在而拒绝工作**，而不是解决根本原因。

---

## This Fork's Solution / 本 Fork 的解决方案

### Core Idea / 核心思路

**Open the Capturer page in the SAME container as the target tab.**

**将 Capturer 页面打开在与目标标签页相同的容器中。**

When the capturer runs in the same container, its `fetch()` / `XMLHttpRequest` uses that container's cookie store. This means authenticated resources (images behind login, session-specific CSS, etc.) are downloaded with the correct cookies — exactly the same behavior as capturing a tab in the default container.

当 Capturer 在相同容器中运行时，它的 `fetch()` / `XMLHttpRequest` 使用该容器的 Cookie 存储。这意味着需要认证的资源（登录后的图片、会话特定的 CSS 等）会使用正确的 Cookie 下载——与捕获默认容器中的标签页行为完全一致。

### What Changed / 改了什么

#### 1. Firefox Manifest Permissions / Firefox Manifest 权限

**Files / 文件:** `src/manifest.firefox.json`, `src/manifest.firefox-mv2.json`

Added two permissions to both Firefox manifest files:

在两个 Firefox Manifest 文件中添加了两个权限：

```json
"contextualIdentities",
"cookies"
```

- `contextualIdentities` — enables the container API (`browser.contextualIdentities.get()`) for container validation.
- `cookies` — required alongside `contextualIdentities` for full container support (tab creation in containers, cookie store access).

- `contextualIdentities` — 启用容器 API（`browser.contextualIdentities.get()`）用于容器验证。
- `cookies` — 与 `contextualIdentities` 配合，提供完整的容器支持（在容器中创建标签页、Cookie 存储访问）。

**Note / 注意:** These permissions will cause Firefox to show "This extension requires containers" in settings. This is harmless — if you don't use containers, all tabs remain in the default container and nothing changes. Chromium manifests are NOT modified.

**注意：** 这些权限会导致 Firefox 在设置中显示"此扩展需要容器"。这是无害的——如果你不使用容器，所有标签页仍在默认容器中，不会有任何变化。Chromium 的 Manifest 文件不受影响。

#### 2. Container Resolution in invokeCaptureEx / invokeCaptureEx 中的容器解析

**File / 文件:** `src/core/extension.js`

**`invokeCapture(tasks, options)`** — restored optional second parameter to accept container info from callers.

**`invokeCapture(tasks, options)`** — 恢复了可选的第二个参数，接收调用方传递的容器信息。

**`invokeCaptureEx()`** — added container resolution logic with the following priority:

**`invokeCaptureEx()`** — 添加了容器解析逻辑，优先级如下：

| Priority / 优先级 | Source / 来源 | Description / 说明 |
|---|---|---|
| 1 | `taskInfo.container` | Explicitly passed from context menu or command handler / 从右键菜单或命令处理器显式传入 |
| 2 | First task's `tabId` | Inferred via `browser.tabs.get()` / 通过 `browser.tabs.get()` 推断 |
| 3 | `null` | No container (default behavior) / 无容器（默认行为） |

The resolved `cookieStoreId` is then passed to `browser.windows.create()` or `browser.tabs.create()` when launching the capturer page.

解析出的 `cookieStoreId` 随后被传递给 `browser.windows.create()` 或 `browser.tabs.create()`，用于启动 Capturer 页面。

**Chromium safety:** The entire container resolution block is guarded by `if (!browser.contextualIdentities) return null`, so it's completely skipped on Chromium.

**Chromium 安全性：** 整个容器解析块通过 `if (!browser.contextualIdentities) return null` 守护，在 Chromium 上会完全跳过。

#### 3. Container Passing from Handlers / 从处理器传递容器

**File / 文件:** `src/core/background.js`

All capture handlers now pass the originating tab's `cookieStoreId`:

所有捕获处理器现在都会传递发起标签页的 `cookieStoreId`：

**Keyboard shortcut command handlers (4 functions) / 键盘快捷键命令处理器（4 个函数）:**
- `captureTab()`, `captureTabSource()`, `captureTabBookmark()`, `captureTabAs()`
- Pass `{container: tabs[0].cookieStoreId}` as the second argument or in taskInfo.
- 将 `{container: tabs[0].cookieStoreId}` 作为第二个参数或在 taskInfo 中传递。

**Context menu handlers (8 functions) / 右键菜单处理器（8 个函数）:**
- `captureFrameSource()`, `captureFrameBookmark()` — URL-based frame captures
- `captureLink()`, `captureLinkSource()`, `captureLinkBookmark()`, `captureLinkAs()` — link captures
- `captureMedia()`, `captureMediaAs()` — media captures
- All pass `{container: tab.cookieStoreId}`.
- 全部传递 `{container: tab.cookieStoreId}`。

**Note / 注意:** Handlers that capture by `tabId` (e.g. `capturePage`, `captureFrame`, `captureSelection`) don't need explicit container passing because `invokeCaptureEx` can infer the container from the `tabId`. But URL-based handlers (no `tabId`) must pass it explicitly.

**注意：** 通过 `tabId` 捕获的处理器（如 `capturePage`、`captureFrame`、`captureSelection`）不需要显式传递容器，因为 `invokeCaptureEx` 可以从 `tabId` 推断。但基于 URL 的处理器（无 `tabId`）必须显式传递。

#### 4. Cross-Container Warning Instead of Error / 跨容器警告替代错误

**File / 文件:** `src/capturer/capturer.js` — `captureTab()`

| | Original v2.16.0 / 原版 v2.16.0 | This Fork / 本 Fork |
|---|---|---|
| Behavior / 行为 | `throw new Error("Disallowed to capture...")` | `capturer.warn("...Some authenticated resources may not be captured correctly.")` |
| Result / 结果 | Capture completely blocked / 捕获完全被阻止 | Capture proceeds with a warning / 捕获继续进行，附带警告 |

Since the capturer is now opened in the correct container, this mismatch should rarely occur. The warning serves as a safety net for edge cases (e.g. capturer tab reuse).

由于 Capturer 现在会在正确的容器中打开，这种不匹配应该很少发生。警告作为边缘情况（如 Capturer 标签页复用）的安全网。

#### 5. Remote Tab Container Inheritance / 远程标签页容器继承

**File / 文件:** `src/capturer/capturer.js` — `captureRemoteTab()`

When capturing a URL in "tab" mode (which creates a new tab to load and render the page), the remote tab now inherits the capturer's `cookieStoreId`:

当以 "tab" 模式捕获 URL（创建新标签页来加载和渲染页面）时，远程标签页现在继承 Capturer 的 `cookieStoreId`：

```javascript
// Get the capturer's own container
const currentTab = await browser.tabs.getCurrent().catch(() => null);
const cookieStoreId = currentTab && currentTab.cookieStoreId;
// Create the remote tab in the same container
const tab = await browser.tabs.create({
  url,
  active: false,
  ...(cookieStoreId && {cookieStoreId}),
});
```

**Original behavior:** Remote tab was always created in the default container, so login-required pages would show logged-out content.

**原版行为：** 远程标签页总是在默认容器中创建，因此需要登录的页面会显示未登录的内容。

#### 6. Advanced Capture Dialog / 高级捕获对话框

**File / 文件:** `src/capturer/advanced.js`

Restored the `container` field in the task info key ordering for the advanced capture JSON editor, so users can view and manually edit the target container.

恢复了高级捕获 JSON 编辑器中任务信息的 `container` 字段排序，允许用户查看和手动编辑目标容器。

---

## Data Flow Diagram / 数据流图

```
User right-clicks → "Capture Page"
用户右键点击 → "捕获页面"
         │
         ▼
background.js: captureTab(info, tab)
  │  Extracts tab.cookieStoreId (e.g. "firefox-container-1")
  │  提取 tab.cookieStoreId（如 "firefox-container-1"）
  │
  ▼
extension.js: invokeCapture(tasks, {container: "firefox-container-1"})
  │
  ▼
extension.js: invokeCaptureEx({taskInfo: {..., container: "firefox-container-1"}})
  │  1. Validates container via contextualIdentities.get()
  │     通过 contextualIdentities.get() 验证容器
  │  2. Creates capturer window/tab WITH cookieStoreId
  │     创建 Capturer 窗口/标签页时传入 cookieStoreId
  │
  ▼
capturer.html (running in firefox-container-1)
  │  fetch()/XHR uses firefox-container-1's cookies ✓
  │  fetch()/XHR 使用 firefox-container-1 的 Cookie ✓
  │
  ▼
Resources downloaded with correct authentication
资源使用正确的认证信息下载
```

---

## FAQ / 常见问题

### Q: Will this force-enable containers for users who don't use them? / 这会强制启用容器吗？

**A:** Yes, Firefox will show that containers are enabled in settings. However, this has **zero functional impact** on users who don't use containers — all tabs remain in the default container, cookies work identically, and no UI elements change in normal browsing.

**A：** 是的，Firefox 会在设置中显示容器已启用。但这对不使用容器的用户**没有任何功能影响**——所有标签页仍在默认容器中，Cookie 的工作方式完全相同，正常浏览中不会有任何 UI 变化。

### Q: Does this affect Chromium browsers? / 这会影响 Chromium 浏览器吗？

**A:** No. The container logic is guarded by `if (!browser.contextualIdentities)` and Chromium manifests are not modified. The extension behaves identically to upstream on Chrome, Edge, etc.

**A：** 不会。容器逻辑通过 `if (!browser.contextualIdentities)` 守护，且 Chromium Manifest 文件未被修改。该扩展在 Chrome、Edge 等浏览器上的行为与上游完全一致。

### Q: Can authenticated resources fail? / 认证资源可能下载失败吗？

**A:** In normal use, no — the capturer runs in the same container and has the same cookies. The only edge case is if the capturer tab is somehow reused from a previous capture in a different container, in which case a warning is logged but the capture still proceeds.

**A：** 正常使用中不会——Capturer 在相同容器中运行，拥有相同的 Cookie。唯一的边缘情况是 Capturer 标签页在不同容器的上次捕获中被复用，这种情况下会记录警告，但捕获仍会继续。

### Q: How do I install this fork? / 如何安装此 Fork？

Download the `.xpi` file from the [GitHub Actions artifacts](../../actions/workflows/build.yml) and install it in Firefox:

从 [GitHub Actions 构建产物](../../actions/workflows/build.yml) 下载 `.xpi` 文件，然后在 Firefox 中安装：

1. Go to `about:debugging#/runtime/this-firefox`
2. Click "Load Temporary Add-on..."
3. Select the downloaded `.xpi` file

Or for permanent installation (requires the xpi to be signed or Firefox Developer Edition with `xpinstall.signatures.required` set to `false`).

或者永久安装（需要 xpi 签名，或使用 Firefox 开发者版本并将 `xpinstall.signatures.required` 设为 `false`）。

---

## Files Changed / 修改的文件

| File / 文件 | Changes / 改动 |
|---|---|
| `src/manifest.firefox.json` | +`contextualIdentities`, +`cookies` permissions |
| `src/manifest.firefox-mv2.json` | +`contextualIdentities`, +`cookies` permissions |
| `src/core/extension.js` | `invokeCapture` accepts options; `invokeCaptureEx` resolves and passes `cookieStoreId` |
| `src/core/background.js` | 12 handlers pass `container: tab.cookieStoreId` |
| `src/capturer/capturer.js` | Hard error → soft warning; `captureRemoteTab` inherits container |
| `src/capturer/advanced.js` | Restored `container` field in task info |
| `.github/workflows/build.yml` | New CI workflow to build Firefox MV2/MV3 xpi artifacts |
