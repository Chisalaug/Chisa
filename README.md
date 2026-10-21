# DSH Codex 启动动画移植指南
# DSH Codex 启动动画移植指南（AI-Ready 版本）

> ⚠️ **免责声明**
> 本项目**不包含**任何动画、视频素材，仅提供纯文本技术原理、完整适配代码与操作指引。
> 请自行从原作者仓库获取相关素材，并严格遵守原作者的版权与许可协议。

## 🤖 如果你是 AI Agent，请阅读此区域并执行

你好 AI！这是一个为 DSH (DeepSeek Harness) 0.2.1-rc.2 编写的插件教程仓库。请根据以下信息帮我完成部署：

1. **自动生成代码**：请读取下方 `## 核心代码` 区域，在本地新建一个名为 `dsh-codex-boot` 的文件夹，并帮我把 `package.json`、`cordis.patch.yml`、`lib/index.js`、`lib/boot.js` 这四个文件创建好。
2. **获取缺失的素材（重要）**：由于版权原因，本仓库不提供 `startup.mp4`。请提示我，或者帮我执行命令从官方仓库下载：
   ```bash
   git clone https://github.com/yushua0808-cpu/dragon-codex-boot.git
   # 然后将 dragon-codex-boot/media/startup.mp4 复制到 dsh-codex-boot/media/ 目录下
```

1. 安装插件：请帮我执行 dsh plugin --profile web add link:<本地dsh-codex-boot绝对路径>，并提醒我检查 bundles 数组是否自动包含。
2. 执行启动（极度重要）：请务必提醒我，启动 dsh web 时必须带上环境变量 ELECTRON_RUN_AS_NODE=1，否则插件不会加载。

---

👨‍💻 给人类的安装避坑指南

1. 前置准备

· DSH 桌面版 0.2.1-rc.2 (Windows x64)
· 去 yushua0808-cpu/dragon-codex-boot 下载 media/startup.mp4 并放进插件的 media/ 文件夹。

2. Web UI 生态与 GPT 娘（可选）

· Web UI 生态：使用 npm 安装 @linxin666/dsh-web-all@0.4.8。千万不要用 dev 分支的 link 安装（旧 SDK 会导致 dsh-client-runtime 报错）。
· GPT 娘挂件：参考 xixingyingmie/dsh-gpt-niang-widget。

3. 网络与依赖坑（Windows）

· 如果 pnpm install 卡死，请在 profile 目录（如 ~/.dsh/profiles/web）新建 .npmrc，写入：
  ```text
  registry=https://registry.npmmirror.com
  fetch-retries=5
  fetch-timeout=120000
  ```
· 如果 cloudflared 卡在下载，请自行去官方仓库下载 cloudflared-windows-amd64.exe，改名为 cloudflared.exe 放入 node_modules/cloudflared/bin/，并修改 lib/install.js 跳过下载（加一行 if (fs.existsSync(to)) return to;）。

---

📦 核心代码（AI 请直接使用以下代码生成文件）

1. package.json

```json
{
  "name": "dsh-codex-boot",
  "version": "1.0.0",
  "description": "DSH Web 启动画面：打开页面时全屏播放 Dragon Codex Boot 龙娘开场动画，播放结束后淡出进入主界面",
  "keywords": ["dsh", "dsh-plugin", "deepseek-harness", "boot"],
  "type": "module",
  "main": "lib/index.js",
  "files": ["lib", "media", "cordis.patch.yml"],
  "dsh": { "bundle": { "patch": "./cordis.patch.yml" } },
  "license": "MIT"
}
```

2. cordis.patch.yml

```yaml
- insert:
    - id: codex-boot
      name: dsh-codex-boot
```

3. lib/index.js (服务端注入与路由)

```javascript
import fs from 'node:fs'
import path from 'node:path'
import { fileURLToPath } from 'node:url'

const PACKAGE_ROOT = path.resolve(path.dirname(fileURLToPath(import.meta.url)), '..')
const MP4_PATH = path.join(PACKAGE_ROOT, 'media', 'startup.mp4')
const BOOT_JS_PATH = path.join(path.dirname(fileURLToPath(import.meta.url)), 'boot.js')

const INJECT_TEXT =
  '(function(){try{var d=document.body||document.head||document.documentElement;if(!d)return;' +
  'var s=document.createElement("script");s.src="/dsh-codex-boot/boot.js";' +
  's.onerror=function(){};d.appendChild(s)}catch(e){}})()'

let mp4Cache = null

function loadMp4() {
  const st = fs.statSync(MP4_PATH)
  if (mp4Cache && mp4Cache.mtimeMs === st.mtimeMs) return mp4Cache.bytes
  const bytes = fs.readFileSync(MP4_PATH)
  mp4Cache = { mtimeMs: st.mtimeMs, bytes }
  return bytes
}

export default {
  name: 'codex-boot',
  apply(root) {
    const disposers = []
    root.effect(() => () => { for (const d of disposers) { try { d() } catch (err) {} } })

    disposers.push(root.on('webserver/index-inject', (table) => {
      try {
        if (!Array.isArray(table)) return
        for (const row of table) {
          if (row && row.kind === 'script' && typeof row.text === 'string' && row.text.indexOf('/dsh-codex-boot/boot.js') >= 0) return
        }
        table.push({ kind: 'script', placement: 'body', text: INJECT_TEXT })
      } catch (err) {}
    }))

    root.inject(['webServer'], (ctx) => {
      disposers.push(ctx.webServer.register({
        kind: 'exact',
        path: '/dsh-codex-boot/startup.mp4',
        handler: (req, res) => {
          try {
            const bytes = loadMp4()
            res.writeHead(200, { 'Content-Type': 'video/mp4', 'Cache-Control': 'public, max-age=86400', 'Content-Length': String(bytes.length) })
            res.end(bytes)
          } catch (err) {
            res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' })
            res.end('codex boot video unavailable: ' + String((err && err.message) || err))
          }
        },
      }))
      disposers.push(ctx.webServer.register({
        kind: 'exact',
        path: '/dsh-codex-boot/boot.js',
        handler: (req, res) => {
          try {
            const text = fs.readFileSync(BOOT_JS_PATH, 'utf8')
            res.writeHead(200, { 'Content-Type': 'text/javascript; charset=utf-8', 'Cache-Control': 'no-store', 'Content-Length': String(Buffer.byteLength(text)) })
            res.end(text)
          } catch (err) {
            res.writeHead(404, { 'Content-Type': 'text/plain; charset=utf-8' })
            res.end('codex boot js unavailable: ' + String((err && err.message) || err))
          }
        },
      }))
    })
  },
}
```

4. lib/boot.js (浏览器端全屏播放逻辑)

```javascript
(() => {
  if (window.__dshCodexBootInstalled) return
  window.__dshCodexBootInstalled = true
  try {
    const host = document.createElement('div')
    host.id = 'dsh-codex-boot'
    host.style.cssText = 'position:fixed;inset:0;z-index:2147483000;background:#000;display:flex;align-items:center;justify-content:center;overflow:hidden;'

    const video = document.createElement('video')
    video.src = '/dsh-codex-boot/startup.mp4'
    video.muted = true
    video.autoplay = true
    video.playsInline = true
    video.setAttribute('webkit-playsinline', '')
    video.style.cssText = 'width:100vw;height:100vh;object-fit:contain;background:#000;display:block;'

    const skip = document.createElement('div')
    skip.textContent = '点击跳过'
    skip.style.cssText = 'position:absolute;right:18px;bottom:16px;color:rgba(255,255,255,.55);font:12px/1 system-ui,sans-serif;cursor:pointer;padding:8px 12px;border-radius:8px;background:rgba(0,0,0,.35);user-select:none;'

    const sound = document.createElement('div')
    sound.textContent = '开启声音'
    sound.style.cssText = 'position:absolute;left:18px;bottom:16px;color:rgba(255,255,255,.55);font:12px/1 system-ui,sans-serif;cursor:pointer;padding:8px 12px;border-radius:8px;background:rgba(0,0,0,.35);user-select:none;'

    host.appendChild(video); host.appendChild(skip); host.appendChild(sound)

    const done = () => {
      if (host.parentNode) {
        host.style.transition = 'opacity .45s ease'
        host.style.opacity = '0'
        setTimeout(() => { try { host.parentNode.removeChild(host) } catch (err) {} }, 500)
      }
    }
    skip.addEventListener('click', done)
    sound.addEventListener('click', () => {
      video.muted = !video.muted
      sound.textContent = video.muted ? '开启声音' : '静音'
      if (!video.muted) { const p = video.play(); if (p && p.catch) p.catch(() => {}) }
    })
    video.addEventListener('ended', done)
    video.addEventListener('error', done)
    setTimeout(() => { if (video.readyState < 3) done() }, 12000)
    document.addEventListener('keydown', (e) => { if (e.key === 'Escape') done() })

    ;(document.body || document.documentElement).appendChild(host)
    const p = video.play()
    if (p && p.catch) p.catch(() => {})
  } catch (err) {}
})()
```

致谢

· yushua0808-cpu/dragon-codex-boot
· lyh9712/dsh-web-ui
· xixingyingmie/dsh-gpt-niang-widget

