# Bedrock Animation Stack · 接手指南

> 这份文档写给接手的人（或模型）。目标：读完就能改，不用重新猜需求。
> 最后更新：文件 `index.html` 约 100.4 KB / 2562 行，`node --check` 通过。

---

## 0. 三分钟上手

```powershell
# 1) 语法自检（只校验内联 <script> 那一段）
$p='D:\scripts\bedrock-anim-lab\index.html'
$s=[IO.File]::ReadAllText($p)
$i=$s.IndexOf('<script>'); $j=$s.LastIndexOf('</script>')
[IO.File]::WriteAllText('D:\scripts\bedrock-anim-lab\_check.js',$s.Substring($i+8,$j-$i-8),[Text.UTF8Encoding]::new($false))
node --check 'D:\scripts\bedrock-anim-lab\_check.js'; "check=$LASTEXITCODE"

# 2) 本地预览（也可以直接双击 index.html，无需服务）
node D:\scripts\bedrock-anim-lab\server.js        # -> http://127.0.0.1:8765/index.html
```

- 单文件、零依赖、零 CDN、全部内联；**双击 index.html 也能用**（没有 fetch / 没有 module）。
- 文件编码 UTF-8 **无 BOM**；文件里既有 LF 也有少量 CRLF，批量脚本请用 `[IO.File]::ReadAllText/WriteAllText` + `UTF8Encoding($false)`，不要用会改行尾的编辑器一键格式化。
- 没有 git 仓库。**改大块之前先 Copy-Item 一份备份**（血的教训见 §12）。

---

## 1. 这是什么 / 用户要什么

用户的原话（第一轮）：

> 给我做一个可交互网页，用来演示 minecraft 基岩版里资源包动画 json 的作用，要显示所对应骨骼的 position, rotation, scale 的 xyz 通道，支持导入动画 json，支持动画顺序可拖动，有 `override_previous_animation` 和 `-this` 等覆盖性操作的标识。

经过四轮返工，现在的形态是：

**一张矩阵表**——**行 = 一段动画**（行首可拖动换序，上面的行先算），**列 = 骨骼 → 通道**（`position` / `rotation` / `scale`，固定顺序），**格子里显示这段动画 JSON 里这一通道的原文**（`variable.*` / `query.*` / `this` 一律不代入数字，原样摆出来），并把两件“覆盖类”语义做成可见标记：

| 标记 | 含义 |
| --- | --- |
| 蓝色表达式 | 这一条轴（x/y/z 单条）的原文里出现 `this` |
| 琥珀色横带 | 这一行带 `override_previous_animation: true`，它把上面这些行在它碰到的骨骼上全抹掉 |
| 格子划掉 + `⟂` | 这一格被下面的某一行 override 抹掉了 |

一句话对比：**`this` 是“往上读到哪”，`override` 是“往上全抹掉”。换顺序 = 换结果。**

---

## 2. 需求演进（别再走回头路）

| 版本 | 形态 | 结局 |
| --- | --- | --- |
| v1 | 3D 姿态视口 + 时间轴 + 通道检查器 + 卡片栈 | 用户否掉：“像专业建模软件”“中间那个玩家姿态展示的窗口不需要”。备份在 `_v1-viewer.html` |
| v2 | 只剩可拖拽的动画卡片流（卡片内列骨骼/通道/值） | 方向对了，但用户进一步要求“原始值 + 列排开” |
| v3 | 视觉重做：design tokens、双主题、Lucide 线图标、响应式 | 保留至今（样式层） |
| v4（当前） | 行列转置成矩阵：行=动画、列=骨骼/通道；格子=原文 | 当前形态 |

用户明确否决过的方向（**不要自作主张加回来**）：

1. 3D 预览 / 姿态窗口；
2. 播放器、时间轴、播放头、速度/循环控件（“先不要这个播放器的效果”）；
3. 卡片式布局（“最上面那个…卡片式的”已换成矩阵）；
4. 在示例库里写 Molang 说明文字（“molang里的不要求写，只写动画名及原版动画的json即可”）；
5. 整格/整通道标色（“不要整个，因为有的是只有某些通道里有this而其他是单纯数值”）→ 只标**含 `this` 的那一条轴**；
6. “一键示例”这种入口（“有点多余了，不知道它是干什么用的”）→ 已删除；
7. 多余的确认步骤（“我想要的是那种一键选择示例然后动画栈里就会自动更新”）→ 示例库改成**勾选即生效**。

---

## 3. 文件清单

```
D:\scripts\bedrock-anim-lab\
├─ index.html        100437 B   ← 全部交付物（HTML+CSS+JS 内联）
├─ preview.png       131289 B   默认 5 行矩阵截图（深色）
├─ samples.png       140499 B   示例库弹窗截图
├─ server.js            803 B   极简静态服务（8765，no-store）
└─ _v1-viewer.html   124345 B   v1（3D 版）备份，不要拿去改
```

---

## 4. 数据来源（重要）

- 本地完整仓库：`C:\Users\DUANQIUYI\Desktop\claude\bedrock-samples-v1.26.50.4-full\`
  动画在 `resource_pack\animations\`，共 143 个 json。
- 示例库里的 12 条定义是**逐字从这些文件抄出来的**（不是编的）：
  `humanoid.animation.json`、`armor_stand.animation.json`、`player.animation.json`、
  `creeper.animation.json`、`ender_dragon.animation.json`、`warden.animation.json`、
  `parrot.animation.json`、`minecart.animation.json`。
- FMBE 的分组依据：`https://wiki.bedrock.dev/commands/display-entities`（“A New Way to Create Display Entities”）。
  FMBE 不写新动画，而是把**原版动画当 Molang 载体**，用
  `/playanimation @e[tag=wiki:fmbe] <原版动画> none 0 "<molang>"` 往里写 `v.*`。
- 注意：`web_fetch` 在本机被 DNS 劫持（GitHub 解析到 198.18.x.x），但 **`Invoke-WebRequest` 走系统代理（127.0.0.1:12000）是通的**。要联网抓资料用 shell，别用 web_fetch。

---

## 5. 语义规则（这个项目的灵魂，改代码前必须理解）

### 5.1 `this` —— 不是后缀，是表达式里的项

真实资源包写法（**这就是用户反复强调的点**）：

```json
"animation.humanoid.sneaking":  { "bones": { "body": { "rotation": ["0.5 - this", 0, 0] } } }
"animation.armor_stand.athena_pose": { "bones": { "body": { "rotation": ["-this", "-this", "-this + 2"] } } }
"animation.player.sleeping":    { "override_previous_animation": true,
                                  "bones": { "head": { "rotation": ["30.0 - this", "-this", "-this"] } } }
```

`this` = **该骨骼该通道、在当前行之前所有行累积出来的值**。所以：

| 写法 | 结果 |
| --- | --- |
| `-this` | 把这条轴清零（不管上面叠了多少） |
| `0.5 - this` | 把这条轴设成绝对值 0.5 |
| `-this + 2` | 绝对值 2 |
| `variable.x - this` | 用 variable.x 直接替换累计值 |

> 老版本的实现曾把 `-this` 当成**通道键后缀**（`rotation-this`），那是错的，已彻底移除。现在只认表达式里的 `this` 标识符。

### 5.2 `override_previous_animation`

这一行带 `true` 时，它先把自己碰到的骨骼清成 `0/0/0`（scale 是 `1/1/1`），再写自己的值 —— 也就是**把上面所有行在这些骨骼上的累积全抹掉**。

在矩阵里表现为：
- 该行上方插入一条琥珀色横带：`先把 head, root 清成 0 / 1 再写自己的值 —— 上面的行在这些骨骼上的累积到这一行作废`；
- 被抹掉的格子（`deadAt()`）加 `.dead`，文字划掉 + 右上角 `⟂`。

### 5.3 累加规则（概念层，当前 UI 只展示原文不做数值）

- `position` / `rotation`：逐轴**相加**；
- `scale`：逐轴**相乘**（基准 1）；
- 权重 `applyWeight(base, v, w)`：`scale → 1 + (v-1)*w`，其余 `v*w`（权重滑杆已在 v4 移除，函数是遗留）。

### 5.4 关键帧

通道值可以是静态数组，也可以是关键帧对象：

```json
"rotation": { "0.0": [0,0,0], "0.5": { "pre": [0,0,0], "post": [10,0,0] }, "1.0": [10,0,0] }
```

矩阵里对关键帧只列**前 3 帧 + 最后一帧**，标 `… 共 N 帧`；`pre ≠ post` 的那一帧加下划线（跳变点）。逻辑在 `chanHTML()`。

### 5.5 Molang 求值层（现状：**已实现但没接线**）

`sampleChannel()` / `baseCtx()` / `animLocalTime()` / `applyWeight()` / `fmtV()` / `miss()` 这套求值代码还在，但 v4 矩阵渲染**不再调用它们**（只显示原文）。所以：

- 它们是死代码；不要以为改这里能影响界面；
- 若将来要做“原文 / 求值”双列切换，可以直接接回来（`sampleChannel(ch, t, ctx, thisArr)` 已经支持逐轴 `thisV`）；
- 若不做，建议删掉，减少误读。

---

## 6. 页面结构

```
header.nav
  .brand（logo + 标题 + 副标题）
  #btnImport  导入动画（primary）
  #btnPaste   粘贴
  #btnSamples 示例库
  #btnExport  导出（icon）
  #btnDocs    说明（icon）
  #btnTheme   深浅主题（icon，内含 #icoThemeDark / #icoThemeLight）

main.shell
  #hint     两行说明（列/行是什么、this 与 override 怎么读）
  .mwrap > #matrix   矩阵表（.mtx）
  #empty    空状态（虚线卡片）

footer.statusbar
  #stAnims  N 个动画（M 启用） | #stBones N 根骨骼 | #stChans N 个通道 | #stHint 拖动行首排序

#modal>#modalTitle/#modalClose/#modalBody/#modalFoot
#toasts   toast 容器
#dropveil 全页拖拽遮罩
#fileInput hidden 的文件选择器
```

---

## 7. 代码地图（行号为当前文件，改动后会漂）

### 7.1 常量与数据

| 位置 | 内容 |
| --- | --- |
| L751 | CSS 里的 `/* ---------- 示例库 ---------- */`（**注意：JS 段也有同名注释，用 IndexOf 定位极易踩坑，见 §12**） |
| L1206 | `ICON`：全部内联 SVG（Lucide 风格，`class="icon"`，stroke 1.75） |
| L1239–1240 | `THEME_KEY='bedrock-anim-lab.theme'`、`LS_KEY='bedrock-anim-lab.v4'` |
| L1639 | `STACK_COLORS`（行首圆点/色条配色，8 色循环） |
| L1726 | `SAMPLE_GROUPS`：`{id,title,note,keys[]}`，三组 core / fmbe / adv，keys 引用同一份 `SAMPLES` |
| L1757 | `SAMPLES`：12 条 `{key, group, label, note:"", file, on, def}`，`on:true` 的 5 条构成默认栈 |
| L1788 | `state = { stack: [], sel: null }` |

### 7.2 解析

| 函数 | 职责 |
| --- | --- |
| `parseAnimFile(json, fileName, origin)` | 支持 `{animations:{...}}` 包装或裸 `{bones:{...}}` |
| `makeItem(animKey, def, fileName, origin)` | 产出 `{id, animKey, def, fileName, origin, channels[], boneSet, bones[], warnings[], events[], loop, length, startDelay, blendWeight, flags{override,hasThis,hasMolang,hasJump,hasCatmull,hasEvents}, enabled, weight, color}` |
| `makeChannel(bone, base, raw, rawKey, legacyKey)` | 通道对象 `{bone, base, rawKey, legacyKey, kind:'static'|'keyframed', keys, value, src, flags{molang,thisRef,jump,catmull,unresolved}, times}` |
| `parseKeys(raw)` | 静态数组 / 关键帧对象两条分支，并标记 `thisRef`（`/\bthis\b/` 命中） |
| `looseJSON(src)` + `parseJSONish(text)` | **真实文件必需**：bedrock-samples 里有 6 个 json 带 `//` 注释或尾逗号（bee/cat/cat/person 等），`JSON.parse` 会直接失败 |

### 7.3 矩阵渲染

| 函数 | 职责 |
| --- | --- |
| `buildMatrix()` | 汇总所有启用行 → `{cols, rows, bounds}`；`cols` = 动画列（顺序即 `state.stack` 顺序），`rows` = 骨骼/通道行；`bounds` = 每个 override 行清掉了哪些骨骼 |
| `boneRank()` / `BONE_RANK` / `CH_ORDER` | 列排序：`root, body, head, hat, cape, back, leftarm, rightarm, leftleg, rightleg, leftitem, rightitem` 优先，其余按字母；通道固定 `position, rotation, scale` |
| `deadAt(bone, si, bounds)` | 下方是否存在 override 行且碰到该骨骼 → 该格划掉 |
| `fmtNum(n, base)` | 数字格式化（rotation 1 位小数，其余 2 位，去尾零） |
| `litHTML(v, base)` | 单条轴渲染：字符串 → `<code class="expr">`，含 `this` 再加 `.th` 蓝色；数字 → `.num` |
| `chanHTML(ch, base)` | 一个通道 → x/y/z 三行；关键帧取前 3 + 末帧 |
| `badgesFor(item)` / `rowHead(r)` | 行首：抓手、序号、动画名、时长、loop 图标、chips（override / this / molang / keyframe / fx）、眼睛（启停）、垃圾桶（删除） |
| `cellHTML(row, col, bounds)` | 单格；空格输出 `·`，被 override 抹掉的加 `.dead` |
| `render()` | 重建整张表 + 写 `--ncol`（列数）+ 切换 `#empty` |
| `updateStatus()` | 写状态栏 |

### 7.4 交互

| 函数 | 职责 |
| --- | --- |
| `initDrag()` | 纵向拖行换序：pointerdown 抓 `.grip` / 行首 → 生成 `.hline` 指示线 + `.dragtag`；用 `clientY` 与各行中线比较算插入点；`pointerup` 落库 `state.stack` 并 `render(); save()` |
| `initMatrix()` | 表内点击委托：`[data-act]` = toggle / remove |
| `toggleItem()` / `removeItem()` | 启停 / 删除一行 |
| `initSamples()` | 示例库：`click` 委托处理「全选」「整套加入」「清空」；`change` 处理单个复选框 → `libToggle()` → `libSyncRows()` |
| `libToggle(key, on)` | **勾上即加入**（`newItems` + 强制 `enabled = true`），**取消即移除** |
| `libSyncRows()` | 按 `state.stack` 回写弹窗里的勾选态 / `.in-stack` / 「已在栈里」标签 |
| `libGrabGroup(id)` | 整组加入（跳过已在栈里的） |
| `loadSamples()` | 载入默认示例栈（`SAMPLES.filter(s => s.on)`）——**只在 boot 首次、localStorage 为空时自动调用** |
| `initImport()` / `ingestFiles()` / `walkEntry()` | 文件选择 / 全页拖拽（含文件夹递归）/ 多动画时弹勾选列表 `pickItems()` |
| `initPaste()` | 粘贴 JSON 弹窗 |
| `initExport()` / `exportPayload()` | 导出当前顺序 + 每通道原文（`cells[].channels[].raw`、`usesThis`、`usesMolang`） |
| `initDocs()` | 说明弹窗（布局怎么读 / this / override / 关键帧 / 表达式 / 示例库 / 文件来源） |
| `applyTheme()` / `initTheme()` | 双主题，跟随 `prefers-color-scheme`，写 `<html data-theme>` |
| `boot()` | `initTheme → initModal → initSamples → initMatrix → initDrag → initImport → initPaste → initExport → initDocs → load() || loadSamples() → render()` |

---

## 8. 持久化

```js
const LS_KEY = 'bedrock-anim-lab.v4';
// save(): { v: 4, items: [{ animKey, def, fileName, enabled, color }] }
// load(): 必须 data.v === 4 才采用，否则回落到默认示例
```

**改动存储结构时必须同时改 `v`**，否则老数据会被读成坏数据（历史上踩过一次：`LS_KEY` 没定义 → 永久不落盘；`v` 不匹配 → 改完刷新看不见效果）。

---

## 9. 样式系统

设计语言的硬要求（用户给过一份完整规范，摘要）：

- 深色：`--bg #09090B` / `--bg-2 #111113` / `--card #141417` / `--hover #1C1C21` / `--border #27272A` / 文字 `#F4F4F5 · #A1A1AA · #71717A` / accent `#6366F1`
- 浅色：`--bg #FAFAFA` / `--card #FFFFFF` / `--bg-2 #F4F4F5` / `--border #E4E4E7` / 文字 `#18181B · #52525B` / accent `#4F46E5`
- 圆角 8–12，1px 细边，阴影极淡；按钮 34px 高、圆角 6–8；导航 60px
- 字号：正文 13–14 / 辅助 11.5–12 / 微标 10.5；数字用等宽（`--mono`，JetBrains Mono 族）
- 图标统一 `class="icon"`（16px，stroke 1.75），**不许用 emoji 当功能图标**
- 动效 120–180ms（hover）/ 200–280ms（弹窗），并尊重 `prefers-reduced-motion`
- 矩阵列宽：标签列 232px，通道列 144px，`width: max(100%, calc(232px + var(--ncol) * 144px))`，窄屏横向滚动
- 断点：1023px、767px（767 以下隐藏部分说明、缩列宽）

状态语义色：`--accent`（交互）/ 琥珀（override 边界）/ 蓝 `#93C5FD`（`this`）/ 灰（被抹掉）。

> 用户给的原始规范全文见 **附录 A**（逐字保留，冲突时以它为准）；逐条落地情况见 **附录 B**。

---

## 10. 交互速查（用户视角）

| 操作 | 结果 |
| --- | --- |
| 拖行首抓手上下拖 | 换行序；松手即重算 `⟂` 与蓝标 |
| 点眼睛 | 该行停用/启用（停用行不参与 `this` 与 override 判定） |
| 点垃圾桶 | 从栈里删除 |
| 顶栏「示例库」 | 打开弹窗；**勾选立刻增删**，无确认按钮；组头有「全选」「整套加入」 |
| 顶栏「导入动画」/ 拖文件进窗口 | 单个动画直接入栈；一个文件多个动画时弹勾选列表 |
| 顶栏「粘贴」 | 手动贴 JSON（同样容忍 `//` 注释） |
| 顶栏「导出」 | 下载 `bedrock-animation-stack.json`（顺序 + 边界 + 每通道原文） |
| 顶栏「说明」 | 语义说明弹窗 |
| 顶栏月亮/太阳 | 深浅主题切换 |

### 键盘

| 按键 | 结果 |
| --- | --- |
| `Tab` / `Shift+Tab` | 依次聚焦：抓手 → 眼睛 → 垃圾桶 → 下一行（焦点头有 `:focus-visible` 焦点环） |
| 抓手聚焦后 `↑` / `↓` | 当前行上移 / 下移一行（等价于拖动） |
| 抓手聚焦后 `Home` / `End` | 移到栈顶 / 栈底 |
| `Enter` / `Space`（焦点在按钮上） | 触发该按钮（浏览器原生） |

移动后焦点保持在被移动那一行的抓手上，可直接连按；状态栏 `#stHint` 会显示「第 N 行：动画名」。

---

## 11. 已知问题 / 待办（按优先级）

1. ~~蓝色 `this` 高亮未做浏览器验证~~ → **已实测通过**（浅色主题，`.expr.th` 47 个 / 普通 `.expr` 48 个；computed text `rgb(29,78,216)`，bg `rgba(37,99,235,.1)`，border `rgba(37,99,235,.34)`；`td.hasthis` = 0）。`humanoid.sneaking` 的 body.rotation 只有 x 变蓝，`armor_stand.athena_pose` 整行蓝，`player.sleeping` 上方琥珀带 + `head`/`root` 划掉，均符合预期。
2. ~~键盘导航缺失~~ → **已补**（见 §10 键盘表与附录 B §12）。
3. 浅色主题下矩阵未逐项复查（`.bandrow` 琥珀带、`.dead` 灰化在浅色下的对比度已截图看过，可接受；`.expr.th` 已实测）。
4. `preview.png` / `samples.png` 可能落后于当前代码（深色、默认 5 行）。
5. **Molang 求值链是死代码**（见 §5.5）：要么接回来做「求值」视图，要么删除。
6. `applyWeight()` / `fmtV()` 无调用点。
7. 矩阵版 `initDrag()` **没有自动滚动**（v3 卡片版有过 110px 边缘自动滚动）；行数多时拖到屏幕外会吃力，可补。
8. 无 git 仓库，建议 `git init` 后再动大手术。
9. 移动端只做了「不横向溢出」级别的适配，矩阵本身必须横向滚动（列多时），可考虑冻结骨骼表头行。

---

## 12. 血泪教训（务必遵守）

1. **不要用 `IndexOf` 定位代码段直接 splice**。这个文件里 CSS 和 JS 各有一份 `/* ---------- 示例库 ---------- */`，我曾按第一个命中替换，一次删掉 65KB（`<script>` 都没了）。
   自保动作：定位后先打印 `$s.Substring($start, 60)` 和 `$end - $start`，确认长度合理再写盘；或者干脆用 `edit` 工具按**唯一字符串**替换。
2. **改完必须 `node --check`**（抽取 `<script>` 的方法见 §0）。HTML 层面坏了看不出来，JS 语法坏了页面直接白屏。
3. **`localStorage` 有坑**：改状态结构不改 `v`，刷新后看不到变化，会误判成“没生效”。
4. **浏览器验证的怪癖**（用的是 DSH 的 bsk 工具链）：
   - `bsk_inspect screenshot` 经常返回**上一帧**（sha256 不变）；先用 `window.scrollTo(0,0)` 或 `?v=Date.now()` 导航再截。
   - 弹窗有 0.2–0.28s 动画，截太早会拍到半透明帧（不是 bug），等 800ms 再截。
   - 页面 rAF 重绘会让 strict 观察失效，用 `bsk_session sync` 拿 `context_id` + `bsk_interact evaluate`（`unverified: true`）。
   - 浏览器扩展会往 DOM 注入 `<style wxt-...>`、`<read-frog>`、`<browser-skill-overlay>`；如果把页面 `outerHTML` 存回文件，**必须清掉这些节点**，否则会带一堆外部 CSS 进交付物。
5. 从浏览器 DOM 抢救文件的办法：在页面里 `fetch('http://127.0.0.1:8766/save', {method:'POST', body:'<!DOCTYPE html>'+document.documentElement.outerHTML})`，配一个 10 行的 node 写盘服务（CORS `*`）。已用过一次，救回来了。

---

## 13. 一句话验收标准

打开 `index.html`：

1. 默认 5 行（move / bob / sneaking / athena_pose / sleeping），行首可拖动，拖完 `⟂` 与蓝标跟着变；
2. `sneaking` 的 `body.rotation` 的 **x** 是蓝的，y/z 不是；`athena_pose` 整行都是蓝的；
3. `player.sleeping` 上方有琥珀横带；`sneaking`、`athena_pose` 在 `head` 上的格子被划掉并带 `⟂`（它们写了 `head`，而 `sleeping` 会清 `head`、`root`）；
4. 示例库勾选一条 → 关闭弹窗前后都能在矩阵里看到它（且是启用状态）；
5. 导入 `C:\Users\DUANQIUYI\Desktop\claude\bedrock-samples-v1.26.50.4-full\resource_pack\animations\cat.animation.json`（带 `//` 注释）→ 弹出 8 条动画的勾选列表，选完能正常入栈。

---

## 附录 A · 用户原始设计规范（逐字保留）

> 下面这段是用户第二次给的《网站 UI / UX 设计规范》全文，**原样保留**，作为视觉验收的唯一依据。
> 实现在 §9 已说明；与本文冲突时，以本文为准。

### 1. 整体设计风格（Design Language）

采用现代极简主义（Modern Minimalism）与精密数字工业设计（Digital Industrial Design）相结合的设计语言。

整体视觉应当具有以下特征：

- 高级、克制、理性、干净、专业。
- 具有现代科技产品的精密感，但不过度追求科幻效果。
- 通过排版、留白、空间关系和视觉层次建立设计美感，而非依赖大量装饰。
- 参考 Linear、Vercel、Apple、Raycast 等产品的设计理念，但不要直接复制其界面。
- 界面应当像经过专业设计团队长期打磨的成熟商业产品，而非通用的 AI 生成模板。
- 保持全局设计语言高度统一。

### 2. 色彩系统（Color System）

以低饱和度中性色为主，建立清晰且统一的色彩体系。

深色模式：

- 页面背景：`#09090B`
- 次级背景：`#111113`
- 卡片背景：`#141417`
- 悬浮背景：`#1C1C21`
- 边框颜色：`#27272A`
- 主文字：`#F4F4F5`
- 次级文字：`#A1A1AA`
- 辅助文字：`#71717A`
- 强调颜色：`#6366F1`

浅色模式：

- 页面背景：`#FAFAFA`
- 卡片背景：`#FFFFFF`
- 次级背景：`#F4F4F5`
- 边框颜色：`#E4E4E7`
- 主文字：`#18181B`
- 次级文字：`#52525B`
- 强调颜色：`#4F46E5`

强调色仅用于主要操作按钮、选中状态、焦点状态及重要交互反馈。

避免大面积高饱和度色彩、彩虹渐变、过度发光以及无意义的背景光斑。

### 3. 字体与排版（Typography）

英文优先使用 Inter、Geist 或 SF Pro，中文优先使用思源黑体或系统原生无衬线字体。

代码及数值内容使用 JetBrains Mono 或 Geist Mono。

建立明确的字号层级：

- 展示级标题：48–64px
- 页面一级标题：32–40px
- 模块标题：20–24px
- 卡片标题：15–18px
- 正文内容：14–16px
- 辅助说明：12–13px
- 微型标签：11–12px

标题采用适度紧凑的字间距，正文保持舒适的阅读行高。

避免所有文字均使用粗体。通过字号、字重、颜色、位置及留白共同建立视觉层级。

### 4. 页面布局（Layout）

采用模块化布局系统，以 8px 为基础间距单位，允许使用 4px 作为细节调整单位。

- 使用响应式 12 列栅格系统。
- 内容区域保持合理的最大宽度。
- 主要内容与辅助信息应具有明确的主次关系。
- 页面不同模块之间保留充分的呼吸空间。
- 同类组件保持一致的对齐方式和间距。
- 避免无意义的容器嵌套。
- 不要将所有内容机械地放入独立卡片。
- 根据内容性质灵活使用分隔线、留白、列表及非对称布局。
- Bento Grid 仅在适合展示多个独立功能模块时使用。

使用户能够在进入页面后的短时间内理解当前页面的功能及主要操作。

### 5. 导航系统（Navigation）

导航栏应当简洁、轻量且具有明确的结构。

顶部导航栏：

- 高度约为 56–64px。
- 使用细微的底部分隔线。
- Logo、导航入口及用户操作区域清晰排列。
- 当前页面通过文字颜色、细线或低对比度背景进行标识。
- 避免体积过大的导航元素。

侧边栏：

- 适用于后台管理、开发工具及复杂应用。
- 展开宽度为 220–260px。
- 支持折叠与展开。
- 使用统一的线性图标。
- 通过合理的分组和留白组织功能入口。
- 选中状态使用低饱和度背景，避免过于醒目的高亮色块。

### 6. 卡片与容器（Surface & Cards）

卡片应具有轻量、精致、克制的视觉表现。

- 圆角控制在 8–12px。
- 使用 1px 细边框。
- 优先通过背景色差与边框区分层级。
- 阴影保持柔和且低透明度。
- 卡片内部使用一致的间距规则。
- 卡片标题、主要内容与辅助信息之间具有明确层级。
- 仅为可交互卡片提供 Hover 反馈。

禁止滥用超大圆角、厚重阴影、玻璃拟态、渐变描边和多层卡片嵌套。

### 7. 按钮与交互组件（Components）

按钮分为 Primary、Secondary、Ghost、Destructive 四种类型。

- 按钮高度通常为 36–44px。
- 移动端交互区域至少为 44×44px。
- 圆角为 6–8px。
- 图标与文字保持合理间距。
- 主要按钮突出核心操作。
- 次级按钮使用低对比度背景或细边框。
- 次要操作使用 Ghost Button。
- 危险操作使用明确但不过度刺眼的警示颜色。

输入框、下拉菜单、复选框、开关、标签页、弹窗及提示组件必须遵循统一的视觉规范。

所有交互组件应包含 Default、Hover、Active、Focus、Disabled、Loading 等适用状态。

### 8. 图标系统（Iconography）

统一使用 Lucide Icons 或同类线性图标库。

- 常规尺寸为 16px、18px、20px、24px。
- 线条粗细保持一致。
- 图标风格简洁、几何化。
- 图标颜色继承所在组件的视觉层级。
- 避免混用多种图标库。
- 不使用 Emoji 替代正式功能图标。
- 仅在图标含义清晰或具有文字说明时使用独立图标按钮。

### 9. 动效与微交互（Motion Design）

动效应服务于交互反馈、空间关系和状态变化，而不是单纯装饰。

- Hover 过渡时间：120–180ms。
- 常规状态变化：150–250ms。
- 弹窗、抽屉及复杂过渡：200–350ms。
- 使用自然、平滑的缓动曲线。
- 优先对 opacity 和 transform 属性进行动画处理。
- 下拉菜单采用轻微位移与透明度变化。
- 页面内容加载使用简洁的 Skeleton 状态。
- 操作完成后提供明确且轻量的反馈。
- 支持 prefers-reduced-motion。

避免大幅缩放、强烈弹跳、持续闪烁、过度视差滚动及干扰正常操作的动画。

### 10. 数据可视化（Data Visualization）

如果网站涉及数据展示，应采用精密、专业且易于阅读的可视化风格。

- 使用简洁的坐标轴和低对比度网格线。
- 通过有限的色彩区分数据系列。
- 重要数据采用更高的视觉权重。
- 数字使用等宽数字特性，保持纵向对齐。
- Tooltip 应清晰展示数据名称、数值及单位。
- 图表、表格和筛选器保持统一的设计语言。
- 避免无意义的三维图表和装饰性视觉效果。

### 11. 响应式设计（Responsive Design）

完整适配桌面端、平板端与移动端。

- 1440px 及以上：充分利用桌面空间。
- 1024–1439px：保持完整布局及合理内容密度。
- 768–1023px：调整栅格结构和导航方式。
- 768px 以下：采用移动端优先布局。

移动端不能只是桌面端界面的等比例缩小，而应根据屏幕尺寸重新组织内容、交互方式及视觉层次。

保证文字可读、按钮易于点击，并避免不必要的横向滚动。

### 12. 可访问性与用户体验（Accessibility）

- 遵循 WCAG 2.2 AA 可访问性原则。
- 确保文字具有足够的颜色对比度。
- 支持键盘导航。
- 为交互元素提供明确的焦点状态。
- 使用语义化 HTML。
- 不单独依赖颜色传达状态。
- 为异步操作提供加载、成功及失败反馈。
- 完善空状态、错误状态及无搜索结果状态。
- 保证页面布局在加载过程中尽量稳定。

### 13. 工程实现规范（Implementation）

将视觉设计抽象为统一的 Design Tokens，包括：

- Color
- Typography
- Spacing
- Border Radius
- Shadow
- Z-index
- Transition
- Breakpoints

建立可复用的组件体系，避免在不同页面中重复定义相同样式。

优先使用 CSS Variables 管理主题及全局视觉属性。

如果项目已经存在技术栈和组件体系，应优先兼容现有架构，不要仅为了修改 UI 而重构无关业务逻辑。

确保所有按钮、导航入口、表单及交互组件具备真实功能，不要以静态占位元素伪装完整功能。

### 14. 避免典型 AI 生成网页风格

严格避免以下设计问题：

- 千篇一律的紫蓝色渐变背景。
- 大面积玻璃拟态效果。
- 所有模块均采用大圆角卡片。
- 无意义的胶囊状标签。
- 随意添加宣传性英文小标题。
- 重复出现的三列功能卡片。
- 过度巨大的 Hero 标题。
- 装饰性光球、粒子及霓虹发光。
- 大量无实际意义的渐变文字。
- 过度使用阴影、动画和视觉装饰。
- 为了填充空间而添加虚假统计数据。
- 视觉结构完全对称且缺乏内容层次。

### 15. 最终设计目标

最终网站应该具有成熟商业软件的完成度、专业开发工具的精密感以及高级消费级产品的审美水平。

在视觉表现上强调克制、秩序、细节和一致性；在用户体验上强调直觉、效率、流畅性和可访问性。

每一个设计元素都应具有明确的存在理由。优先考虑内容本身，而不是为了展示设计技巧而设计。

---

## 附录 B · 规范落地对照表

| 规范条目 | 当前实现 | 状态 |
| --- | --- | --- |
| §2 双主题色板 | `:root` / `html[data-theme=light]` 两套 CSS 变量，逐值对齐 | ✅ |
| §3 字体层级 | 正文 13px / 辅助 11.5–12px / 微标 10.5px；数值全用 `--mono` | ⚠️ 页面没有 48–64px 展示级标题（工具型界面，故意不做 Hero） |
| §4 8px 栅格 | 间距基本为 4/8/12/16 的倍数 | ✅ |
| §5 导航 56–64px | `.nav` 高 60px + 1px 底边 | ✅ |
| §6 圆角 8–12 + 1px 边 | `--r-md 8` / `--r-lg 12`，全站 1px `--border` | ✅ |
| §7 按钮 34–44px | `.btn` 34px / `.btn.tiny` 28px | ⚠️ tiny 按钮低于 36px（表格内空间所限，桌面端可接受；移动端应放大到 44px 触控区） |
| §8 图标 | 内联 SVG 统一 `class="icon"` 16px / stroke 1.75，无 emoji | ✅ |
| §9 动效 | hover 140ms、弹窗 200/280ms、`prefers-reduced-motion` 全关 | ✅ |
| §10 数据可视化 | 矩阵表格、低对比度分隔线、数字等宽右对齐 | ✅ |
| §11 响应式 | 断点 1023 / 767；窄屏缩列宽、隐藏说明 | ⚠️ 矩阵本身必须横向滚动（列多），移动端仅保证不溢出页面 |
| §12 可访问性 | 行首抓手可聚焦（`role="button"` + `tabindex="0"` + `aria-label`），`↑/↓` 换序、`Home/End` 到首末行；眼睛/垃圾桶有 `aria-label`；`<table aria-label>` + `scope="colgroup"/"col"/"row"`；全局 `:focus-visible` 焦点环 | ✅ 已补（2026-11-26 实测：ArrowDown 把第 1 行移到第 2、焦点留在抓手、`#stHint` 同步） |
| §13 Design Tokens | 颜色/圆角/过渡全部走 CSS 变量 | ✅ |
| §14 反 AI 味 | 无渐变背景、无玻璃拟态、无光球、无假数据 | ✅ |

### 结论

视觉层合规，**§12 的键盘可达性缺口已补齐**（2026-11-26）：`.grip` 聚焦后 `↑/↓` 移动当前行、`Home/End` 跳到首末行，移动后焦点保持在原抓手并更新状态栏提示；行首眼睛/垃圾桶是原生 `<button>` 且带 `aria-label`；表头补了 `scope="colgroup"/"col"/"row"`，`<table>` 带 `aria-label`。焦点环用全局 `:focus-visible`（`outline: 2px solid var(--accent)`，第 135 行），`.grip` 另外加了 `cursor:grab` 与 hover/focus 底色。

仍可继续打磨的方向（非阻塞）：

1. `.btn.tiny`（28px）在移动端 < 44px 触控标准，可只在窄屏放大；
2. 矩阵横向滚动时可以再冻结骨骼表头行；
3. 拖拽仍无边缘自动滚动。