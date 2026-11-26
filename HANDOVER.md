# Bedrock Animation Stack · 接手指南

> 这份文档写给接手的人（或模型）。目标：读完就能改，不用重新猜需求。
> 最后更新：**2026-11-26 15:52（接手第五次改动）** · `index.html` **2986 行 / 120,631 B**，`node --check` 通过。
> 仓库已 `git init`（分支 `main`），接手前的基线是 commit `0eeaf25`，用 `git diff 0eeaf25` 就能看到我改了什么。

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
| 琥珀色横带 | 这一行带 `override_previous_animation: true`，它把上面这些行在它碰到的骨骼上全抹掉。**该行被眼睛停用时横带不出现**（见 §5.2） |
| 格子划掉 + `⟂` | 这一格被下面的某一行 override 抹掉了（`⟂` 在格子**右下角**，不是右上角） |
| 单轴划掉 + 右侧小 `⟂` | 这一条轴被下面的 `- this` 写成了绝对值，上面的累积不算了（三条轴都这样时按整格划掉，只画一个 `⟂`） |
| 收起的骨骼列 | 点骨骼表头可把该骨骼的通道列收起来，矩阵上方出现「已隐藏 N 根骨骼」+ 恢复按钮；**只影响显示，不参与判定** |
| 斜纹格 | 这段动画**压根没有**这根骨骼（`·` 则是「有骨骼但没写这个通道」）；行头 `骨骼 4/8` 是覆盖率，悬停看具体漏了哪几根 |

**格子里的值一律完整显示**：表达式换行不截断，关键帧一帧一行全列（2026-11-26 改，之前是省略号 + 只列前 2 帧）。代价是行高会随内容变高（`player.sleeping` 的 root.position 那种一行 140 多字符的会撑到 300px 上下）。

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
D:\scripts\bedrock-anim-lab\        ← 已是 git 仓库（main）
├─ index.html        120631 B  ← 全部交付物（HTML+CSS+JS 内联）
├─ preview.png       145278 B  默认 5 行矩阵截图（深色，1848×857，2026-11-26 重拍：表格铺满到状态栏、覆盖率 + 斜纹 + -this 划掉）
├─ samples.png       144938 B  示例库弹窗截图（深色，含「替换整栈」按钮）
├─ server.js            803 B  极简静态服务（8765，no-store）
├─ _v1-viewer.html   124345 B  v1（3D 版）备份，不要拿去改
├─ HANDOVER.md                 本文
├─ .gitignore / .gitattributes 忽略 _check.js / _backup；**关掉一切行尾转换**（`* -text`）
└─ _backup/index-2026-11-26-1401.html   改 bug 前的手工备份（与基线 commit 同内容）
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
- **组内顺序 = 该页 `render.mcfunction` 里 `/playanimation` 的先后**，一条都不能乱：
  `player.sleeping → creeper.swelling → ender_dragon.neck_head_movement → warden.move → player.attack.rotations → parrot.moving → minecart.move → parrot.dance`。
  之前把 `parrot.dance` 排在 `minecart.move` 前面（错：zpos 会抢在 ypos 之前算），2026-11-26 已改正。
- wiki 里写的是 `animation.minecart.move.v1.0`，但 bedrock-samples v1.26.50.4 的 `minecart.animation.json` 里键名就是 `animation.minecart.move`（无 `.v1.0`）——**以文件为准**，别照 wiki 改名。
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

**UI 怎么标「被它抹掉」**（2026-11-26 加）：只要某个轴上的 `this` 是**系数 -1 的线性项**，这一轴就等于写成绝对值，上面所有行在这一轴上的累积都不再计入 → 上面的格子划掉。

判定在 `axisCancels()`，用压掉空白后的正则 `/(^|[^*/])-this(?!\*|\/)/`：

| 表达式 | 算不算「写成绝对值」 |
| --- | --- |
| `-this` / `0.5 - this` / `30.0 - this` / `variable.x - this` | ✅ 算 |
| `-this * 0.5` / `2 * -this` / `this * 0.5` / `this + 5` | ❌ 不算（还依赖累计值） |

`buildMatrix()` 会算出 `shadow`（`bone|base|ax → 最后一个抵消行的 si`），`cellHTML()` 拿它决定哪几轴画成 `.cut`。所以 `athena_pose` 会把 `move`/`bob` 的四肢旋转划掉 —— 那是真的被它写成绝对值了，不是渲染错误。

### 5.2 `override_previous_animation`

这一行带 `true` 时，它先把自己碰到的骨骼清成 `0/0/0`（scale 是 `1/1/1`），再写自己的值 —— 也就是**把上面所有行在这些骨骼上的累积全抹掉**。

在矩阵里表现为：
- 该行上方插入一条琥珀色横带（原文）：`上面的行在 head, root 上的累积到这一行作废 —— 下面几行从这个新起点重新叠`；
- 被抹掉的格子（`deadAt()`）加 `.dead`，文字划掉（透明度 .42）+ **右下角** `⟂`。

**停用（点眼睛）时不生效**：横带不出现，上方格子也不再划掉。实现上 `buildMatrix()` 给每行算了两个位：
`ov`（这段动画是否带 override，身份，停用也还是）与 `ovOn`（此刻是否真的在生效 = `ov && enabled`）；
`bounds` 只收 `ovOn` 的行，横带也用 `if (r.ovOn)`。改这里时两个位别混用 —— 2026-11-26 之前只判了 `ov`，
导致「停用 override 行后横带照画、下面照划」，与「停用行不参与判定」自相矛盾。

### 5.3 累加规则（概念层，当前 UI 只展示原文不做数值）

- `position` / `rotation`：逐轴**相加**；
- `scale`：逐轴**相乘**（基准 1）；
- 权重 `applyWeight(base, v, w)`：`scale → 1 + (v-1)*w`，其余 `v*w`（权重滑杆已在 v4 移除，函数是遗留）。

### 5.4 关键帧

通道值可以是静态数组，也可以是关键帧对象：

```json
"rotation": { "0.0": [0,0,0], "0.5": { "pre": [0,0,0], "post": [10,0,0] }, "1.0": [10,0,0] }
```

矩阵里对关键帧：**一帧一行全部列出，不折叠**（2026-11-26 起；之前只列前 2 帧 + 末帧，中间用 `… 共 N 帧` 吞掉）。时间标在行首，`pre ≠ post` 的那一帧标琥珀色（跳变点）。逻辑在 `chanHTML()`：

```js
/* 关键帧：全部列出，一帧一行，不折叠 */
return '<div class="keys">' + ch.keys.map(line).join('') + '</div>';
```

真实数据里 88% 的通道 ≤ 8 帧，但 bedrock-samples 里最长有 **52 帧**（`bee.animation.json` 的 `leftwing_bone / rotation`）。全列会让那一行很高 —— 这是「完整优先」的代价，别为了好看又加回 slice。

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

body（flex column，`overflow: hidden` —— 整页不滚，只有 .mwrap 滚）
  header.nav
  main.shell（flex: 1 1 auto; min-height: 0; 自身也是 flex column）
    #hint     两行说明（列/行是什么、this 与 override 怎么读、点骨骼表头可收起列）· flex: none
    #colbar   已隐藏骨骼提示栏（默认 hidden：`已隐藏 N 根骨骼` + 骨骼 chip + 全部显示）· flex: none
    .mwrap > #matrix   矩阵表（.mtx）· flex: 1 1 auto; min-height: 0; overscroll-behavior: contain
    #empty    空状态（虚线卡片）· flex: 1 1 auto（空栈时 .mwrap 一起 hidden，否则会多出一块空边框）

footer.statusbar（fixed 40px，所以 .shell 底部留 56px）
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
| L801 | CSS 里的 `/* ---------- 示例库 ---------- */`（**注意：JS 段也有同名注释，用 IndexOf 定位极易踩坑，见 §12**） |
| L1267 | `ICON`：全部内联 SVG（Lucide 风格，`class="icon"`，stroke 1.75） |
| L1300–1301 | `THEME_KEY='bedrock-anim-lab.theme'`、`LS_KEY='bedrock-anim-lab.v4'` |
| L1701 | `STACK_COLORS`（行首圆点/色条配色，8 色循环） |
| L1788 | `SAMPLE_GROUPS`：`{id,title,note,keys[]}`，三组 core / fmbe / adv，keys 引用同一份 `SAMPLES` |
| L1819 | `SAMPLES`：12 条 `{key, group, label, note:"", file, on, def}`，`on:true` 的 5 条构成默认栈 |
| L1850 | `state = { stack: [], sel: null, hiddenBones: [] }` |

> 上面的行号是 2026-11-26 14:52 的；文件一改就会漂，**优先按名字搜**。

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
| `buildMatrix()` | 汇总 → `{rows, groups, cols, bounds, allBones}`。`rows` = 动画行（顺序即 `state.stack`），`groups` = 骨骼分组（**已按 `state.hiddenBones` 过滤**），`cols` = 骨骼 × 通道列，`bounds` = 生效中（`ovOn`）的 override 行，`allBones` = 数据里出现的全部骨骼（给恢复栏用，不受隐藏影响） |
| `boneRank()` / `BONE_RANK` / `CH_ORDER` | 列排序：`root, body, head, hat, cape, back, leftarm, rightarm, leftleg, rightleg, leftitem, rightitem` 优先，其余按字母；通道固定 `position, rotation, scale` |
| `deadAt(bone, si, bounds)` | 下方是否存在生效中的 override 行且碰到该骨骼 → 该格划掉 |
| `axisCancels(ch, ax)` / `CANCEL_THIS` | 这一轴的值是不是「被 `- this` 写成绝对值」（系数 -1、没被乘除）→ 供 `shadow` 用 |
| `fmtNum(n, base)` | 数字格式化（rotation / 时间 2 位，其余 3 位，去尾零） |
| `litHTML(v, base)` | 单条轴渲染：字符串 → `<code class="expr">`，含 `this` 再加 `.th` 蓝色；数字 → `.num` |
| `chanHTML(ch, base, cuts)` | 一个通道 → x/y/z 三行；`cuts[ax]` 是抵消行号时该轴加 `.cut` + 右侧小 `⟂`；关键帧**全部列出**、一帧一行（不再 slice），整通道被抵消时加 `.dead-keys` |
| `badgesFor(item)` / `rowHead(r, allBones)` | 行首：抓手、序号、动画名、时长、**覆盖率 `骨骼 4/8`**（title 列涉及/未涉及）、loop 图标、chips（override / this / molang / keyframe / fx）、眼睛（启停）、垃圾桶（删除） |
| `cellHTML(row, col, bounds, shadow)` | 单格：没有该通道时输出 `·`（`.none`）或斜纹（`.bnone`，连骨骼都没有）；有值时按 override（整格 `.dead`）与 `shadow`（逐轴 `.cut`）上标记 |
| `render()` | 重建整张表 + 写 `--ncol`（列数）+ 切换 `#empty` / `#mwrap`（空栈时两个都 hidden）+ 调用 `renderColbar()` |
| `renderColbar(allBones)` | 画「已隐藏 N 根骨骼」栏：只列当前数据里真的存在、且被隐藏的骨骼；每个是一个 `data-show` 按钮，右侧「全部显示」 |
| `hideBone(bone)` / `showBone(bone)` | 隐藏 / 恢复单根骨骼的列，`render(); save()`，隐藏时 toast 提示去哪恢复 |
| `updateStatus()` | 写状态栏（**只统计启用行的骨骼/通道，不受隐藏影响**） |

### 7.4 交互

| 函数 | 职责 |
| --- | --- |
| `initDrag()` | 纵向拖行换序：**整块行头 `th.rowh` 都能起手**（除了眼睛/垃圾桶），先过 5px 阈值、且竖向位移要大于横向才进拖拽（轻微抖动 / 横向滑动不算），之后生成 `.hline` 指示线 + `.dragtag`；用 `clientY` 与各行中线比较算插入点；`pointerup` 落库 `state.stack` 并 `render(); save()`。拖完 300ms 内的 click 被 `lastDragAt` 吞掉，避免顺带把行点开/关 |
| `initMatrix()` | 表内点击委托：点骨骼表头 → `hideBone()`；`[data-act]` = toggle / remove。键盘：骨骼表头聚焦后 Enter / 空格隐藏，抓手聚焦后 ↑↓ / Home / End 换序 |
| `toggleItem()` / `removeItem()` | 启停 / 删除一行 |
| `initColbar()` | 「已隐藏」栏点击委托：`[data-show]` → `showBone()`，`#colShowAll` → 清空 `state.hiddenBones` |
| `initHistory()` / `undo()` / `redo()` | Ctrl+Z / Ctrl+Y（含 Ctrl+Shift+Z）+ 顶栏两个箭头。快照存在 `hist.{past,future}`，只记 item 引用 + 启停 + `hiddenBones`，上限 80 步；输入框内不抢浏览器自己的撤销 |
| `histPush(label)` / `histApply(snap)` / `syncHistUI()` | 每个会改栈的动作在动手**之前** push 一条带中文 label 的快照；`histApply` 回放时 `assignColors → render → save → libSyncRows`（弹窗开着也同步勾选） |
| `initSamples()` | 示例库：`click` 委托处理「全选」「替换整栈」「清空」；`change` 处理单个复选框 → `libToggle()` → `libSyncRows()` |
| `libToggle(key, on)` | **勾上即加入**（`newItems` + 强制 `enabled = true`），**取消即移除** |
| `libSyncRows()` | 按 `state.stack` 回写弹窗里的勾选态 / `.in-stack` / 「已在栈里」标签 |
| `libGrabGroup(id)` | **替换整栈**：`state.stack = ` 这一组（顺序 = `g.keys` 顺序），其他动画全删；`assignColors()` 重排配色；toast 报「移除 N 个」。（旧行为是「追加缺的」，2026-11-26 按用户要求改成替换） |
| `loadSamples()` | 载入默认示例栈（`SAMPLES.filter(s => s.on)`）——**只在 boot 首次、localStorage 为空时自动调用** |
| `initImport()` / `ingestFiles()` / `walkEntry()` | 文件选择 / 全页拖拽（含文件夹递归）/ 多动画时弹勾选列表 `pickItems()` |
| `initPaste()` | 粘贴 JSON 弹窗 |
| `initExport()` / `exportPayload()` | 导出当前顺序 + 每通道原文（`cells[].channels[].raw`、`usesThis`、`usesMolang`） |
| `initDocs()` | 说明弹窗（布局怎么读 / this / override / 关键帧 / 表达式 / 示例库 / 收起骨骼列 / 文件来源） |
| `applyTheme()` / `initTheme()` | 双主题，跟随 `prefers-color-scheme`，写 `<html data-theme>` |
| `boot()` | `initTheme → initModal → initSamples → initMatrix → initColbar → initDrag → initImport → initPaste → initExport → initDocs → load() || loadSamples() → render()` |

---

## 8. 持久化

```js
const LS_KEY = 'bedrock-anim-lab.v4';
// save(): {
//   v: 4,
//   items: [{ animKey, def, fileName, enabled, color }],
//   hiddenBones: ['head', 'root']     // 可选；纯显示偏好
// }
// load(): 必须 data.v === 4 才采用，否则回落到默认示例
//         hiddenBones 缺失 / 不是数组都按 [] 处理（老数据照样能读）
```

**改动存储结构时必须同时改 `v`**，否则老数据会被读成坏数据（历史上踩过一次：`LS_KEY` 没定义 → 永久不落盘；`v` 不匹配 → 改完刷新看不见效果）。

例外：`hiddenBones` 是**可选字段**，加了它没有升 `v` —— 升 `v` 会把别人的动画栈整个丢掉，代价比兼容读取大。判断标准：**新增可选字段可以不动 `v`，改动已有字段语义必须升**。

`hiddenBones` 里的骨骼名可能不在当前栈里（比如换了示例组），`renderColbar()` 只显示当前数据里存在的那些；`buildMatrix()` 里过滤时也不在乎（不存在的骨骼本来就没有列）。

---

## 9. 样式系统

设计语言的硬要求（用户给过一份完整规范，摘要）：

- 深色：`--bg #09090B` / `--bg-2 #111113` / `--card #141417` / `--hover #1C1C21` / `--border #27272A` / 文字 `#F4F4F5 · #A1A1AA · #71717A` / accent `#6366F1`
- 浅色：`--bg #FAFAFA` / `--card #FFFFFF` / `--bg-2 #F4F4F5` / `--border #E4E4E7` / 文字 `#18181B · #52525B` / accent `#4F46E5`
- 圆角 8–12，1px 细边，阴影极淡；按钮 34px 高、圆角 6–8；导航 60px
- 字号：正文 13–14 / 辅助 11.5–12 / 微标 10.5；数字用等宽（`--mono`，JetBrains Mono 族）
- 图标统一 `class="icon"`（16px，stroke 1.75），**不许用 emoji 当功能图标**
- 动效 120–180ms（hover）/ 200–280ms（弹窗），并尊重 `prefers-reduced-motion`
- 矩阵列宽：标签列 232px，通道列 **208px**（2026-11-26 从 144 加宽，因为格子改成完整换行后 144 会把 `variable.tcos0 * -1.4` 这种常见表达式折成两行），`width: max(100%, calc(232px + var(--ncol) * 208px))`，窄屏横向滚动
- 格子文字**不截断**：`.expr` 用 `white-space: pre-wrap` + `overflow-wrap: break-word`（`break-word` 而不是 `anywhere`，避免把 `query.body_y_rotation` 从中间劈开，除非单个 token 真的超过一行）；`.keys span` 同理
- `table-layout: fixed`：列宽由第一行（`thead` 的 corner + 骨骼 colspan）决定，所以改列宽要动 `--ncol` 那个公式和 `.chanh` 的 `width` 两处
- 行头 chips（`override` / `this ×N` / `molang ×N` …）是 10px 等宽小字、按语义分色（`.chip.ov` 琥珀 / `.chip.th` 蓝 / `.chip.bad` 红），不再继承 12.5px 正文；`.rh-meta` 允许换行，chips 多了会掉到第二行而不是被裁掉
- 空单元格分两种：`td.none` 一个 `·`（有骨骼没通道）；`td.none.bnone` 斜纹（连骨骼都没有，`repeating-linear-gradient` + `color-mix` 取 `--border-2` 22%，深色很淡、浅色更淡）
- 被 `- this` 抹掉的单轴：`.vec span.cut` 透明 42% + 删除线 + 行尾 `<em class="cutmark">⟂</em>`；三条轴全中时改用整格 `.dead`（省掉三个 ⟂，和 override 观感统一）
- **行头是 sticky 的，背景必须不透明**：`.rowh.ov` 以前用 `--amber-soft`（9% alpha），横向滚动时会透出下面的格子文字、看起来像重影；现在改成 `color-mix(in srgb, var(--amber) 9%, var(--card))` 的实色。以后给 sticky 元素加底色，别直接用带 alpha 的 token
- 行头 `cursor: grab`（整块可拖），但 `.key` 和 `.acts button` 单独 `cursor: pointer`；抓手图标的视觉尺寸不变，靠 `padding:4px` + 负 margin 把热区撑到 24×29
- **滚动模型：全页只有一个滚动容器**。`body` 是 `flex column + overflow: hidden`，`.shell` 是 `flex: 1 1 auto; min-height: 0` 的 flex column，`.mwrap` 吃剩余高度并 `overscroll-behavior: contain` —— 这样滚轮/触控板永远只滚表格，滚到头也不会把手势传给页面（以前 `body` 会跟着滚几十像素，看起来像「两层抢滚轮」）。`.mwrap` 不再写死 `max-height: calc(100vh - 168px)`
- 断点：1023px、767px（767 以下隐藏部分说明、缩列宽）

状态语义色：`--accent`（交互）/ 琥珀（override 边界）/ 蓝 `#93C5FD`（`this`）/ 灰（被抹掉）。

> 用户给的原始规范全文见 **附录 A**（逐字保留，冲突时以它为准）；逐条落地情况见 **附录 B**。

---

## 10. 交互速查（用户视角）

| 操作 | 结果 |
| --- | --- |
| 拖行首上下拖 | 换行序（**整块行头都能起手**，不一定要抓小圆点）；松手即重算 `⟂` 与蓝标 |
| 点骨骼表头（`root` / `head` 那一格） | 收起该骨骼的全部通道列；矩阵上方出现「已隐藏 N 根骨骼」，点骨骼名或「全部显示」恢复。**只影响显示** |
| 点眼睛 | 该行停用/启用（停用行不参与 `this` 与 override 判定） |
| 点垃圾桶 | 从栈里删除 |
| 顶栏「示例库」 | 打开弹窗；**勾选立刻增删**，无确认按钮；组头有「全选」「**替换整栈**」（清掉现有全部，只留这一组） |
| 顶栏「导入动画」/ 拖文件进窗口 | 单个动画直接入栈；一个文件多个动画时弹勾选列表 |
| 顶栏「粘贴」 | 手动贴 JSON（同样容忍 `//` 注释） |
| 顶栏「导出」 | 下载 `bedrock-animation-stack.json`（顺序 + 边界 + 每通道原文） |
| 顶栏「说明」 | 语义说明弹窗 |
| 顶栏撤销 / 重做（或 `Ctrl+Z` / `Ctrl+Y`） | 回退 / 重做上一步：换序、启停、增删、替换整栈、收起或恢复骨骼列都能回退 |
| `Esc` | 关闭当前弹窗 |
| 顶栏月亮/太阳 | 深浅主题切换 |

### 键盘

| 按键 | 结果 |
| --- | --- |
| `Tab` / `Shift+Tab` | 依次聚焦：抓手 → 眼睛 → 垃圾桶 → 下一行（焦点头有 `:focus-visible` 焦点环） |
| 抓手聚焦后 `↑` / `↓` | 当前行上移 / 下移一行（等价于拖动） |
| 抓手聚焦后 `Home` / `End` | 移到栈顶 / 栈底 |
| 骨骼表头聚焦后 `Enter` / `Space` | 收起该骨骼的通道列（等价于点表头） |
| `Enter` / `Space`（焦点在按钮上） | 触发该按钮（浏览器原生） |
| `Ctrl+Z` / `Ctrl+Y`（或 `Ctrl+Shift+Z`） | 撤销 / 重做；焦点在 `input` / `textarea` 时不拦截 |
| `Esc` | 关闭弹窗 |

移动后焦点保持在被移动那一行的抓手上，可直接连按；状态栏 `#stHint` 会显示「第 N 行：动画名」。

---

## 11. 已知问题 / 待办（按优先级）

1. ~~蓝色 `this` 高亮未做浏览器验证~~ → **已实测通过**（浅色主题，`.expr.th` 47 个 / 普通 `.expr` 48 个；computed text `rgb(29,78,216)`，bg `rgba(37,99,235,.1)`，border `rgba(37,99,235,.34)`；`td.hasthis` = 0）。`humanoid.sneaking` 的 body.rotation 只有 x 变蓝，`armor_stand.athena_pose` 整行蓝，`player.sleeping` 上方琥珀带 + `head`/`root` 划掉，均符合预期。
2. ~~键盘导航缺失~~ → **已补**（见 §10 键盘表与附录 B §12）。
3. 浅色主题下矩阵未逐项复查（`.bandrow` 琥珀带、`.dead` 灰化在浅色下的对比度已截图看过，可接受；`.expr.th` 已实测）。
4. ~~`preview.png` / `samples.png` 落后于当前代码~~ → **2026-11-26 已按当前代码重拍**（深色、默认 5 行，1848×857，截图时窗口 1466 CSS px）。注意旧图里格内的 `this` 小徽标已被移除，现在只有**逐轴蓝色表达式** + 行首 `this ×N` chip，看旧图别以为丢了功能。
5. **Molang 求值链是死代码**（见 §5.5）：要么接回来做「求值」视图，要么删除。
6. `applyWeight()` / `fmtV()` 无调用点。
7. 矩阵版 `initDrag()` **没有自动滚动**（v3 卡片版有过 110px 边缘自动滚动）；行数多时拖到屏幕外会吃力，可补。
8. ~~无 git 仓库~~ → **已 `git init`**（`main`，基线 `0eeaf25`；`.gitattributes` 用 `* -text` 锁死行尾，避免 Windows autocrlf 改动字节）。提交身份是仓库内的 `DUANQIUYI <duanqiuyi@localhost>`，想换就 `git config user.email ...`。
9. 移动端只做了「不横向溢出」级别的适配，矩阵本身必须横向滚动（列多时），可考虑冻结骨骼表头行。
10. 示例库分组说明 `.lib-note` 用 `--text-3`（深色 #71717A on #141417 ≈ **3.8:1**），11.5px 正文低于 WCAG AA 4.5:1，截图里几乎看不清。改 `--text-2`（≈5.9:1）即可，但会动视觉层级，**未擅自改**。
11. **撤销栈只在内存里**：刷新页面就清空（`save()` 不写历史，避免把整份 `def` 再复制进 localStorage）。颜色 `it.color` 也不在快照里，撤销后若被 `assignColors()` 重排过不会复原（视觉小瑕疵，不影响语义）。
12. **行高完全由内容决定**：`player.sleeping` 的 root.position 有 140+ 字符，那一行约 338px；`bee` 那种 52 帧的通道会更高。这是「完整显示」的必然代价，没有截断。若以后嫌太高，正确做法是加一个「完整 / 紧凑」视图开关（默认完整），**不要偷偷把 slice 或省略号加回来**。
13. `load()` 里 `!data.items.length` 直接返回 false，于是**把行全删光再刷新，5 条默认示例会自己回来**。空状态（`#empty`）其实是合法状态，是否要持久化空栈待定。

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
5. 导入 `C:\Users\DUANQIUYI\Desktop\claude\bedrock-samples-v1.26.50.4-full\resource_pack\animations\cat.animation.json`（带 `//` 注释）→ 弹出 8 条动画的勾选列表，选完能正常入栈；
6. **点掉 `player.sleeping` 的眼睛** → 琥珀带消失、`sneaking`/`athena_pose` 的 `head` 格子恢复成正常值（`⟂` 清零，状态栏变成「5 个动画（4 启用）· 7 根骨骼 · 10 个通道」）；再点亮 → 全部复原。这条是 2026-11-26 修的 bug 的回归用例。
7. 示例库点「**替换整栈**」（FMBE 基础）→ 栈里只剩这 8 行，顺序 `sleeping → creeper.swelling → dragon.neck_head_movement → warden.move → attack.rotations → parrot.moving → minecart.move → parrot.dance`，toast 报「移除 4 个」；再对「bedrock-samples 原生」点一次 → 回到默认 5 行。
8. **完整显示**：`sleeping` 的 `root.position` 那格要能看到完整的 `24.0 * math.cos(query.body_y_rotation) * …` 一直到最后一个 `)`，**没有省略号**；这一行约 338px 高属正常。
9. **收起骨骼**：点表头 `head` → `head` 的两列消失，上方出现「已隐藏 1 根骨骼 · head× · 全部显示」；点 `head×` 或「全部显示」→ 复原；刷新页面后隐藏状态仍在（`localStorage` 里的 `hiddenBones`）。
10. **撤销 / 重做**：随便点掉一行的眼睛 → `Ctrl+Z` 恢复（顶栏撤销按钮可点，重做按钮亮起）→ `Ctrl+Y` 再停用一次；换序、移除、替换整栈同样能回退。输入框里按 Ctrl+Z 应该只影响输入框文本。
11. **`- this` 覆盖**：`athena_pose` 写了 `-this/-this/-this+2`，所以 `move`、`bob` 的四肢旋转格被划掉并带 `⟂`；`sneaking` 的 body.rotation 整格划掉；而 `athena_pose` 自己的格子是正常的。`sleeping` 那条 `30.0 - this` 只影响 head 的 x，但那一格已被 override 整格抹掉。
12. **覆盖率与斜纹**：`move` 行头是 `骨骼 4/8`，`root/body/head` 的格子是斜纹；`sneaking` 的 `rightitem` 是斜纹，但 `leftleg.rotation` 这种「有骨骼没通道」是 `·`。
13. **拖拽范围**：在行头任意位置（动画名、序号、时间那一行）按住往上下拖 5px 以上就能换序，不必瞄准抓手；轻点动画名仍然只是开/关，不会误移。
14. **横向滚动**：把矩阵横向滚到底，行头（尤其 `player.sleeping` 那条浅琥珀底）必须完全不透明，不能透出下面格子的文字。
15. **滚轮不分层**：页面本身不滚（`documentElement.scrollHeight === clientHeight`），滚轮在表格上只滚表格、滚到表尾也不会带动整页；在说明文字上滚滚轮，页面纹丝不动。

---

## 14. 变更记录（接手后）

### 2026-11-26 16:12 · 修「外面滚一层、表格滚一层」的抢滚轮

- 原因：`body` 会滚（`.shell` 底部 padding 72px + `.mwrap` 的 `max-height: calc(100vh - 168px)`，整页比视口高约 70px），而表格自己也是滚动容器。滚轮在表上滚到尽头会把剩余的滚动量传给 body；在说明文字上滚又会让整页动、表格跟着位移。
- 改法：`body` 改成 `flex column + overflow: hidden`；`.shell` = `flex: 1 1 auto; min-height: 0` 的 flex column；`.mwrap` = `flex: 1 1 auto; min-height: 0; overscroll-behavior: contain`，去掉写死的 `max-height`；底部 padding 72 → 56（状态栏 fixed 40 + 16 缝）。
- 连带修：空栈时 `.mwrap` 会因 `flex: 1` 撑出一块空白边框，`render()` 里现在连 `#mwrap` 一起 hidden，`#empty` 自己 `flex: 1` 铺满。
- 实测：`docScroll [685,685]` / `bodyScroll [685,685]` / `bodyCanScroll false`，`mwrap` 高 460、内容 784（只有它在滚），`overscroll-behavior: contain`，表格底边到状态栏 16px。
- `preview.png` 重拍（表格现在铺满到状态栏上方）。

### 2026-11-26 15:52 · 行头可拖范围放大 + 修 sticky 行头透明

- **横向滚动时动画名重影**：`.rowh.ov` 用的是 `--amber-soft`（9% alpha），sticky 行头本该挡住下面滚过去的格子，结果透了上来。改成 `color-mix(in srgb, var(--amber) 9%, var(--card))`，实测 computed `background-color` 已是无 alpha 的 `color(srgb …)`。
- **可拖范围**：以前只有 16px 的抓手能起手，现在整块 `th.rowh`（眼睛/垃圾桶除外）都能拖；加了 5px 阈值 + 「竖向位移 > 横向」判定，轻点仍是开关、横滑不会误触；拖完 300ms 内的 click 直接忽略（`lastDragAt`），避免 dragging 结束时把行点成停用。抓手热区也从 16px 撑到 24×29（`padding:4px` + 负 margin，布局不动）。
- `cursor`：行头 `grab`，`.key` / `.acts button` 保持 `pointer`。
- 验证：`bsk drag` 从行头中部（不是抓手）把第 3 行拖到第 2 行 → 顺序变了、状态栏仍是「5 个动画」没被误停用；点撤销按钮复原；`grip` 实测 24×29。
- `preview.png` 无需重拍（外观只有 override 行底色从半透明变实色，肉眼几乎一致）。

### 2026-11-26 15:35 · 撤销/重做 + `- this` 也算「被覆盖」+ 骨骼覆盖率

- **撤销 / 重做**：`hist` 快照（只记 item 引用 + 启停 + `hiddenBones`，上限 80 步），`histPush(label)` 在每个改栈动作前打点。`Ctrl+Z` / `Ctrl+Y` / `Ctrl+Shift+Z` + 顶栏两个箭头（禁用态 + title 显示上一步是什么）。输入框内不拦截。弹窗开着的撤销会 `libSyncRows()` 同步勾选。
- **`- this` 的覆盖效果**：新增 `axisCancels()` + `shadow`（`bone|base|ax → 最后一个抵消行`）。系数 -1 且没被乘除的 `- this` 视为「写成绝对值」，它上面的行在这条轴上被划掉；三条轴全中时按整格划掉，视觉与 override 一致。像 `this * 0.5`、`2 * -this` 不算。
- **骨骼覆盖率**：行头加 `骨骼 4/8`（title 列涉及/未涉及）；完全不涉及的格子加淡斜纹，与「有骨骼没通道」的 `·` 区分开；说明弹窗新增「空与划掉」卡片。
- 顺带：行头 chips 改成 10px 等宽分色（之前是继承 12.5px 的裸文本，容易和正文混）；`.rh-meta` 允许换行，chips 不再被裁；`Esc` 关弹窗。
- **验证**：合成用例断言 `- this` 三种情形（单轴 / 三轴 / 覆盖率）—— `partial{cut:1,marks:1,dead:0}`、`full{dead:1}`、`cover:["骨骼 3/3","骨骼 1/3"]`、`bnone:2`；真机 Ctrl+Z/Ctrl+Y、隐藏列撤销、按钮撤销均通过。
- `preview.png` 重拍（深色 1848×857）。

### 2026-11-26 14:52 · 通道值完整显示 + 骨骼列可收起

- **不再折叠**：`.expr` 去掉 `text-overflow: ellipsis` + `nowrap`，改 `pre-wrap` + `break-word`；关键帧从「前 2 帧 + 末帧」改成**全部列出**。列宽 144 → 208px（不然常见的 20 字符表达式会折两行）。
- **骨骼列可收起**：点表头（`root` / `head` 那一格）收起该骨骼的通道列；`#colbar` 显示「已隐藏 N 根骨骼」+ chip + 全部显示；`state.hiddenBones` 落 `localStorage`（可选字段，未升 `v`）。
- **顺手修的文案**：
  - 导出弹窗写着「当前时间点每个骨骼三个通道的最终值（已按顺序叠加…）」—— 实际导出的是顺序 + 边界 + 原文，**不求值**，已改。
  - 说明弹窗「关键帧只列前 3 帧」「this 徽标 + 淡紫底」都是旧版描述，已改；新增「只看想看的骨骼」卡片。
  - 顶部说明补一句「点骨骼表头可以收起它的通道列」。
- **验证**（bsk + 8765）：长表达式完整换行（无省略号）；点 `head` 表头 → 列消失 + 提示栏出现；点 chip / 全部显示 → 复原；隐藏后刷新 → 仍在；把 8 根骨骼全部隐藏 → 表格只剩行首，不崩。
- `preview.png` 重拍（1848×857，深色）。`samples.png` 未变（示例库弹窗这轮没动）。

### 2026-11-26 14:22 · 示例库「整套加入」改成「替换整栈」+ FMBE 组顺序对齐 wiki

- **替换整栈**（`libGrabGroup()` 重写，`index.html` 3 处）：以前是「把缺的追加进来」，现在是 `state.stack = 这一组`，其他动画全删，`assignColors()` 重排配色，toast 报「移除 N 个」。按钮文案从「整套加入」改成「替换整栈」并加 `title` 说明，弹窗顶部说明同步改口。
- **FMBE 顺序**：组里 `parrot.dance` 原来排在 `minecart.move` 前面，与 wiki `render.mcfunction` 相反（那边是 `wiki.ypos` 的 minecart 先、`wiki.zpos` 的 dance 后）。已按 wiki 顺序改正，组说明里注明「顺序 = render.mcfunction 里 /playanimation 的先后」。`adv` 组是同一序列的前 5 条，本来就对，没动。
- **验证**（bsk）：对 FMBE 点「替换整栈」→ 8 行，顺序与 wiki 逐条一致，toast「移除 4 个」，弹窗勾选态同步（core 只剩 sleeping 打勾、fmbe 全勾、adv 全勾）；再对 core 点一次 → 回到默认 5 行。
- `samples.png` 重拍（按钮改名了）；`preview.png` 无需重拍（矩阵没动）。

### 2026-11-26 14:05 · 修「停用 override 行仍然生效」

- **现象**：点掉 `player.sleeping` 的眼睛后，琥珀横带照样画、`sneaking`/`athena_pose` 的 `head` 格子照样划掉，和 §10「停用行不参与 this 与 override 判定」自相矛盾。
- **原因**：`buildMatrix()` 的 `bounds` 取的是 `rows.filter(r => r.ov)`，`render()` 也是 `if (r.ov)` —— 只判了「这是不是一段 override 动画」，没判「它此刻是否启用」。
- **修法**（`index.html` 3 处，+8 −3）：行记录加 `ovOn = ov && item.enabled`；`bounds` 只收 `ovOn`；横带改 `if (r.ovOn)`；`ov` 留着给行首琥珀竖条（那是身份，停用也还是 override 动画）。
- **验证**（bsk + `http://127.0.0.1:8765`，浅/深两套主题都看过）：停用 `sleeping` → 无「作废」横带、`⟂` 为 0、状态栏 `5 个动画（4 启用）· 7 根骨骼 · 10 个通道`；重新点亮 → 横带回来、`⟂` 回到 2 个格子。
- **基线**：git `0eeaf25`（改前）→ 修复提交。

### 同批顺带

- `git init`（`main`），`.gitattributes` 写 `* -text` 关掉行尾转换（`git show :index.html` 与工作区 sha256 一致）。
- 文档纠错：关键帧列表是**前 2 帧 + 末帧**（不是前 3）；`⟂` 在**右下角**（不是右上角）；文件行数/体积更新。
- `preview.png` / `samples.png` 按当前代码重拍。

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