# Harmonia JS 模块结构

> 本文件说明 `js/main.js`（单体文件，约 1.3 万行）中各功能区的职责与关键函数。
> 代码已紧凑化，通过函数名前缀可快速定位：`init*` 初始化、`render*` 渲染、`fetch*` 网络请求、
> `handle*` 事件处理、`update*` 状态更新、`build*` 构造 HTML、`open/close*` 弹窗控制、`st*` 智能过渡。

## 顶层基础设施（文件首部）

| 符号 | 职责 |
|------|------|
| `el(id)` | document.getElementById 简写 |
| `qs(s)` / `qsa(s)` | document.querySelector / querySelectorAll 简写 |
| `safeCall(fn,...)` | 捕获异常的调用包装 |
| `safeJsonParse(v, fb)` | JSON.parse 的 try/catch 包装（顶层存储读取统一使用） |
| `escapeHtml(s)` | HTML 文本转义（含引号，可用于属性上下文） |
| `window.onerror / unhandledrejection` | 全局未捕获错误兜底（console） |

## 功能区概览

| 区段 | 关键函数 | 说明 |
|------|----------|------|
| DOM 缓存 | `dynamicIsland`, `searchInput`, `audioPlayer`... | 全局 DOM 引用（启动时一次性缓存） |
| 状态 | `isPlaying`, `currentSongInfo`, `playlist`, `favorites`, `history`, `playlists`, `harmoniaStats` | 应用状态（顶层 let，多处直接修改） |
| 网络 | `wrappedFetch`（25s 超时 + AbortController）、`withKugouRequestDedup` | 统一请求与去重 |
| 灵动岛 | `toggleDynamicIsland`, `expandDynamicIsland`, `showDynamicIslandToast` | 灵动岛展开/收起/提示 |
| 播放控制 | `playSong`, `playFromPlaylist`, `getNextSongId`, `preloadNextSongForGapless`, 无逢隙 `onGaplessEnded` | 播放/切歌/预加载（ended 自动切歌在 `onGaplessEnded`；预加载按音源设置 crossorigin，见 `stApplySourceMediaAttrs`） |
| 智能过渡 | `stTimeUpdateHook`, `stTryStartTransition`, `stChooseEffect`, `stRunBassSwap`, `stPerformEchoOut`, `stRunVolumeMix`, `stEnsureMixer`, `stDetectBpm`, `stMeasureTail`, `stGetSongFeatures` | DJ 式混音引擎（非音量交叉淡化）：按结尾形态选效果——淡出结尾→bassSwap 低频交接（自建混音台，无需开 EQ，A/B 均可安全入图时）、骤然结尾→echoOut 回声收尾（尾部原料进 BPM 同步延迟反馈拖尾）、尾部静音→静音段浮现、分析失败→音量淡化+响度匹配；启动点吸附 4 拍小节边界，高置信鼓点歌轻对齐速度（±3%）；酷狗分析通路：桌面直连（跨域全放行），网页/手机经自建 cors 代理（Range 能力探测 7 天缓存，不支持则降级淡化）；负缓存分级 noCors 24h/networkFail 1h；分析缓存 localStorage（`stAnalysisCache3`，含 loudnessDb），回声原料内存缓存 30 首；交接时响度匹配（matchGain 0.5-2）+4s 缓释；纯函数（效果选择/响度/匹配增益/代理URL）在 `js/lib/pure.js` |
| 歌词 | `parseLyrics`（LRC 家族）、`parseWordLyrics`（YRC/QRC）、`parseKugouKrc`（KRC）、`parseTTMLContentToAMLLLines`（TTML）、`renderAMLLLines`, `updateAMLyricsHighlight`, `normalizeAMLLLine`, `amllSetLyricLinesNoBurst`, `applyAMLLProcessConfig`, IndexedDB 缓存 `getCachedLyrics` | 歌词解析/渲染/高亮/缓存。解析优先走 AMLL 官方 parser（`parseLrcLike`/`parseYrc`/`parseQrc`/`parseTTML`），手写实现降级为兜底；高行数歌词由 core 0.6.0 内置的渲染范围门控按需构建（渲染器不降级、行不合并） |
| 搜索 | `searchMusic`, `displaySearchResults`, `updatePagination` | 搜索/结果/分页 |
| 歌单 | `createPlaylist`, `addTrackToPlaylist`, `syncKugouPlaylists`, `renderPlaylists` | 歌单 CRUD（含酷狗同步） |
| 酷狗 | `fetchKugouVipDetail`, `loginKugou`, `initKugouQrLogin`, `runKugouVipClaimAndUpgrade` | 酷狗 API 与 VIP |
| 均衡器 | `ensureEqAudioGraph`, `applyEqToGraph`, `persistAndRefreshEqUi` | Web Audio EQ（CORS 探测 + 自动关闭） |
| 分享 | `generateShareCard`, `downloadPoster` | 分享卡片（canvas） |
| MV | `fetchAndPlayMV`, `setMvPlaceholder` | MV 播放 |
| PiP | `openDesktopLyricsPip`, `openPipPlayer`, `closePipPlayer`, `syncPipLyrics`, `schedulePipLyricsSync`, `computeNextSyncDelayMs`, `adjustLyricsThemeColor` | 画中画（两窗口互斥，封面按 src 指纹更新；歌词同步按行密度自适应节奏；桌面歌词字体颜色随封面亮度自适应：白/黑字切换 + 中间调背景推向对比侧，经 CSS 变量由 updateTheme 换歌时刷新） |
| 主题 | `initTheme`, `toggleTheme` | 主题切换（body.light-theme） |
| 设置 | `saveAllSettings`, `loadTranslationSettings`, `loadVisualSettings`, `loadEqSettings` | 设置持久化 |
| 统计 | `accumulateStats`, `saveStatsThrottled`, `renderStats` | 播放统计（5s 节流落盘） |
| 初始化 | `init()` | 入口（文件主体唯一调用一次，勿重复调用） |

## AMLL 歌词引擎

引擎版本：`@applemusic-like-lyrics/core@0.6.0` + `@applemusic-like-lyrics/lyric@1.1.0`。

**交付方式**：本地 vendor bundle（`js/vendor/amll-core.bundle.mjs`、`js/vendor/amll-lyric.bundle.mjs`、
`css/vendor/amll-core.css`），由 `tools/amll-build/` 用 esbuild 构建（详见该目录 `build.mjs` 头部说明）。
运行时按 `本地 vendor → esm.sh → esm.sh 重试` 顺序加载，CDN 仅作兜底
（原备用源 jsdelivr 在部分网络下已不可达，故改为 esm.sh 重试）。

> ⚠️ **JS 与 CSS 必须同版本**。`main.html` 引用的 `css/vendor/amll-core.css` 要与
> `js/vendor/amll-core.bundle.mjs` 出自同一次构建。二者错配不会报错，但会静默破坏渲染：
> 曾出现引擎升 0.6.0、样式仍为 0.5.1 的事故——0.6.0 改用 mask-image 渐变做逐字高亮，
> 依赖 `--bright-mask-alpha` / `--dark-mask-alpha` 与 `.FmKaba_gradientMask`（0.5.1 样式均无），
> 变量缺失时 JS 兜底双双取 1，渐变两侧同色 → **所有歌词一律全白，失去逐字高亮**。
> 现在由 `tests/js/amll-lyrics.test.js` 的版本一致性用例守住（含类名交叉校验）。

**升级注意（0.5.1 → 0.6.0 的破坏性变更）**：

| 变更 | 影响 |
|------|------|
| `calcLayout(force, immediate)` → `calcLayout(reason)` | 传旧的两个布尔参数会让 `LayoutReasonStrategyMap[reason]` 取到 `undefined` 并在解引用时抛错；须改传 `LayoutReason` 值（取自 core 模块导出，勿硬编码字符串） |
| 渲染门控重构：`isInSight`/`applyAlphaToDom` → `isInRenderRange()` + 行组 `isUiDirty` | 原先「拦截 update + 等尺寸就绪 + 强制 calcLayout」的防构建风暴 hack 与 `renderStyles` 缓存补丁均已移除：0.6.0 只在行进入渲染范围时才 `rebuildElement()`，且用脏标记避免重复写样式 |
| 新增 `updateLyricProcessConfig` / `setEnableAutoSeekDetection` | `applyAMLLProcessConfig` 用前者批量下发优化项与掩码配置，避免多次重建视图 |

**生命周期**：`registerAMLLUnloadCleanup` 在 `pagehide` 时取消自建 rAF 并调用 `dispose()`
（文档「时序与生命周期 · 清理」检查清单）。切换渲染器路径同样会先 `deactivateAMLLRenderer` 再重建。

**解析分工**（官方优先、手写兜底，两条路径都保留以应对社区格式变体）：

| 格式 | 官方 parser | 兜底实现 | 备注 |
|------|------------|---------|------|
| LRC / LRC A2 / SPL / ESLyric | `parseLrcLike` | `pure.js:parseLyrics` | 官方支持 SPL 规范时间戳（毫秒不足 3 位后位补 0）、显式行结尾、行内逐字标记 |
| 网易云 YRC / QQ 音乐 QRC | `parseYrc` / `parseQrc` | `parseWordLyricsLegacy` | 官方额外做「整行圆括号 → 背景行并去括号」等规范化 |
| 酷狗 KRC | —（私有格式） | `parseKugouKrc` | 官方无对应实现 |
| TTML | `parseTTML` | `simpleTTMLToAMLLLines` | 官方支持 Apple 风格 Head Sidecar（`iTunesMetadata` 翻译/音译）、`tts:ruby` 注音、`amll:obscene`、`amll:empty-beat`、`x-bg` 嵌套 |

适配层 `adaptAmllTtmlLines` / `amllLinesToLegacyWordLines` 位于 `js/lib/pure.js`（纯函数，
`node --test` 直接覆盖），`main.js` 中为委托封装。

**注意**：`normalizeAMLLWord` 必须透传 `ruby` / `obscene` / `emptyBeat` / `romanWord`。
这些字段曾在归一化时被重建丢弃，导致解析层解析正确却渲染不出注音与掩码。
其中 `emptyBeat` 是**数值**节拍数（`amll:empty-beat="2"`，官方用 `parseInt` 解析，
写成 `"true"` 会得到 NaN 被丢弃），`obscene` 为布尔。

## 错误处理

- 网络请求统一走 `wrappedFetch`（25s 超时 + AbortController）
- UI 错误通过 `showError(msg, ms)` 显示在灵动岛
- 关键操作（init、login、search、play）外层 try/catch 兜底
- 全局 `window.onerror` / `unhandledrejection` 兜底（console.error）

## 注意事项

- `main.js` 为全局作用域单体脚本，无模块化/构建步骤；新增函数注意命名冲突
- `js/` 目录部署 `main.js`、`stFlacRepair.js`（FLAC/WAV 无损音源尾部修复）与 `MODULES.md`
- 顶层 localStorage 读取必须走 `safeJsonParse`，避免存储损坏导致整站白屏
- `init()` 只在文件主体执行一次（`<script defer>` 加载），勿在 DOMContentLoaded 中再次调用
- 动态 import 的相对说明符须以 `./` 开头并以 `document.baseURI` 归一：`main.js` 位于 `/js/` 下，
  直接用 `'js/vendor/x.mjs'` 会被当成裸模块说明符而解析失败
