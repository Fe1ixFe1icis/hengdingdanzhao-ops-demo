# ApiAdapter 接口契约（前后端分离的接缝）

> UI（`app.js`）**只**通过 `window.DZAdapter` 暴露的方法取数据与执行逻辑，不直接碰底层存储或网络。
> 这样公开演示版与私有完整版可以共用同一套 UI，只替换适配器实现。

## 两种实现

| 实现 | 文件 | 形态 | 用途 |
|---|---|---|---|
| `ClientAdapter` | `adapter.client.js` | 浏览器本地（localStorage） | 公开演示版 / 单文件离线完整版 |
| `HttpAdapter`（预留） | `adapter.http.js` | 调用私有后端 `/api/*` | 需要硬保护或多端共享数据时 |

切换到后端版本只需：让 `adapter.http.js` 实现下表同名方法，然后在 `index.html` 里替换脚本引用即可，`app.js` 无需改动。

## 必须实现的方法

### 元信息
| 方法 | 返回 | 说明 |
|---|---|---|
| `isDemo()` | `boolean` | 是否演示版（影响到导出行为与 UI 文案） |
| `version` | `string` | 版本号 |

### 字典
| 方法 | 返回 | 说明 |
|---|---|---|
| `getGroups()` | `Array<Group>` | 学校/专业组列表（含内嵌考纲） |
| `getGroup(id)` | `Group \| null` | 单个专业组 |
| `groupLabel(g)` | `string` | 显示用短标签 |

### 提示词
| 方法 | 返回 |
|---|---|
| `fillPrompt(tpl, vars)` | `string` |
| `buildNotePrompt(groupId, opts)` | `string` |
| `buildPlanPrompt(vars)` | `string` |
| `buildXhsPrompt(vars)` | `string` |

### 解析与合规
| 方法 | 返回 | 说明 |
|---|---|---|
| `normalizePaste(raw)` | `string` | 归一化粘贴文本 |
| `parseNotes(raw)` | `Note` | AI 文本 → 结构化笔记 |
| `scanCompliance(text)` | `Array<Hit>` | 命中违规词 |
| `noteTextForScan(note)` | `string` | 拼接笔记文本用于扫描 |

### 渲染
| 方法 | 返回 | 说明 |
|---|---|---|
| `renderNotePreview(note, opts)` | `string(HTML)` | 屏幕预览 |
| `renderNoteHTML(note, opts)` | `string(HTML)` | 独立成品 HTML（完整版导出） |
| `renderRichFragment(note)` | `string(HTML)` | 公众号用 inline-style 片段 |
| `renderPreviewImage(note, opts)` | `string(dataURL)` | 带/不带水印的图片预览 |

### 导出
| 方法 | 返回 | 说明 |
|---|---|---|
| `exportNote(note, mode)` | `{ok, locked, preview?, message?}` | 演示版 `locked:true` 且仅返回 `preview` |
| `download(filename, content, mime)` | `void` | 触发下载（完整版） |

### 存储（ClientAdapter 提供；HttpAdapter 可映射为 REST）
`list / save / upsert / removeItem / store.get / store.set / exportAll / importAll / ensureSeed`

## AI 调用的接缝

AI 调用封装在 `window.DZAI`（`ai.client.js`），同样是可替换层：

| 方法 | 说明 |
|---|---|
| `load() / save(cfg)` | 读写配置（Base URL / API Key / 模型 / 温度 / 流式） |
| `isConfigured()` | 是否已配置 |
| `endpoint()` | 由 Base URL 推出 `/chat/completions` 地址 |
| `chat(messages, {stream, onDelta, signal})` | OpenAI 兼容对话，支持 SSE 流式；返回完整文本 |
| `test()` | 连通性测试 |
| `abort()` | 中断当前生成 |

公开演示版：浏览器直连服务商（Key 存本地）。
私有完整版如需**隐藏 Key / 统一计费 / 规避 CORS**，把 `chat()` 改为请求自家后端 `/api/ai/chat`，
由后端持 Key 转发即可，UI 无需改动。

## 升级为硬保护的迁移路径

把以下方法从**客户端**迁到**服务端**，浏览器不再持有原文与全量逻辑：

1. `buildNotePrompt` / 提示词库 → 服务端下发，前端只拿到拼好的字符串；
2. `renderNoteHTML` / `renderRichFragment` / `renderPreviewImage` → 服务端渲染并加水印后返回，成品永不进浏览器；
3. `parseNotes` 的解析内核 → 服务端执行；
4. 数据 CRUD → 改为 `/api/notes`、`/api/leads` 等接口（需鉴权）。

迁移后，即使前端源码被拿走，也无法独立运行成产品。
