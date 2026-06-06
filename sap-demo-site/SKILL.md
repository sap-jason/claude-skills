---
name: sap-demo-site
description: "Use this skill whenever working on the SAP Demo single-page website — adding features, fixing bugs, optimizing performance, updating content, or deploying. The site is a single-file HTML app (index.html) deployed on Netlify. Trigger on: 'SAP demo site', 'SAP演示网站', 'index.html', 'Netlify部署', 'Saleo', '行业页', 'Demo演示' in context of this project."
---

# SAP Demo Site Skill

## 项目概览

SAP 端到端业务演示网站，单文件 HTML SPA，部署在 Netlify。

| 属性 | 值 |
|------|-----|
| 主文件 | `C:\Users\<YOU>\Desktop\ClaudeOutputs\SAP-Demo-V2\_deploy\index.html` |
| 源文件（同步） | `C:\Users\<YOU>\Desktop\ClaudeOutputs\SAP-Demo-V2\index.html` |
| Netlify Site ID | `YOUR_NETLIFY_SITE_ID` |
| 密码 | `YOUR_PASSWORD` |
| 总行数 | ~3700 行 |

**编辑后必须同步：**
```bash
cp ".../_deploy/index.html" ".../index.html"
```

---

## 文件结构

```
_deploy/
├── index.html          # 全部 CSS + HTML + JS，单文件
├── netlify.toml        # 缓存头配置
└── img/
    ├── SAPlogo.jpg
    ├── ae_hero_img.png      # 自主运营页 hero 图（有 preload link）
    ├── ae_diagram.jpg       # 自主运营页流程图（原 .png，已压缩为 .jpg）
    ├── cases/               # 客户案例图（6张）
    ├── industry/
    │   ├── manifest.json    # 22个行业，每个的图片数量
    │   └── slides_clean/    # slide001.jpg/.webp ... （约271文件）
    └── *_avatar.jpg/svg     # 角色头像（5个固定头像，新版直接复制）
```

---

## 页面结构

共 6 个页面（`.page` + `id="page-*"`），通过 `showPage(id)` 切换：

| id | 内容 |
|----|------|
| `intro` | SAP 介绍（默认页） |
| `autonomous` | 自主运营企业（ae图两张） |
| `demo` | Demo 演示（含4个子tab） |
| `industry` | 行业解决方案（22个行业） |
| `contact` | 联系我们（表单） |

### Demo 子页面（`.demo-subpage`）

| id | tab名 | Saleo URLs数量 |
|----|-------|---------------|
| `page-demo-mts` | 按库存生产 MTS | 12个 |
| `page-demo-mto` | 按订单生产 MTO | 13个 |
| `page-demo-eto` | 按订单设计 ETO | 16个 |
| `page-demo-flow` | 流程总览 | 6个 |

Saleo URL 顺序（每个tab）：端到端总览 → 销售员 → 物控员 → 车间计划员 → 仓管员 → 开票专员 → 其余角色

---

## 核心 JS 架构

### 全局变量

```javascript
const SALEO_URLS_BY_TAB = { mts:[...], mto:[...], eto:[...], flow:[...] };
const iframePool = {};           // url → iframe element
const saleoLoadedTabs = new Set();
let activePopup = null;
let isFullscreen = false;
let currentIframe = null;
```

### 资源加载链（`startPreloadChain`，DCL+500ms 后启动）

严格串行，充分利用带宽，禁止并发：

```
ae图(并行2张) → Saleo MTS(12个串行) → MTO(13个串行) → ETO(16个串行) → flow(6个串行) → 行业图(串行)
```

**插队机制 `window.prioritizeLoad(items)`**：用户点击页面时把该页面资源插到队头，正在飞的请求完成后立即执行插队任务。

触发插队的时机：
- `showPage('autonomous')` → ae图插队
- `showPage('demo')` / `selectDemo()` → 当前tab的Saleo URLs插队
- `showPage('industry')` / `selectIndustry()` → 当前可见行业的图片插队

### Saleo iframe 管理

- **`iframePool{}`**：url → iframe element，预热和按需共享同一池
- **`warmPool`**：`position:fixed;width:0;height:0` 隐藏容器，预热 iframe 在这里
- **`openDemo(label, url)`**：从 iframePool 取 iframe，移入 `demo-iframe-pool`，显示 spinner 直到 loaded
- **`preloadSaleo(tab)`**：用户进入 demo 页时触发（插队机制会更快）
- iframe 加载完成标志：`f.dataset.loaded = '1'`

### 行业图懒加载

- 所有 `img/industry/` 的图保留 `data-src`，不在 DOMContentLoaded 时加载
- `loadIndImgs(sectionEl, onDone)`：加载指定 section 内的所有图，并行
- 进入行业页时：先加载可见行业，完成后加载其余行业

---

## 密码门

```javascript
// 密码存在 sessionStorage，key: 'sap_auth'，value: '1'
// 淡出：gate.classList.add('hiding') → 300ms → classList.add('hidden')
// 关键：checkPassword() 必须在 DOMContentLoaded 之前可调用（已满足）
// 关键：所有预加载延迟 500ms 启动，避免阻塞密码门动画
```

CSS：
```css
#pw-gate.hiding { opacity: 0; pointer-events: none; }  /* transition: opacity 0.3s */
#pw-gate.hidden { display: none; }
```

---

## Demo Modal

```html
<div id="demo-modal">
  <div id="demo-modal-box">
    <div id="demo-iframe-pool" style="flex:1;width:100%;position:relative;">
      <div id="demo-loading">  <!-- spinner，iframe 加载时显示 -->
      <!-- iframe 们在这里 -->
    </div>
  </div>
</div>
```

- `openDemo(label, url)` — 打开 modal，显示 spinner
- `closeModal()` — 关闭，iframe 保持 loaded 状态（不销毁）
- `toggleFullscreen()` — 全屏切换

---

## 行业页

- 22个行业，`manifest.json` 记录每个行业的图片数量
- HTML 中 `<div class="ind-content" id="ind_N">` 包含行业的所有 `<img data-src="...">`
- `selectIndustry(btn, id)` — 切换显示的行业，触发图片插队

---

## 角色气泡 & 弹窗

- `.role-bubble#rb-{role}` 和 `.role-popup#popup-{role}`
- 前缀：MTS无前缀，MTO用 `mto-`，ETO用 `eto-`，flow用 `flow-`
- `togglePopup(role)` — 点击气泡开关弹窗
- `positionPopup(role)` — 计算弹窗位置（相对 canvas，避免溢出边缘）

---

## SAP 视频（Facade 模式）

视频 iframe 默认不加载，显示黑色占位 div + 播放按钮：

```javascript
function loadVideoFacade(el) {
  // el 是 .video-facade div，有 data-src 属性
  // 点击后替换为真实 iframe
}
```

---

## 联系表单

- 提交到第三方表单服务（forminit.com）
- `handleFormSubmit(e)` — fetch POST，成功显示 `#form-success`

---

## Netlify 部署

```toml
# netlify.toml
[[headers]]
  for = "/img/*"
  [headers.values]
    Cache-Control = "public, max-age=31536000, immutable"  # 图片缓存1年

[[headers]]
  for = "/index.html"
  [headers.values]
    Cache-Control = "public, max-age=0, must-revalidate"   # HTML每次重新验证
```

**部署命令：**
```bash
cd "_deploy"
netlify deploy --prod --dir .
```

---

## 设计规范

### CSS 变量

```css
--sap-blue: #0070F2
--sap-dark: #001E60
--sap-mid:  #0050C4
--sap-light: #E8F0FE
--ink: #1A1A2E
--ink-2: #4A5568
--ink-3: #718096
--bg: #F7F9FC
--bg-2: #EEF2F8
--border: #E2E8F0
--nav-h: 64px
--radius: 12px
```

### 字体
`'Inter', 'Microsoft YaHei', -apple-system, sans-serif`

---

## 已知坑 & 修复记录

| 问题 | 原因 | 修复方式 |
|------|------|----------|
| `display:none` 内的图不下载 | 浏览器优化跳过隐藏元素 | 用 `new Image()` in JS heap 强制下载 |
| 135张行业图在DCL同时请求 | `img.src = img.dataset.src` forEach 全触发 | 行业图保留 `data-src`，只在进入页面时加载 |
| `loadIndImgs` 加载后图片不显示 | 改了 data-src skip 逻辑后遗漏 | 检查 `dataset.src` 和 `naturalWidth === 0` 两个条件 |
| 密码门点击后要等很久 | DCL 回调里大量同步操作阻塞主线程 | 所有预加载延迟 500ms 后启动 |
| 预热 iframe 在 warmPool，openDemo 找不到 | iframe 在不同容器里 | `if (f.parentElement !== pool) pool.appendChild(f)` |
| 多资源竞争带宽 | ae/Saleo/行业图同时并发 | 严格串行队列 + 插队机制 |
| ae_diagram.png 355KB 太大 | 未压缩 | 转为 ae_diagram.jpg，71KB |
