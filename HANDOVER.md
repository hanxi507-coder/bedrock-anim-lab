# Bedrock Animation Stack · 接手指南

> 这份文档写给接手的人（或模型）。目标：读完就能改，不用重新猜需求。
> 最后更新：**2026-11-27 13:20（接手第二十六次改动）** · `index.html` **3796 行 / 153,583 B**，`node --check` 通过。
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
- 仓库已 `git init`（分支 `main`，作者取本机账号）。**改大块之前先 `Copy-Item` 一份备份**（血的教训见 §12），改完立刻 `commit`。

---

## 0.5 接手速查（WB 交接笔记 + dsh 核对）

| 事项 | 结论 |
| --- | --- |
| 看“接手后到底改了什么” | `git diff 0eeaf25 HEAD` —— `0eeaf25` 是 `baseline: v4 矩阵版`。**信 diff，别信文档里的行号**（行号会漂，当前 `index.html` 3796 行 / 153,583 B） |
| 提交数口径 | WB 记的是 52 条；dsh 这边 `git rev-list --count --all` = **32**（单分支 `main`，无其它 ref）。数量对不上多半是中途 `--amend`/重建过分支，内容没丢——以 diff 为准 |
| `.workbuddy/memory/` | WB 的会话笔记目录，只在 WB 侧完整；本机这份几乎是空的，**别依赖它**，要点都已镜像进 §14 |
| B 站 UID | 文档里刻意清掉了（commit `89e3297`）；真值在代码 `openAbout()` 的链接上（`index.html` 约 3722–3730 行）。要改只改那里，别写回文档 |
| 浏览器验证环境 | 只有 **Edge Profile 1** 装了扩展；断连时的恢复步骤见 §14 的 12:32 条目（MV3 worker 约 10s 休眠 / daemon revoked / `~/.bsk/local-*`） |
| 动语义之前 | 先读 §5。`this` / `override_previous_animation` / 累加是这套东西的灵魂，历史上两次「看起来是 bug 其实是语义」都出在这里 |
| 文档习惯 | 每轮改完在 §14 追一条：用户原话 + 做法 + 验证数据 + commit。这份文档的价值全靠这个习惯 |

---

## 1. 这是什么 / 用户要什么

用户的原话（第一轮）：

> 给我做一个可交互网页，用来演示 minecraft 基岩版里资源包动画 json 的作用，要显示所对应骨骼的 position, rotation, scale 的 xyz 通道，支持导入动画 json，支持动画顺序可拖动，有 `override_previous_animation` 和 `-this` 等覆盖性操作的标识。

经过四轮返工，现在的形态是：

**一张矩阵表**——**行 = 一段动画**（行首可拖动换序，上面的行先算），**列 = 骨骼 → 通道**（`position` / `rotation` / `scale`，固定顺序），**格子里显示这段动画 JSON 里这一通道的原文**（`variable.*` / `query.*` / `this` 一律不代入数字，原样摆出来），并把两件“覆盖类”语义做成可见标记：

| 标记 | 含义 |
| --- | --- |
| 蓝色表达式 | 这一条轴（x/y/z 单条）的原文里出现 `this` |
| 琥珀色顶边 | 这一行带 `override_previous_animation: true`：它碰到的骨骼，上面的累计到这里切断（行顶一条 2px 琥珀线 + 行头琥珀底色）。**该行被眼睛停用时线不出现**（见 §5.2） |
| 格子划掉 + `⟂` | 这一格被下面的某一行 override 抹掉了（`⟂` 在格子**右下角**，不是右上角） |
| 单轴划掉 + 右侧小 `⟂` | 这一条轴被下面的 `- this` 写成了绝对值，上面的累积不算了（三条轴都这样时按整格划掉，只画一个 `⟂`） |
| 收起的骨骼列 | 点骨骼表头可把该骨骼的通道列收起来，矩阵上方出现「已隐藏 N 根骨骼」+ 恢复按钮；**只影响显示，不参与判定** |
| 斜纹格 | 这段动画**压根没有**这根骨骼（`·` 则是「有骨骼但没写这个通道」）；行头 `骨骼 4/8` 是覆盖率，悬停看具体漏了哪几根 |

**长值默认折叠，但可以展开**（2026-11-26 16:42 定稿）：单轴超过 44 字符、或关键帧超过 4 个的格子，默认只显示一行（`nowrap` + 省略号）/ 前 3 帧，格子右下角有 `展开` 按钮；点开后完整换行、按钮变 `收起`。**不是静默截断**——每个被折叠的格子都明确给了入口，状态在 `state.openCells`（内存）。短值（`variable.tcos0 * -1.4`、`-this + 10` 这类）永远完整显示，不会被折。

早一点的中间版本曾经「一律完整显示」，行高会飙到 338px（`player.sleeping` 的 root.position 一行 140 多字符），所以又收回来了——**但收法是给按钮，不是改回 `slice` 吞掉**。

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
├─ index.html        153583 B  ← 全部交付物（HTML+CSS+JS 内联）
├─ bili.png             5.8 KB 哔哩哔哩头像（i0.hdslb.com 拉下来 → 64×64 PNG）
├─ avatar.png           803 B  作者头像：从 %APPDATA%/minecraft-desktop-pet/skin.png 裁的 8×8 脸（+帽子层），放大到 160×160 最近邻
├─ preview.png       139615 B  默认 5 行矩阵截图（深色，1848×857，2026-11-26 重拍：表格铺满到状态栏、覆盖率 + 斜纹 + -this 划掉）
├─ samples.png       144938 B  示例库弹窗截图（深色，含「替换整栈」按钮）
├─ vars.png          145973 B  变量取值弹窗截图（深色，variable.tcos0 = 40 / query.life_time = 0.5）
├─ compare-override.png  119169 B  两种作废画法（有底色 vs 划线）的上下对比图
├─ server.js            803 B  极简静态服务（8765，no-store）
├─ _v1-viewer.html   124345 B  v1（3D 版）备份，不要拿去改
├─ HANDOVER.md                 本文
├─ .gitignore / .gitattributes 忽略 _check.js / _backup；**关掉一切行尾转换**（`* -text`）
└─ _backup/index-2026-11-26-1401.html   改 bug 前的手工备份（与基线 commit 同内容）
```

---

## 4. 数据来源（重要）

- 本地完整仓库：`%USERPROFILE%\Desktop\claude\bedrock-samples-v1.26.50.4-full\`
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

> 注意这里的前提：同一通道的多行是**相加**的，`this` 读到的是加到现在的结果，所以 `-this` 才等于清零。底部「最终值」行就是按这个模型算的（详见 §5.5）。
>
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
- 这一行**顶边一条 2px 琥珀线**（`tr.ovon > th/td` 的 `border-top`）+ 行头琥珀底色，表示「上面的累计到这里切断」；被它抹掉的格子是**淡琥珀底（6%）+ 琥珀色划掉 + 琥珀 ⟂**；而被 `- this` 写成绝对值的格子**不铺底**、只有灰色划掉 + 灰 `⟂`。两种「作废」靠有无底色区分 —— 用户明确要求这个区分（否则分不清是 override 还是 this 干的）。
  - 23:20 之前是一条通栏文字横带，嫌占高度；23:30 改成行头里一句「head, root 从这里重算」，用户嫌多余；23:40 彻底改成纯视觉。骨名不再显示在行里，改到行头 `override` 芯片的 `title`（`badgesFor()`）和 `th` 的 `aria-label` 里；
- 被抹掉的格子（`deadAt()`）加 `.dead`，文字划掉（透明度 .42）+ **右下角** `⟂`。

**停用（点眼睛）时不生效**：横带不出现，上方格子也不再划掉。实现上 `buildMatrix()` 给每行算了两个位：
`ov`（这段动画是否带 override，身份，停用也还是）与 `ovOn`（此刻是否真的在生效 = `ov && enabled`）；
`bounds` 只收 `ovOn` 的行，行头那块说明也用 `r.ovOn` 控制。改这里时两个位别混用 —— 2026-11-26 之前只判了 `ov`，
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

矩阵里对关键帧：**默认先列前 3 帧 + `… 共 N 帧`，点格子右下角的「展开」看全部**（2026-11-26 下午定稿：中午一度改成全列，行太高，又加回了折叠——但这次是显式的、能一键展开，不是静默吞掉）。时间标在行首，`pre ≠ post` 的那一帧标琥珀色（跳变点）。逻辑在 `chanHTML()`：

```js
const ks = clipped ? ch.keys.slice(0, 3) : ch.keys;   // clipped = isLongChannel() && !state.openCells.has(key)
return '<div class="keys">' + ks.map(line).join('')
  + (clipped ? '<span class="more">… 共 ' + ch.keys.length + ' 帧</span>' : '') + '</div>';
```

真实数据里 88% 的通道 ≤ 8 帧，但 bedrock-samples 里最长有 **52 帧**（`bee.animation.json` 的 `leftwing_bone / rotation`）。

### 5.5 「最终值」行：符号叠加，不代入数字

表格底部那一行是把**启用中的行**按顺序叠起来的结果。用户定过两条规矩，缺一不可：

1. **不代入时间**（「最终值和时间没有关系」）→ 关键帧那种随时间变的量只记作 `t`；
2. **不代入数字**（「有 query 和 variable，不用写成数字形式，直接带着」）→ 式子里的 `variable.*` / `query.*` 原样留在结果里，只做化简和抵消。

实现（`computeFinal()`，无参数）：

```js
const e = [0,1,2].map(i => {
  let row = astFrom(ch.value[i]);                 // 原文 → AST
  if (exprUsesThis(ch, i)) row = astSub(row, prev.e[i]);   // this := 上一行结果
  if (w !== 1) row = …乘权重…;
  return astSum(prev.e[i], astFold(row));         // ★ 累加，不是覆盖
});
```

**累加语义**（这点很关键，改动过一次）：引擎里同一通道的多行是**相加**的，`this` 读到的是加到现在的结果。所以 `-this` 等于「把这条轴清零」，`0.5 - this` 等于「设成绝对值 0.5」，`-this + 10` 等于「设成 10」——app 里 §5.1 的说明就是这么写的，早先按「覆盖」算会得到 `-62` 这种数，是错的。

`astSum(prev, row)` 会把两边摊平成项列表，**把结构相同、符号相反的两项直接消掉**（`this` 与 `-this` 是同一个对象引用，所以 `=== ` 就能判），常量项合并，最后重建表达式并折叠。效果：

| 情形 | 结果 |
| --- | --- |
| `var.tcos0` + 72 + `10 - this`（this = var.tcos0 + 72） | `10`（未知量自己消掉，不给数字也能得出确定值） |
| `-90` / `0` / `-3.1` 这类纯常量 | 直接是数字 |
| `24 * math.cos(query.body_y_rotation) * …` | 原样带着 `query`，不硬算 |
| 关键帧通道 | `t`（`TV_NODE` 占位；后面若有 `-this` 也会被消掉） |

渲染：`astText()`——`num` 走 `fmtV()`，其余走 `astPrint()`（带优先级括号、`math.cos(x)` 这种函数调用格式）。太长（> `LONG_EXPR`）的格子加 `.clip` + `展开`，和普通单元格同一套。

**已删除的死代码**：`sampleChannel` / `num` / `hermite` / `applyWeight` / `animLocalTime` / `baseCtx` / `MOLANG_VARS` / `evalMolang`（现在只需要 `Molang.compile` 拿 AST、`Molang.evalAst` 折叠常量）。不要再把「采样某一帧」那套加回来。

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
    #colbar   已隐藏骨骼提示栏（默认 hidden：`已隐藏 N 根骨骼` + 骨骼 chip + 全部显示）· flex: none
    .mwrap > #matrix   矩阵表（.mtx）· flex: 1 1 auto; min-height: 0; overscroll-behavior: contain
      └ tbody 最后一行 tr.sumrow = 最终值（sticky bottom；th 里是「最终值 · 不代入时间」，没有时间控件）
  body 末尾动态挂 .bonemgr（骨骼开关面板，点矩阵左上角「骨骼」按钮弹出）
    #empty    空状态（虚线卡片）· flex: 1 1 auto（空栈时 .mwrap 一起 hidden，否则会多出一块空边框）

footer.statusbar（fixed 40px，所以 .shell 底部留 56px）
  #stAnims  N 个动画（M 启用） | #stBones N 根骨骼 | #stChans N 个通道 | #stHint 拖动行首排序 | #btnAbout 作者（avatar.png + 汉西 · Hancy）

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
| `computeFinal()` | 无参数，按顺序把各行的表达式**相加**成每个通道的 AST（override 先清零、`this` 代入、`blend_weight` 乘权重），返回 `Map<'bone|base', {e:[AST,AST,AST], by}>` |
| `astFrom` / `astSub` / `astFold` / `astSum` / `astPrint` / `astText` | 符号层：原文转 AST、把 `this` 换成上一行 AST、常量折叠、相加并抵消同类项、按优先级打印、单元格文本（数字走 `fmtV`） |
| `sumRowHTML(cols, fin)` / `sumCellHTML(col, fin, vaCount)` | 底部「最终值」行：左侧只写「最终值」（2026-11-27 去掉了「不代入数字」副标题）+ 设了变量时一个小角标「已代入 N 个」；`vaCount > 0` 时每条轴先试 `astNumeric()`，能折叠成常数就显示数字、否则退回符号式；折叠判定看**显示文字**长度（代完只剩 `24` 就不给「展开」） |
| `exprUsesThis(ch, ax)` / `identVec(base)` | 判断某条轴是否读 `this`（用于脏值传播）/ 给通道的单位元（0 或 1） |
| `axisCancels(ch, ax)` / `CANCEL_THIS` | 这一轴的值是不是「被 `- this` 写成绝对值」（系数 -1、没被乘除）→ 供 `shadow` 用 |
| `fmtNum(n, base)` | 数字格式化（rotation / 时间 2 位，其余 3 位，去尾零） |
| `litHTML(v, base)` | 单条轴渲染：字符串 → `<code class="expr">`，含 `this` 再加 `.th` 蓝色；数字 → `.num` |
| `chanHTML(ch, base, cuts, clipped)` | 一个通道 → x/y/z 三行；`cuts[ax]` 是抵消行号时该轴加 `.cut` + 右侧小 `⟂`；`clipped` 时关键帧只渲染前 3 帧 + `… 共 N 帧`，整通道被抵消时加 `.dead-keys` |
| `badgesFor(item)` / `rowHead(r, allBones)` | 行首：抓手、序号、动画名、时长、**覆盖率 `骨骼 4/8`**（title 列涉及/未涉及）、loop 图标、chips（override / this / molang / keyframe / fx）、眼睛（启停）、垃圾桶（删除） |
| `cellHTML(row, col, bounds, shadow)` | 单格：没有该通道时输出 `·`（`.none`）或斜纹（`.bnone`，连骨骼都没有）；有值时按 override（整格 `.dead`）与 `shadow`（逐轴 `.cut`）上标记；长值加 `.clip` + 「展开」按钮 |
| `isLongChannel(ch)` / `LONG_EXPR` | 单轴字符串 > 44 字符，或关键帧 > 4 个 → 算「长」，默认折叠 |
| `toggleCell(key)` | 展开/收起单格，key = `item.id|骨骼|通道`，存在 `state.openCells`（内存，不落盘）；重建后把焦点还给同一个按钮 |
| `collectVarNames()` / `usedVarCount()` / `hasVars()` | 栈里出现过的 `variable.* / query.* / temp.* / context.*`（**含停用行**，停用了也允许先设值）、真正被用到且已设值的数量（决定最终值要不要代数字）、是否设过任何值 |
| `astApplyVars(n)` / `astNumeric(n)` | 把 `state.vars` 里已知的标识符节点换成 `num` 节点；整棵 AST 都是常数就返回数字，否则 `null`（保留符号式） |
| `varsBodyHTML()` / `varRowHTML(name)` / `openVars()` / `initVars()` | 「变量」弹窗：按 variable / query·其他 / 已设置但没用到 三段列输入框，`input` 事件即时写 `state.vars` + `render()` + `save()`；清空输入即删除该值 |
| `snapRows()` / `playRows(snap)` / `renderFlip()` | 换序动画：重建前记下每行 `top`，重建后把行从旧位移 `translateY` 拉回 0（220ms）；`renderFlip()` = 量 → render → 播。`prefers-reduced-motion` 时跳过，标签页在后台（rAF 被挂起）有 90ms 兜底 |
| `render()` | 重建整张表 + 写 `--ncol`（列数）+ 切换 `#empty` / `#mwrap`（空栈时两个都 hidden）+ 调用 `renderColbar()` |
| `renderColbar(allBones)` | 画「已隐藏 N 根骨骼」栏：只列当前数据里真的存在、且被隐藏的骨骼；每个是一个 `data-show` 按钮，右侧「全部显示」 |
| `setBonesHidden(next, label)` | 骨骼显隐的唯一入口：一次改完 `state.hiddenBones` 再 `histPush` + `render` + `save`，批量操作不刷一堆历史 |
| `soloBone(bone)` | Alt+点表头：只留这一根；已经是「只看它」就还原成全部显示 |
| `toggleBoneMgr()` / `refreshBoneMgr()` / `closeBoneMgr()` / `placeBoneMgr()` | 骨骼面板：开/关、重画芯片、跟着按钮定位（`scroll` 捕获阶段 + `resize` 重新定位） |
| `hideBone(bone)` / `showBone(bone)` | 隐藏 / 恢复单根骨骼的列，`render(); save()`，隐藏时 toast 提示去哪恢复 |
| `updateStatus()` | 写状态栏（**只统计启用行的骨骼/通道，不受隐藏影响**） |

### 7.4 交互

| 函数 | 职责 |
| --- | --- |
| `initDrag()` | 跟手拖拽换序（手机 app 那种）：**整行任意位置**都能起手（`button` / `[data-exp]` / 链接输入框除外），过 5px 阈值才进拖拽。被拖的行加 `.lift`，`translateY` 贴着指针走；指针越过别的行中线时**立刻改 `state.stack` 并 `render()`**，其它行用 FLIP 滑到新位置（190ms），被拖的行自己重新贴回指针。松手才 `save()` + toast；拖完 300ms 内的 click 被 `lastDragAt` 吞掉 |
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
| `openAbout()` | 「关于这个项目」弹窗：头像 + 汉西 / Hancy / B 站 @小汉西 + vibe coding 说明 + 版权行 |
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
- override 行是 `tr.ovon`：`> th/td` 一条 2px 琥珀 `border-top`，行头底色 `color-mix(amber 9%, card)`；`.cell.dead`（override 作废）= 6% 琥珀底 + 琥珀划掉 + 琥珀 `⟂`；`.cell.cutall`（三条轴被 `- this` 写死）= 无底 + 灰色划掉 + 灰 `⟂`
- 骨骼面板 `.bonemgr` 是 `position: fixed`（挂在 body 上，`render()` 重建表格不会把它冲掉），跟着角落按钮定位；`.bm-chip.on` 用实心点 + `--card` 底表示「显示中」
- 最终值里的表达式用 `.sumv`（10.5px 等宽、`--text-2`，纯数字 `.isnum` 提亮）；太长走 `.clip` 的省略号 + 「展开」
- 底部「最终值」行是 sticky（`bottom: 0`），底色 `--bg-2` 必须不透明；行头那格同时 sticky left（`z-index: 6`），不然横向滚动会错位
- 换序动画只动 `transform`（FLIP），不动布局；`<tr>` 上的 transform 在 `border-collapse: separate` 下是可靠的
- 拖着的那行是 `.lift`：`z-index: 9`、单元格底色补成 `--card`（td 本来透明，不补就会透出下面的行）、上下各一条 1px 边线当"被拿起"的效果；松手/重建时 `place()` 重新把它贴回指针
- 面板宽度：`.shell max-width: 1560px`（顶部说明已整块删除，表格从导航下方 20px 处开始）；矩阵列宽：标签列 232px，通道列 **208px**（2026-11-26 从 144 加宽，因为格子改成完整换行后 144 会把 `variable.tcos0 * -1.4` 这种常见表达式折成两行），`width: max(100%, calc(232px + var(--ncol) * 208px))`，窄屏横向滚动
- 格子文字**不截断**：`.expr` 用 `white-space: pre-wrap` + `overflow-wrap: break-word`（`break-word` 而不是 `anywhere`，避免把 `query.body_y_rotation` 从中间劈开，除非单个 token 真的超过一行）；`.keys span` 同理
- `table-layout: fixed`：列宽由第一行（`thead` 的 corner + 骨骼 colspan）决定，所以改列宽要动 `--ncol` 那个公式和 `.chanh` 的 `width` 两处
- 行头 chips（`override` / `this ×N` / `molang ×N` …）是 10px 等宽小字、按语义分色（`.chip.ov` 琥珀 / `.chip.th` 蓝 / `.chip.bad` 红），不再继承 12.5px 正文；`.rh-meta` 允许换行，chips 多了会掉到第二行而不是被裁掉
- 空单元格分两种：`td.none` 一个 `·`（有骨骼没通道）；`td.none.bnone` 斜纹（连骨骼都没有，`repeating-linear-gradient` + `color-mix` 取 `--border-2` 22%，深色很淡、浅色更淡）
- 长值折叠：`td.cell.clip` 下 `.expr` 用 `nowrap + ellipsis`；`.xbtn` 是 10px 等宽小按钮（`--text-2`，hover 提亮），右对齐贴在格子底部
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
| 按住整行上下拖 | 行跟手走、其它行实时让位（手机 app 的手感）；松手才落库，`⟂` 与蓝标全程跟着重算。格子里的值也能起手，按钮和「展开」除外 |
| 点骨骼表头（`root` / `head` 那一格） | 收起该骨骼的全部通道列；矩阵上方出现「已隐藏 N 根骨骼」，点骨骼名或「全部显示」恢复。**只影响显示** |
| `Alt` + 点骨骼表头 | 只看这一根（其余全收起），再 Alt + 点还原 |
| 点左上角「骨骼」按钮 | 弹出面板列出所有骨骼，点芯片开/关（面板不关，可以连着点），底部「全部显示 / 全部隐藏」；点面板外面或 Esc 收起 |
| 拖动 / ↑↓ 换序 | 落手后各行有 220ms 的滑动到位动画（`renderFlip()`），移动的那行会从旧位置滑到新位置 |
| 点格子右下角的「展开」 | 把这一格的长表达式 / 全部关键帧摊开，按钮变成「收起」；再点收回。**只影响这一格**，不写 localStorage |
| 点眼睛 | 该行停用/启用（停用行不参与 `this` 与 override 判定） |
| 点垃圾桶 | 从栈里删除 |
| 顶栏「示例库」 | 打开弹窗；**勾选立刻增删**，无确认按钮；组头有「全选」「**替换整栈**」（清掉现有全部，只留这一组） |
| 顶栏「导入动画」/ 拖文件进窗口 | 单个动画直接入栈；一个文件多个动画时弹勾选列表 |
| 顶栏「粘贴」 | 手动贴 JSON（同样容忍 `//` 注释） |
| 顶栏「导出」 | 下载 `bedrock-animation-stack.json`（顺序 + 边界 + 每通道原文） |
| 顶栏「说明」 | 语义说明弹窗 |
| 顶栏撤销 / 重做（或 `Ctrl+Z` / `Ctrl+Y`） | 回退 / 重做上一步：换序、启停、增删、替换整栈、收起或恢复骨骼列都能回退 |
| 顶栏「`{ }` 变量」 | 打开变量取值弹窗：给 `variable.*` / `query.*` 填数字，最终值行立刻把能算的算成数字；清空输入即退回符号式；「全部清空」一次清完 |
| 变量输入框（按住左右拖） | 拖动改值，**拖多少改多少**（1px = 1，`Shift` ×10，`Alt` ×0.1），手停就停；位移小于 3px 不算拖动，可以正常点进去敲数字 |
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
3. 浅色主题下矩阵未逐项复查（`.rh-ov` 琥珀说明块、`.dead` 灰化在浅色下的对比度已截图看过，可接受；`.expr.th` 已实测）。
4. ~~`preview.png` / `samples.png` 落后于当前代码~~ → **2026-11-26 已按当前代码重拍**（深色、默认 5 行，1848×857，截图时窗口 1466 CSS px）。注意旧图里格内的 `this` 小徽标已被移除，现在只有**逐轴蓝色表达式** + 行首 `this ×N` chip，看旧图别以为丢了功能。
5. **Molang 求值链是死代码**（见 §5.5）：要么接回来做「求值」视图，要么删除。
6. `applyWeight()` / `fmtV()` 无调用点。
7. 矩阵版 `initDrag()` **没有自动滚动**（v3 卡片版有过 110px 边缘自动滚动）；行数多时拖到屏幕外会吃力，可补。
8. ~~无 git 仓库~~ → **已 `git init`**（`main`，基线 `0eeaf25`；`.gitattributes` 用 `* -text` 锁死行尾，避免 Windows autocrlf 改动字节）。提交身份取本机账号（全局无 `user.name` 时会被拒绝提交，先 `git config user.name/email` 设一个，别写真实姓名）。
9. 移动端只做了「不横向溢出」级别的适配，矩阵本身必须横向滚动（列多时），可考虑冻结骨骼表头行。
10. 示例库分组说明 `.lib-note` 用 `--text-3`（深色 #71717A on #141417 ≈ **3.8:1**），11.5px 正文低于 WCAG AA 4.5:1，截图里几乎看不清。改 `--text-2`（≈5.9:1）即可，但会动视觉层级，**未擅自改**。
11. **展开状态只在内存里**：`state.openCells` 不写 localStorage，刷新后全部回到折叠态（故意的，免得下次打开发现一堆被撑高的行）。
12. **撤销栈只在内存里**：刷新页面就清空（`save()` 不写历史，避免把整份 `def` 再复制进 localStorage）。颜色 `it.color` 也不在快照里，撤销后若被 `assignColors()` 重排过不会复原（视觉小瑕疵，不影响语义）。
12. **行高**：默认折叠时一行 60–90px；点开长值后那一行会明显变高（`player.sleeping` 的 root.position 全展开约 250px），这是预期，不是 bug。想再压高度就调 `LONG_EXPR` 阈值或列宽，**别把「展开」按钮去掉、改成静默截断**（用户明确要的是「能展开」）。
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
3. `player.sleeping` 的顶边有琥珀线、行头偏琥珀；它上面 `sneaking` 的 head 格子是**淡琥珀底 + 琥珀划掉 + 琥珀 ⟂**；同一行 `body.rotation`（`0.5 - this`）**没有底色**、只有灰色划掉 + 灰 `⟂`；`sneaking`、`athena_pose` 在 `head` 上的格子被划掉并带 `⟂`（它们写了 `head`，而 `sleeping` 会清 `head`、`root`）；
4. 示例库勾选一条 → 关闭弹窗前后都能在矩阵里看到它（且是启用状态）；
5. 导入 `%USERPROFILE%\Desktop\claude\bedrock-samples-v1.26.50.4-full\resource_pack\animations\cat.animation.json`（带 `//` 注释）→ 弹出 8 条动画的勾选列表，选完能正常入栈；
6. **点掉 `player.sleeping` 的眼睛** → 琥珀带消失、`sneaking`/`athena_pose` 的 `head` 格子恢复成正常值（`⟂` 清零，状态栏变成「5 个动画（4 启用）· 7 根骨骼 · 10 个通道」）；再点亮 → 全部复原。这条是 2026-11-26 修的 bug 的回归用例。
7. 示例库点「**替换整栈**」（FMBE 基础）→ 栈里只剩这 8 行，顺序 `sleeping → creeper.swelling → dragon.neck_head_movement → warden.move → attack.rotations → parrot.moving → minecart.move → parrot.dance`，toast 报「移除 4 个」；再对「bedrock-samples 原生」点一次 → 回到默认 5 行。
8. **完整显示**：`sleeping` 的 `root.position` 那格要能看到完整的 `24.0 * math.cos(query.body_y_rotation) * …` 一直到最后一个 `)`，**没有省略号**；这一行约 338px 高属正常。
9. **收起骨骼**：点表头 `head` → `head` 的两列消失，上方出现「已隐藏 1 根骨骼 · head× · 全部显示」；点 `head×` 或「全部显示」→ 复原；刷新页面后隐藏状态仍在（`localStorage` 里的 `hiddenBones`）。
10. **撤销 / 重做**：随便点掉一行的眼睛 → `Ctrl+Z` 恢复（顶栏撤销按钮可点，重做按钮亮起）→ `Ctrl+Y` 再停用一次；换序、移除、替换整栈同样能回退。输入框里按 Ctrl+Z 应该只影响输入框文本。
11. **`- this` 覆盖**：`athena_pose` 写了 `-this/-this/-this+2`，所以 `move`、`bob` 的四肢旋转格被划掉并带 `⟂`；`sneaking` 的 body.rotation 整格划掉；而 `athena_pose` 自己的格子是正常的。`sleeping` 那条 `30.0 - this` 只影响 head 的 x，但那一格已被 override 整格抹掉。
12. **覆盖率与斜纹**：`move` 行头是 `骨骼 4/8`，`root/body/head` 的格子是斜纹；`sneaking` 的 `rightitem` 是斜纹，但 `leftleg.rotation` 这种「有骨骼没通道」是 `·`。
13. **拖拽范围**：在行头任意位置（动画名、序号、时间那一行）按住往上下拖 5px 以上就能换序，不必瞄准抓手；轻点动画名仍然只是开/关，不会误移。
14. **横向滚动**：把矩阵横向滚到底，行头（尤其 `player.sleeping` 那条浅琥珀底）必须完全不透明，不能透出下面格子的文字。
15. **长值折叠**：`humanoid.bob` 的左右臂 z 轴、`player.sleeping` 的 root.position/rotation 默认显示成一行 + 省略号，格子右下角有「展开」；点开后完整换行、按钮变「收起」；其余短值不受影响。
16. **最终值**：底部一行左边只有「最终值」三个字，**没有时间控件、也没有副标题**；默认 5 行时 `body.rotation` 是 `0 / 0 / 2`（sneaking 的 `0.5 - this` 被 athena 的 `-this` 抵掉），`head.rotation` 是 `30/0/0`（override 清零后 sleeping 的 `30-this`），`leftarm.rotation` 是 `10/0/-5`（`variable.tcos0` 被 `-this + 10` 消掉），`root.position` 显示成带 `query.body_y_rotation` 的长表达式并带「展开」。
17. **换序手感**：按住行里任意位置（行头或格子）往下拖，那一行会跟手走并盖住经过的行，其它行实时让位（190ms 滑动）；拖到第 4 行再松手，顺序真的落到第 4 位，不是只挪一格。松手后位置保持、刷新后还在。
18. **快速藏骨骼**：左上角「骨骼」按钮弹出 8 个芯片（当前 8/8），点 `head` → 芯片变暗、表头少一列、顶部出现「已隐藏 1 根骨骼」、面板不关且计数变 7/8；「全部显示」回到 8/8。`Alt` + 点 `body` 表头 → 只剩 body 一列，再 Alt + 点还原。
19. **滚轮不分层**：页面本身不滚（`documentElement.scrollHeight === clientHeight`），滚轮在表格上只滚表格、滚到表尾也不会带动整页；在说明文字上滚滚轮，页面纹丝不动。
20. **变量取值**：点顶栏「`{ }` 变量」→ 弹窗按 `variable` / `query·其他` 分组列出当前栈出现的标识符，没设值的显示 `未设置`；给 `variable.tcos0` 填 `40` → 最终值行左边出现「已代入 1 个」，而 `leftarm.rotation` 仍是 `10`（被 `-this + 10` 抵消，说明代入真的参与了运算）；把 sneaking / athena / sleeping 三行停用后，同一格变成 `40`；再把 `query.body_y_rotation`、`query.sleep_rotation` 都填 `0` → `root.position` 从两条长式变成 `24 / 0 / 0`、`root.rotation` 变 `-90 / 270 / 0`、格子右下角的「展开」消失；清空输入 → 一切退回符号式，角标消失。
21. **拖动调值**：按住任一变量输入框左右拖 —— 向右拖 10px 值就变 `10`、继续拖到 40px 变 `40`，**手停住数字立刻停住**（松手前后都不会自己往上爬）；按住 `Shift` 拖是 ×10、按住 `Alt` 拖是 ×0.1；拖完松手，刷新页面值还在（`localStorage`）；不想拖就直接点进去敲数字，两条路都通。

---

## 14. 变更记录（接手后）

### 2026-11-27 23:15 · 原则：双端不追求像素一致，各自最优

用户明确：「手机和电脑端不用完全一致，符合双端的各自需求就行」。所以以后不要为了"看起来一样"把桌面端的东西往手机上搬（或反过来）。当前**有意为之**的差异：

| 方面 | 桌面（fine pointer，≥1024） | 手机（coarse，≤480） |
|---|---|---|
| 拖动抓手 | 不画点阵，靠 `cursor: grab` + 整块行首可拖 | 画 9×17 点阵（`--text-3`，opacity .85）+ 首次进入的一次性提示条 |
| 行首列 | 210px，标题两行 + 元信息（骨骼 4/8、molang 数） | 88px（收起态），标题两行、元信息收起、按钮竖排 |
| 图标按钮 | 28px / 16px 图标 | 26px / 15px 图标 |
| 顶部导航 | 单行，图标按钮常驻 | 换行 + 可横滑，`.nav-actions` 折两行 |
| 骨骼列 | 全部展开，横向滚动 | `只看` 选择器：点一根只看它（一屏放得下） |
| 表达式 | 单行省略 + 触屏/鼠标点「展开」 | 同左（共用一套规则） |

判断标准不是"两端长得像"，而是：**桌面追求一屏信息密度，手机追求单手能点中、能看清当前一行**。### 2026-11-27 23:00 · 行首那列「几行名字叠在一起」：补分隔线 + 顶端对齐

- 现象：手机上「上面那几行还是重叠了」。查证：`th.rowh` 里几行的名字挤在一条连续白带上，原因有两个：
  1. `th.rowh` 是 `vertical-align: middle`，而每一行的高度由**最长的那个格子**决定（3 行表达式 ≈ 85px），于是名字被垂直居中、浮在行中间，和右侧数据的第一行错位；
  2. 行首这一列**没有行分隔线**（`.mtx tbody th` 只有 `border-bottom`，被 sticky 背景盖住），几行的名字连成一片，看起来就是"叠在一起"。
- 改：`th.rowh` 改 `vertical-align: top`、`padding: 6px 10px 7px`、补 `border-top: 1px solid var(--border)`（`:first-child` 除外）、`.rh-top` 加 `overflow: hidden` 防任何溢出。
- 实测（374 与 1440 两个 iframe 同时量）：行首 `.key` 的顶边距行顶 6–12px（贴着第一行数据），行与行之间无重叠（`ov: []`），名字能读到两行时也不会串到下一行。### 2026-11-27 22:50 · 拖动仍会闪：根因是「拿带 transform 的 rect 当布局位置」

- 用户「还会闪，特别是往下拖的时候和下面的卡片互换那个动画」。物理原因不在渲染时机，而在**测量**：
  1. `place(fresh)` 在 `fresh` 时把 `tr.getBoundingClientRect().top` 当作"布局位置"，但此时元素身上还挂着上一帧的 `transform: translateY(ty)` → 算出来的新 `ty` 把旧的位移又算了一遍。往下换位后元素自身下移一整行，于是**瞬间跳到下一个格位再被 FLIP 拉回**，看着就是"抽一下"。
  2. `topsNow()` 同样读的是带 transform 的 rect，`targetIndex()` 与 `flipOthers()` 的基准因此偏移一整行。
- 改法（自纠正式测量）：
  - 新增 `measure()`：先把所有行的 `transform` 清空，再量 rect，同时把 `drag.ty` 归零；
  - `place()` 不再分 `fresh`：清空自身 transform → 量布局位置 → `ty = (curY - offY) - natural` → 写回；
  - `applyTarget()` 改成 `measure() → splice → insertBefore → measure() → flipOthers(prev) → place()`，`flipOthers` 直接用两次 measure 的差值，不再自己读 rect；
  - `startDrag()` 先把 `.lift` 与 `body.dragging` 挂上**再**量 `offY`（否则一起手整行会先跳十几个像素）。
- 验证（合成 pointer 事件，向下拖动跨过两行）：被拖行 `top - pointerY` 恒为 `-38`（**全程零跳变**，跨越换位也没变化）；被顶走的行拿到 `translateY(89px)` 再缓动回 0；松手 500ms 后 `lift=0`、`dragging=false`、`idx` 回到 `01…05`、`state.stack` 与 `localStorage` 顺序一致。### 2026-11-27 22:45 · 桌面端三个真 bug：render 每帧抛异常 / 单元格折成麻花 / 桌面没有展开

- 用户说「电脑端你同步了吗，感觉问题还是挺多」。文件确实同步了（Pages 与 `main` 同字节），但**代码本身有 bug**：
  1. **`render()` 末尾还在调 `syncHeadToggle()`** —— 那个函数在 22:05 改成 `toggleCol/toggleSum` 时被我删了。于是**每一次 render 都在最后一行抛 `ReferenceError`**：表格已经重建完，所以肉眼看不出，但 `render()` 之后的语句全被跳过 —— 拖完松手那次 `save()` 没执行（刷新就回退）、toast 不弹、示例库加入后不落盘。这是"问题挺多"的主因。
     - 修：删掉该调用（`grep syncHeadToggle` = 0）。用 `?` 打点验证：`render / save / loadSamples / toggleSum×2 / toggleCol×2 / addToStack / removeItem / 点展开` 全部 `errs=[]`，`localStorage` 正常写入。
  2. **单元格在 136px 列里折成 4 行**：`clip` 只在「字符串长度 > 44」时生效，而 `variable.head_position_x - this`（31 字符）不达标 → `white-space: pre-wrap` 把它按 `variable.head_` / `position_x -` / `this` 硬切，行高参差不齐（一行 220px）。
     - 修：`.expr/.num` 全局 `white-space:nowrap; overflow:hidden; text-overflow:ellipsis`，`.vec>span/.keys li` 加 `min-width:0`；`LONG_EXPR` 44 → **18**（一行约放得下 18–20 个等宽字符），超过就给「展开」按钮。
  3. **「展开」点了没反应**：上一条把它变成永远单行后，`open` 状态没有对应样式。
     - 修：`cellHTML` 在 `open` 时给 `<td>` 加 `.open`，CSS `.mtx td.cell.open .expr { white-space:pre-wrap; overflow:visible; overflow-wrap:break-word }`。实测点开后 `white-space=pre-wrap`、格高 105px、按钮变「收起」。
- 结论：以后改完必须**在浏览器里跑一遍关键路径**（render/save/拖拽/增删），不能只看 `node --check` —— 语法通过不代表运行时没引用死亡函数。
### 2026-11-27 22:35 · 拖动换位不再整表重建（修「两张卡抽一下」）+ 拖拽态彻底去色

- 用户反馈两条：「电脑端拖动时不是中性色」「拖动交换时两个卡片会抽出一下」。
- **抽一下的真因**：`applyTarget()` 每次跨过中线都调 `render()`，把 `tbody` 整个 innerHTML 换成新节点；紧接着 `flipOthers()` 又给这些**新节点**设 FLIP 位移。被拖的那行与其它行的变换不在同一帧落地，中间就闪一下（看着像两张卡被拉伸/复制）。指针快速划过多行时每跨一次都会来一遍。
- 改法：
  1. `applyTarget()` 不再 `render()`，只 `state.stack` 换位 + `tbody.insertBefore(tr, anchor)` 挪一个节点（anchor = 去掉被拖行后的第 `to` 个 `tr.arow`，没有就插到 `tr.sumrow` 前）。其它行照旧 FLIP，且因为节点没被替换，过渡是连续的。
  2. 松手：先把被拖行 `transition: transform 140ms` 落回 `translateY(0)`，150ms 后 `cleanup(); render();` —— 序号（01…）、`最终值`、`dead/cutall` 标记都在这一刻归位。
  3. 去色：`body.dragging .mtx th/td:focus-visible { outline: none }`（拖拽时点下行首会命中 `th.rowh:focus-visible` 的 2px 强调色描边，那就是"不是中性色"的来源）；`.lift` 行首竖线用 `--text-2`、行底色 `--hover`、其余行 `opacity:.45`。
- 验证（桌面 1482×685，合成 PointerEvent 序列）：拖动中 `body.dragging=true`、`.lift=1`、DOM 顺序随指针实时变；松手后 `idx` 变回 `01…05`、DOM 顺序 === `state.stack`、`.lift=0`、`body.dragging=false`。
- 注意：拖动过程中单元格内容不会重算（矩阵显示的是 JSON 原文，与顺序无关），但 `最终值` 与 `dead/cutall` 标记是顺序相关的 —— 它们统一在松手后的那次 `render()` 刷新。
### 2026-11-27 22:28 · 「正在拖」不占色相：改用抬升 + 压暗其它行

- 用户反馈「我正在拖的这一行的强调色也是紫色」—— 对，`--accent`(#6366F1) 和 `--violet`(#A78BFA) 在深色底上都是紫，拖拽色又会和 `this` 撞车。**结论：色相已经被语义占满了（琥珀=override、紫=this、天蓝=molang），"正在操作"这类瞬时状态不该再要一个色相。**
- 改法（纯 CSS，`index.html` 拖动段）：
  1. `tr.lift` 的背景用 `--hover`（更亮的表面），上下各 1px `--border-2` 描边，首尾格加 `0 14px 26px -10px rgb(0 0 0 / .55)` 投影，行首左边改成 3px **中性灰** `--text-2`。
  2. `body.dragging .mtx tbody tr.arow:not(.lift) > * { opacity: .45 }` —— 拖的时候其它行自动压暗，焦点靠对比度而不是颜色。
  3. 悬浮（仅 fine pointer）也从中性灰走：`inset 2px 0 0 var(--text-3)`。
- 现在规则一句话：**彩色的出现即语义**（琥珀 override / 紫 this / 天蓝 molang），黑白灰负责一切"交互态"（悬浮、拖拽、焦点）。
### 2026-11-27 22:25 · 拖动/悬浮也改用强调色，身份色彻底不再上屏

- 用户看完配色解释后说「你先改我看看效果…以后不用问我，直接改」。
- 改法：
  1. `tr.lift`（正被拖的那一行）：左侧 3px 从身份色换成 `var(--accent)`；行底色改成 `color-mix(in srgb, var(--accent) 9%, var(--card))`（深色底上比投影更醒目）；首尾格加了 `0 10px 22px -8px rgb(0 0 0 / .45)` 的柔和投影做"抬起"。
  2. 悬浮（`@media (hover:hover) and (pointer:fine)`）：只留 `inset 2px 0 0 var(--accent)`。
  3. 清掉 `--c` 在 CSS 里的最后两处引用（`th.col` 顶边、旧的 hover 双线），现在 `var(--c` 在样式表里 0 命中。
- 结果：屏幕上彩色只表达三件事 —— 琥珀 = override 行、紫 = `this` 单元格、强调色 = 你正在指/正在拖的那一行。`it.color` 仍存着（导出 JSON 里有），但不再画画。
- 真机确认（374×677）：01–04 行左侧无任何颜色，05 仍是琥珀 override 行。
### 2026-11-27 22:15 · 行首那根颜色竖条：说清它是什么，并把它从「像状态」改成「像身份」

- 用户问「动画左边的这个颜色有什么特殊含义吗」。答案：**没有语义**，它只是每段动画的身份色（`STACK_COLORS` 轮转、`assignColors()` 保证同一时刻不撞色、跟着 `animKey` 存进 localStorage，换行序不换色）。
- 但提问本身暴露了真问题：原来的 8 个身份色里有 3 个和语义色几乎同色 —— `#ffb04d`≈`--amber`(override)、`#c084fc`≈`--violet`(this)、`#38bdf8` 就是 `--sky`(molang)。于是「一条普通的紫色行」会被读成「这行有 this」。
- 改法：
  1. 调色板只留 4 个不与语义色冲突的：`#5fd48b` 绿 / `#f472b6` 粉 / `#a3e635` 柠 / `#2dd4bf` 青，第 5 个起统一给钢蓝 `#8b9dc3`。
  2. **身份色默认不显示**：`th.rowh` 的 `inset 3px` 竖条改成 `box-shadow:none`，只在 `tr.lift`（正被拖）与 `@media (hover:hover) and (pointer:fine)` 的 `tr.arow:hover` 时出现 —— 一次只露一行。
  3. 现在整张表里**唯一常驻的彩色边**是 `.rowh.ov`（琥珀，override 行，`index.html:1400`）与 `tr.ovon` 的琥珀顶边（`index.html:1481`），语义唯一。
- 真机确认（374×677）：01–04 行左侧无色；05 `player.sleeping` 仍是琥珀左边 + 琥珀顶边（它是 override 行）。
- 提交 `3c40930`（同一提交里 `HANDOVER.md` 曾被误写成 `index.html` 的内容，已用 `git checkout da4c7d8 -- HANDOVER.md` 复原后重写本节）。
### 2026-11-27 22:05 · 收起交互贴到 sticky 元素上 + 触屏不再残留 hover

- 起因（两条用户反馈）：①「表头不用收起，把收起这个交互直接搞在这两个 sticky 元素上面，不用单起一行」②「手机上触碰到某一行之后背景还是会消失」。
- 去掉上一轮的 `.togglerow`（三颗胶囊开关）与 `headCollapsed`（state / save / load / render 全部清空，`grep headCollapsed` = 0）：
  1. **左列收窄**：按钮挪进 `th.corner`（`<button class="coltoggle" data-coltoggle>`，`chevronsLeft/Right`，20×20，`margin-left:auto`），点它把 168px 的行首列压到 96px（序号 + 省略名 + 眼睛/垃圾桶竖排），再点还原。
  2. **最终值收起**：`th.sumh` 的标签本身变成按钮（`<button class="sumtoggle" data-sumtoggle>`，`min-height:34px` 保证手指点得到 + `chevronsDown`）；收起时 `render()` 不再输出这一行，改成表格后一条 `position:sticky;bottom:0;height:22px` 的 `.sumbar`「最终值 · 已收起，点这里展开」。
  3. 事件改为 `initViewToggles()` 在 `document` 上委托 `[data-coltoggle]` / `[data-sumtoggle]`，不再逐个绑按钮。
- **触屏 hover 残留**：真正的元凶是上一轮我自己加的 `.mtx tbody tr.arow:hover td { background: transparent }` —— 触屏 tap 之后 `:hover` 会一直挂着，于是那一行的底色"消失"。改为整段 `@media (hover: hover) and (pointer: fine)`，且只给 `th.rowh` 加 `inset` 竖线，**完全不碰 td 背景**。
- 另一个坑（已修）：`.mtx.col-collapsed thead th.corner { display:flex }` 会让带 `rowspan="2"` 的 `th` 不再是 table-cell，**下面那行的通道标签整体错位一格**。改成只调 `padding` / `text-align`，保持 `display: table-cell`。
- 真机验证（Pura 70 Pro / 374×677 dpr 3.375，`hdc` + `uitest uiInput click`）：收窄后多出两根通道列、表头与格子对齐；最终值收起后底部只剩 22px 细条且一屏多出约 60px；点任意一行格子区域，行背景**不再变化**。
- 顺带：`ICON` 表补 `chevronsLeft/chevronsRight/chevronsUp/chevronsDown`；`.sumtoggle/.sumfold/.coltoggle/.sumbar` 样式；`node --check` 通过。
### 2026-11-27 14:05 · 窄屏适配（表格列宽 + 导航两行）

- 起因：用户问「移动端和电脑端都适配好了是吧」。实测 390px：`.mwrap` 内部宽 **2728px**（要横滑 8 屏）、导航 118px 挤成 3 行、撤销图标贴边被裁。桌面 1852px 也只能看到 6 个通道列。
- **根因不是「缺 media query」，而是转置后列宽没跟着改**：`.mtx thead th.chanh{width:208px}` + `.mtx{width:calc(232px + ncol*208px)}` 还是「列 = 动画」时代的尺寸，而现在一列只是一个 `position/rotation/scale`，208px 太宽。同时 `@media (max-width:1023px)` / `(max-width:767px)` 两段里写的全是旧卡片类名（`.legend`、`.row`、`.card-head`、`.tags`、`.bound`、`.acc`、`.v`、`.bname`、`.base`），HTML 里 0 命中，等于死代码。
- 改法（只动 CSS）：
  1. 桌面列宽 `232+208n` → **`210 + 140n`**（`.mtx` 的 width、`th.chanh`、`th.corner`、`th.rowh` 四处一起改）。1852px 下可见通道列 6 → 9，12 列总宽 2728 → 1890。
  2. 1023 档：`186 + 128n`。767 档：`168 + 118n`。
  3. 767 档重写：`.nav` 收成两行（品牌一行 / 动作一行），`.nav-actions{flex-wrap:nowrap;overflow-x:auto}` + 隐藏滚动条、按钮 30px，导航高 118 → 84px；`.mwrap` 去左右边框与圆角吃满宽度；`thead`/`th`/`.cell` 字号 11–11.5px；`.rh-meta .cover`（「骨骼 4/8」）与 `.len` 在窄屏隐藏，行高明显变矮；`.modal`/`.statusbar`/`.toasts`/变量弹窗的 `.var-row` 都收窄。
  4. 删掉 767 档里所有旧卡片类名的规则（`.toolbar`、`.legend`、`.card-head`、`.row`、`.bname`、`.base`、`.v`、`.acc`、`.card-empty`）——它们对应的元素早就不存在了。
- 教训：列宽用 `140` 不是随便取的 —— `136` 会让 `variable.tcos0` 在单元格里折成两行（`.expr` 是 `white-space:pre-wrap`，故意不截断），`140` 刚好放得下。
- 验证：390 / 820 / 1852 三档截图；`.mwrap` 宽 2728 → 1890（桌面）/ 1584（手机）；导航高 118 → 84；页面本身仍无横向溢出（`documentElement.scrollWidth == clientWidth`）。

### 2026-11-27 13:08 · 变量输入框改成「拖多少改多少」

- 用户：「这个变量要根据鼠标的位置去改值，鼠标划多少就增加或减少多少，而不是鼠标不动还一直在增减」。
- 实现：`#modalBody` 上 `pointerdown` 记 `{x0, v0}`，`window` 上挂 `pointermove` / `pointerup` / `pointercancel`；位移不足 3px 不算拖动（交回给输入框正常点选、键入）；超过阈值后 `v = (v0 ?? 0) + dx * step`，`step` 默认 1、`Shift` ×10、`Alt` ×0.1。
- **绝对值、不是累加**：每次都由「按下时的值 + 总位移」重算，所以手停就停、来回拖也不会漂移或自增。
- 重绘走 `applyVar()`：按 ~16ms 节流 + `setTimeout` 兜底。**没用 rAF** —— 后台标签页 rAF 会被挂起，拖动时表格会僵住（第一版就是 rAF，测试脚本直接超时，页面卡在 `vscrub` 状态）。`save()` 只在松手时落盘。
- 视觉：`.var-in{cursor:ew-resize;touch-action:none}`；拖动中 `body.vscrub`（`user-select:none` + 输入框强制高亮边框）；说明里写清「按住输入框左右拖动：拖多少改多少（1px = 1，Shift ×10，Alt ×0.1），手停就停」。
- 验证（合成 pointer 序列）：2px → 不变；10px → `10`；同一坐标再发一次 → 仍是 `10`；`Shift` +40px → `400`；`Alt` +35px → `3.5`；松手后 `body.vscrub` 移除、`localStorage.vars` 存到 `3.5`、最终值行角标「已代入 1 个」。

### 2026-11-27 12:58 · 变量取值（WB 起草，dsh 补完）

- 用户需求原话：「再加个可以由用户改变 variable 和 query 的值的功能，就像 blockbench 的变量占位符一样」。WB 的 dsv4.1f 把主体写完了（未提交、未验证、未写文档），本轮接手做验证 + 四处收尾。
- 半成品的实现：顶栏 `{ }` 变量按钮 → 弹窗按 `variable` / `query·其他` 分组列出栈里出现的标识符，一格一个 `input[type=number]`；`input` 事件即时写 `state.vars` + `render()` + `save()`；最终值行在 `hasVars()` 时对每条轴跑 `astNumeric()`，能折叠成常数就显示数字，否则退回符号式。
- 本轮改的四处：
  1. 新增 `usedVarCount()`，「已代入 N 个」只数**栈里真的用到、并且设了值**的（原来是 `Object.keys(state.vars).length`，删过行 / 换过栈之后会虚高）；
  2. `collectVarNames()` 去掉「跳过停用行」——否则一停用，它的变量就从弹窗里消失，值再也改不回来；
  3. 弹窗新增第三段「已设置但当前栈没用到 · 清空可移除」（换栈后残留的值能在这儿清掉；清空后这一段自动消失，且**只在段数真的变了时才重建弹窗**，不抢输入焦点）；
  4. `sumCellHTML()` 的折叠判定从 `sym` 改成 `texts` —— 代完只剩 `24` 时不该再挂一个「展开」。
- 语义边界（刻意如此）：只影响底部**最终值**那一行，格子里的原文永远不动；`t`（随时间变化）和没设值的 `query.*` 原样留在式子里。
- 验证（Edge + 8765）：给 `variable.tcos0` 填 40 → 最终值角标「已代入 1 个」，`leftarm.rotation` 仍是 `10`（被 athena 的 `-this + 10` 抵掉）；把 sneaking/athena/sleeping 三行停用后同一格变 `40`（= `variable.tcos0`）；`query.body_y_rotation` 与 `query.sleep_rotation` 都填 0 → `root.position` 从两条长式变成 `24 / 0 / 0`、`root.rotation` 变 `-90 / 270 / 0`、`展开` 按钮消失、角标「已代入 4 个」；「全部清空」后角标消失、式子复原；刷新后 `state.vars` 从 `localStorage` 正常回来。
- 新增 `vars.png`（变量弹窗实拍）。
- 追加：行内格子悬停也能看到这条原文的值 —— `litValue(v)` 单独把那条字面量代入变量并折叠（`null` 表示还算不出来），`chanHTML()` 的静态分支把它拼进 `<span>` 的 `title`：「这条原文代入变量后 = 40」。没设变量时完全不输出（不产生多余 tooltip）；和「绝对值」的提示共存时用 ` · ` 连起来。
- 注：这轮改动**没有提交**，都在工作区里（约定：等用户确认再 commit）。

### 2026-11-27 12:32 · 浏览器桥接恢复，重拍 preview

- 断连原因：Edge 没开 + 守护进程重启后设备记录是 revoked + MV3 service worker 约 10 秒就休眠（日志里每次「browser connected」后 10s 就 10054）。
- 处理：用 Profile 1 启动 Edge；`bsk daemon trust-local on`；清掉 `~/.bsk/local-pairings.json` 里那条 revoked 设备（备份 `local-pairings.backup.json`）；在扩展弹窗里点一次「配对」，弹窗别关（保活 SW）。
- 已按当前代码重拍 `preview.png`（1848×857，默认 5 行，字号为"只放大元信息"那一版）。
- 附：`bsk open --width/--height` 不会改已有窗口尺寸，`bsk window resize` 需要 cooperate sync 的 observation；截图尺寸取决于 Edge 窗口本身。


### 2026-11-27 12:26 · 「关于」加哔哩哔哩空间链接与头像

- 用户要求把 B 站空间和头像也放进来。
- 账号按用户的公开昵称检索得到；**UID 与主页地址只写在页面里**（页面是公开作品页，属用户本人授权的署名），不要再抄进文档、注释或提交信息里。
- 头像：`https://api.bilibili.com/x/web-interface/card?mid=<UID>` 拿 `face` 字段（300×300，Minecraft 风格的 "Hancy" 字标），下载后 `scale=64:64` 存 `bili.png`（5.8 KB）。
- 布局：名字下方那行改成链接 `[18px b站头像] 哔哩哔哩 @小汉西 ↗`，`target="_blank" rel="noopener"`，指向个人空间；悬停变亮 + 浅底。
- 说明句从「界面头像取自其本人的 Minecraft 皮肤」改成「头像分别取自其 Minecraft 皮肤与哔哩哔哩个人主页」—— 现在有两张图，得说清。
- 12:32 桥接恢复后已重拍 `preview.png`（1848×857）。


### 2026-11-27 12:18 · 「关于」文案改正式

- 用户觉得上一版太口语（「上面这张脸就是他自己的 Minecraft 皮肤，从桌宠里导出的那一张」这类），要求正式一点。
- 现在的文案：
  - 标题「关于」（原来是「关于这个项目」）
  - 名字块：**汉西 · Hancy** / 哔哩哔哩 @小汉西
  - 正文：本项目由汉西发起并主持，以 **vibe coding** 的方式与 AI 协作完成。产品定位、交互细节与全部取舍由其本人决定，实现由 AI 协助产出。
  - 头像说明独立成句：界面头像取自其本人的 Minecraft 皮肤。
  - 来源：动画示例逐字引自 `bedrock-samples v1.26.50.4`，未作改写。
  - 页脚：© 2026 汉西 Hancy
- 头像 `alt` 改成空串（名字就在旁边，重复朗读没意义）。
- 没重拍 `preview.png`（浏览器桥接仍未连接）。


### 2026-11-27 12:15 · 字号：撤回整体放大，只放大"确实太小"的那一档

- 用户澄清：「字体大小改成之前的样式，我其实想说有些地方的字特别小需要调整一下，这种本来就能看清的不用调了」。
- 撤回上一轮的整体 +1px：`.num` 回 12、`.expr` 回 11.5、`.boneh` 回 11.5、`.rh-top .key` 回 12.5、`.sumv` 回 10.5、`.vec i` 回 9.5 再单独调。
- 只把 **≤10.5px 的元信息** 提到 11 左右（这些是真正看不清的）：
  - 行号 `.idx` 10.5 → 11
  - 时长 `.len` 10.5 → 11
  - 徽标 `.chip` 10 → 11
  - 覆盖率 `.cover`（`骨骼 4/8`）10 → 11
  - 通道表头 `.chanh`（`position` / `rotation`）10 → 11
  - 左上角 `.corner` 10 → 11
  - 「展开」按钮 `.xbtn` 10 → 10.5
  - `… 共 N 帧` `.more` 10.5 → 11
  - 逐轴 `⟂` 9.5 → 10；格子角 `⟂` 10 → 10.5；轴字母 9.5 → 10.5
- 表格主体（表达式 / 数字 / 动画名）回到原尺寸，行高也回到原样。
- 这一轮没重拍 `preview.png`（浏览器桥接断开），12:32 连上后已按当前代码重拍。


### 2026-11-27 12:05 · 署名（关于弹窗 + 头像）与字号上调

- 用户要求：在页面上声明这是他的 vibe coding 项目 —— 中文名 **汉西**，英文 **Hancy**（姓 Jared，一般不写出），B 站 **@小汉西**；头像用他 Minecraft 皮肤的脸。
- 头像来源：`%APPDATA%/minecraft-desktop-pet/skin.png`（64×64）。用 ffmpeg 裁 `crop=8:8:8:8`（正脸）+ `crop=8:8:40:8`（帽子层）叠加，`scale=160:160:flags=neighbor` 存成 `avatar.png`（803 B）。显示时一律 `image-rendering: pixelated`。
- 状态栏最右侧加了作者按钮（16px 头像 + 「汉西 · Hancy」），点开 `openAbout()`：58px 头像、汉西 / Hancy · B 站 @小汉西、vibe coding 说明（「由汉西发起并主导，借助 AI 一起完成」）、示例来源、`© 2026 汉西 Hancy`。
- 字号整体上调一档（用户：「字体有点小」）：`.num` 12→13、`.expr` 11.5→12.5、`.vec i` 9.5→10、`.boneh` 11.5→12.5、`.chanh` 10→11、`.rh-top .key` 12.5→13.5、`.sumv` 10.5→11.5。代价是行更高，5 行时第 5 行又有一点点压在最终值行后面（可滚动）。
- `preview.png` 重拍。


### 2026-11-27 00:20 · 顶部说明整块删掉，最终值去掉副标题

- 用户：「不传入数字这个还是不要了吧，让界面简洁一些，说明性的不用太多，使用这个网站的应该都懂这些」，顶部那行提要也不要了。
- 删掉 `.sumrow` 行头里的「不代入数字」副标题（`.sum-sub` 的 markup + CSS 一起删），现在只剩「最终值」三个字。
- 删掉整块 `#hint`（一行提要 + 「完整说明」按钮）以及配套的 `.intro` / `.hint-line` / `.hint-more` 样式和 `#hintMore` 的绑定。需要看说明还有顶栏 `?`（16 张卡片）。
- 净效果：表格从导航下方 20px（`.shell` 的 `padding-top`）直接开始，5 行 + 最终值行全部同屏可见（之前第 5 行被压在最终值行后面）。
- `preview.png` 重拍。


### 2026-11-27 00:15 · 顶部说明压成一行，把高度还给表格

- 用户反馈：顶部那两段说明占了太多竖向空间，表格被挤扁。
- 把两段文字（约 5 行、~90px）压成**一行提要** + 右侧「完整说明」按钮：
  `行 = 动画 · 列 = 骨骼 / 通道 · 上面的行先算（this 读上面的累计）· 按住整行上下拖改顺序 · 底部一行是叠加后的最终值`
- 删掉的内容不是丢了，本来就在顶栏 `?` 打开的说明弹窗里（「这张表怎么读」「顺序就是语义」「this」「override」等卡片）。`#hintMore` 和 `#btnDocs` 绑的是同一个 `openDocs()`。
- 净效果：这一块从 ~90px 降到 ~34px，表格多出约 56px ≈ 多半行高度（现在 5 行能看到 4 行半，之前是 3 行半）。
- 布局：`#hint` 改成 `display:flex; align-items:center`，左边 `.hint-line` 可换行、右边按钮 `flex:none`；窄屏时按钮仍在右侧，文字自己折行。
- 坑：替换 `initDocs` 里的闭包时，`});` → `};` 的替换误命中了 `initModal` 的同名结尾，导致 `});` 丢失、语法报错；已按上下文精确修复。**以后改这种"箭头函数注册 → 具名函数"的写法，先看清楚有几处一样的收尾。**
- `preview.png` 重拍。


### 2026-11-26 23:55 · 两种「作废」分开画：override 带底，`- this` 不带

- 用户要求把琥珀底加回来，理由很实在：**没有底色就分不清格子是被 override 抹的还是被 `- this` 写死的**。
- 于是把原来共用的 `.dead` 拆成两个类：
  - `.dead`（被下面某行的 `override_previous_animation` 作废）= **6% 琥珀底 + 琥珀划掉 + 琥珀 `⟂`**；
  - `.cutall`（三条轴都被下面的 `- this` 写成绝对值）= **不铺底 + 灰色划掉 + 灰 `⟂`**。
- 单轴 `- this` 的小 `⟂` 也从琥珀改成 `--text-3`，跟"作者自己写的绝对值"归一类。
- 顺手修了个老问题：`<code class="expr">` 是 `display: block`，父级的 `text-decoration` 传不下去，所以**表达式格子一直没有划掉线**（只有纯数字有）。现在 `.dead` / `.cutall` 直接给 `.expr` 也加 `text-decoration: line-through`。
- 验证：默认栈里 `td.cell.dead` 2 个（sneaking 的 head.position、athena 的 head.rotation，都是被 sleeping 的 override 抹的）、`td.cell.cutall` 9 个（含 `0.5 - this`）；计算样式 `text-decoration-line: line-through`，颜色 `rgb(113,113,122)` / 琥珀 60%。
- `preview.png` 重拍。


### 2026-11-26 23:40 · override 改成纯视觉（去掉文字）

- 用户反馈：「head, root 从这里重算」这句话多余，希望**直观**体现 override。
- 现在的视觉语言：**琥珀 = 切断**。
  - override 行（`tr.ovon`）顶边一条 **2px 琥珀线**（`border-top`），行头底色也偏琥珀（9%）—— 一眼看到「累计到这里断」；
  - 它抹掉的格子（`.cell.dead`）加 **6% 琥珀底**，配原有的划掉 + `⟂`，整块连成一片「作废区」，和那条线颜色呼应；
  - 骨名不写在行里了，挪到行头 `override` 芯片的 `title`（`badgesFor()`）和 `th` 的 `aria-label`，需要时悬停/读屏能拿到完整语义。
- 删掉 `.rh-ov` 的 markup 和 CSS；`rowHead()` 不再为 override 拼字符串。
- `badgesFor()` 的 override 芯片提示改成带骨名的完整句：「这一行会把它碰到的骨骼（head, root）上面的累积清掉，从自己开始重新叠」。
- 验证：`sleeping` 行顶边有琥珀线、行头偏琥珀；`sneaking` 的 head 格子是淡琥珀底 + 划掉；最终值不变。
- `preview.png` 重拍。


### 2026-11-26 23:20 · override 说明并进行头（去掉通栏横带）

- 原来每个 override 行动画上方插一条 `tr.bandrow`（`colspan` 整行）：`上面的行在 head, root 上的累积到这一行作废 —— 下面几行从这个新起点重新叠`。太占高度，而且把表格横着切一刀。
- 改成行头里一块 `.rh-ov`：`⟲ head, root 从这里重算`，琥珀虚线跟上方的 chips 隔开；原句完整保留在 `title` 里。
- `render()` 里删掉了 band row 的生成；`.mtl tr.bandrow` / `.ovline` 那几段 CSS 一并去掉（`.ovline` 只有横带在用）。
- 行头高度略增（约 20px），但整张表少了一行，净高度更省；`⟂`、`被覆盖格子划掉` 这些不变。
- 验证：`player.sleeping` 的行头文本里出现「head, root 从这里重算」；通栏横带消失；`computeFinal` 与 override 清零逻辑没动（最终值仍是 `30/0/0`）。
- `preview.png` 重拍。


### 2026-11-26 23:15 · 删掉抓手图标，整行都能拖

- 行首那个「⋮⋮」抓手（`.grip`）删了：它本来只是拖拽的提示，现在整行都能拖，图标反而占地方。顺带删掉 `.rh-top .grip` 的三条样式和 `ICON.grip`。
- 拖拽起手从 `th.rowh` 放宽到 **`tr.arow` 任意位置**，只排除 `button` / `[data-exp]` / `a` / `input`（眼睛、垃圾桶、展开按钮照旧不拖）。格子里的表达式也能起手。
- 键盘可达性搬到 `th.rowh` 自己身上：`tabindex="0"` + `aria-label="第 N 行：xxx。拖动或按上下方向键调整顺序"`，`moveRow()` 之后焦点还在这一行上；`↑ ↓ Home End` 的逻辑改成从 `closest('th.rowh[data-id]')` 取行。
- 光标：`tr.arow > td` 和 `th.rowh` 都是 `grab`，格子里只有按钮和「展开」是 `pointer`。
- 验证（探针，合成 KeyboardEvent）：`focusOk: true` → `ArrowDown` 后 bob 从第 1 掉到第 2，焦点仍在 `th.rowh`；`hasGrip: false`；`bsk drag` 从某个**单元格**拖到别的行，顺序正确（move 从第 1 挪到第 4）。
- `preview.png` 重拍。


### 2026-11-26 22:25 · 快速隐藏骨骼（面板 + Alt 独显）

- 矩阵左上角加了个「骨骼」小按钮（`.bm-btn`，有隐藏时变强调色），点开是 `.bonemgr` 浮层：所有骨骼排成芯片，点一下开/关，面板**不关**，可以连着点几根；底部「全部显示 / 全部隐藏」；点外面或 Esc 收起。
- `Alt` + 点骨骼表头 = **只看这一根**，再 Alt + 点还原（`soloBone()`）。
- 显隐统一走 `setBonesHidden(next, label)`：一次算完再 `histPush` + `render` + `save`，批量点芯片不会塞一堆撤销记录；原来的 `hideBone` / `showBone` 也改成调它。
- 面板挂在 `body` 上（`position: fixed`），所以 `render()` 重建表格不会把它冲掉；`scroll`（捕获）和 `resize` 时跟着按钮重新定位。
- 验证：8 芯片 → 点 head 变 7/8 + 表头 7 列 + 顶部恢复栏出现 "head"；全部显示回 8/8；Alt+点 body 只剩 1 列，再点回 8 列；全部隐藏不会崩（顶部恢复栏 8 个 chip 全在）。
- `preview.png` 重拍（角落多了按钮）。


### 2026-11-26 21:58 · 拖拽改成「跟手 + 实时让位」

- 用户反馈：1）拖的时候也要有动画；2）只能拖到相邻位置，跨不了行，想要手机 app 那种手感。
- 现在的做法：按下并越过 5px 后，被拖的行加 `.lift` 并用 `translateY` **贴着指针走**；指针越过别的行的中线 → 立刻 `state.stack` 换位 + `render()`，其余行用 FLIP 从旧位置滑到新位置（190ms），被拖的行再 `place(true)` 贴回指针。松手才 `save()`。
- 插入位算法换成「去掉被拖行之后，有多少行的中线在指针上方」→ 想在哪儿落就在哪儿落，不再受相邻限制。
- 删掉了 `.hline`（蓝色插入线）和 `.dragtag`（跟手小标签）——行本身跟手了，这两样是干扰。
- 加了 `host.addEventListener('dragstart', e => e.preventDefault())`，防止浏览器把拖动接管成原生拖拽（这也是之前"拖不远"的一个可能原因）。
- 坑：替换 `initDrag` 时把紧跟在后面的 `assignColors()` / `save()` 一起截断了，页面直接 `ReferenceError`，已从 git 恢复。**下次用「函数名→下一个函数名」的区间替换，别用模糊结束标记。**


### 2026-11-26 20:20 · 最终值改成「符号叠加，不代入数字」

- 用户要求：式子里的 `query` / `variable` 不要硬算成数字，直接带着。于是把数值采样换成 **AST 叠加**：`astFrom` 解析原文 → `astSub` 把 `this` 换成上一行的 AST → `astSum` 相加 → `astFold` 折叠常量 → `astPrint` 打印。
- **顺手修了一个语义错误**：以前按「覆盖」算（`总 = 本行值`），但引擎是**相加**的（`-this` = 清零）。改成 `总 = 上一行 + 本行值` 之后，`-this + 10` 才真的等于 10。默认栈里 `leftarm.rotation` 从错误的 `-62` 变成正确的 `10`（`variable.tcos0` 与 `-this` 直接抵消），`body.rotation` 从 `-0.5` 变成 `0`。
- `astSum` 会做项级抵消：结构相同、符号相反的直接消掉（`this` 与 `-this` 是同一对象引用，`===` 即可判），常量合并，重建后按优先级打印（`math.cos(x)`、`(a + b) * c`）。
- 删掉彻底没用的 `sampleChannel` / `num` / `hermite` / `applyWeight` / `animLocalTime` / `baseCtx` / `MOLANG_VARS` / `evalMolang`（−4.4KB）；`Molang` 只多导出 `evalAst`（折叠常量用）和 `FN`。
- 最终值格子太长时和普通格子一样 `.clip` + 「展开」（键用 `sum|骨骼|通道`，和 `item.id|…` 不冲突）。
- 验证：`head.rotation` = 30/0/0、`leftarm.rotation` = 10/0/-5、`root.rotation` = `-90 / 270 - query.sleep_rotation - query.body_y_rotation / 0`、`root.position` = 两条带 `query` 的长式子（`.sumv` 实测 `clientWidth 174 < scrollWidth 867`，省略号生效）。

### 2026-11-26 17:35 · 最终值去掉时间 + 换序加动画

- **最终值不再有时间**（用户：「最终值和时间没有关系」）：删掉时间滑块、`state.t`、`paintSum()`；`computeFinal()` 不再收 `t`，`strictQuery = {}`（所有 query 都不喂），关键帧通道直接标 `t`。渲染三态：数值 / `t`（随时间变化）/ `—`（缺变量），行头写成「最终值 · 不代入时间」。
- **换序动画**：`snapRows()` + `playRows()` 做 FLIP——重建前记每行 `top`，重建后 `translateY(旧-新)` 再过渡回 0（220ms，`cubic-bezier(.2,.8,.2,1)`）。`moveRow` / 拖拽落手 / 撤销重做都走 `renderFlip()`；`prefers-reduced-motion` 跳过；后台标签页 rAF 被挂起，用 90ms 定时器兜底；`snapRows()` 开头先清残留 transform，避免从错位置起跳。
- 验证：探针里 `moveRow(0,2)` 后三行的 transform 分别是 `translateY(86px)/86px/-159px`（方向、幅度正确），落定后归零；`#tslider` 已不存在，行头文本「不代入时间」。

### 2026-11-26 17:15 · 底部「最终值」行（当天 17:35 已去掉时间滑块）

- 把停摆很久的求值链（`sampleChannel` / `animLocalTime` / `applyWeight`）接上：表格底部多一行 **sticky 的「最终值」**，按当前顺序叠加，`this`、`override`、`blend_weight` 都算进去；左边一个时间滑块（0 ~ 最长动画时长），拖一下整行数字实时变。
- `sampleChannel()` 现在多返回 `bad:[bool,bool,bool]`（该轴是否引用了没值的符号），`computeFinal()` 再按「override 清零 → 直写覆盖 → 读 this 才继承脏值」的规则往下传。
- **不编数**：`ctx.vars = {}`，`strictQuery` 只放 `anim_time / life_time / delta_time / frame_alpha / modified_distance_moved`；`QUERY` 里那些演示假值在严格模式下走 `miss()`。算不出的轴显示 `—`，悬停看缺了哪个符号、最后是哪一行写的。
- 验证（探针，iframe 内直接调 `computeFinal`）：bob 的 z 在 t=0 → **5.73**、t=0.5 → **4.645**；`variable.tcos0` 单行 → `bad[0]=true`；`variable.tcos0` 后接 `-this + 10` → **10** 且 `bad=false`（未知量被抵消）。
- `preview.png` 重拍。

### 2026-11-26 16:42 · 面板加宽 + 长值可展开

- **面板加宽**：`.shell` 的 `max-width` 1120 → **1560**（`.shell > #hint` 单独收在 1040px，说明文字不会拉成一条长线）。同屏能多看到一列多。
- **长值默认折叠、可展开**：新增 `isLongChannel()`（单轴 > 44 字符或关键帧 > 4 个算长）与 `state.openCells`。长的格子在 `.clip` 下：静态通道每条轴 `nowrap + ellipsis` 一行，关键帧只渲染前 3 帧 + `… 共 N 帧`；格子右下角一个 `展开 ⌄` 按钮，点开变 `收起 ⌃` 并完整换行/列出全部。
- 数据上：默认 5 行里有 4 个格子会被折叠（bob 左右臂 z、sleeping 的 root.position / root.rotation），行高从 338px 量级回落到 86px。
- 展开状态只放内存（`state.openCells`，key = `item.id|骨骼|通道`），刷新即折叠；撤销/重排后仍跟着同一段动画。
- 说明弹窗、顶部提示同步（「长表达式默认收成一行，点格子里的展开看全文」）。
- `preview.png` 重拍（1848×857）。

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
---

## 15. 项目收尾（2026-11-28）

**状态：功能完成，线上稳定。** HEAD `baed985`，构建号 `11-28 03:22`，`index.html` 4260 行 / 179 797 字节。

### 15.1 这一夜改了什么（按时间）

| 提交 | 内容 |
| --- | --- |
| `8f34876` | 恢复「动画名列折叠」，但**只在 ≤767px 生效**；桌面端彻底没有折叠（此前桌面折叠时会因 `table-layout:fixed` 把差额摊给通道列，看着像"整表被缩放"） |
| `14e2d27` | 变量弹窗每行改成 `名字 │ 输入框` 两列网格（原来 `.var-row` 是 flex，`grid-template-columns` 不生效，输入框吃满整行把名字压成 0 宽） |
| `2f483ea` | 删掉状态栏里的构建号（只在「关于」里保留）；变量行名字不再被挤掉 |
| `05766d3` | 排序提示按设备区分：鼠标「整张卡片任意位置都能拖」／触屏「按住行首那一列拖」 |
| `2f73b3c` | 提示条去掉左侧点阵图标（文字已说明抓手，图标多余） |
| `049ee67` | 变量拖动灵敏度 1 → **0.4/px**（Shift 4、Alt 0.04），双端一致 |
| `6ce3909` | 再慢一倍 → **0.2/px**（Shift 2、Alt 0.02）；提示文案改成**速率等式**：`值 = 0.2 × 位移(px)` |
| `3a8829b` | 最终值行「已代入 N 个」不再顶出名字列（换行 + 截断 + 手机去掉量词）；**新增：表格里点变量名直接改值** |
| `7e7db03` | 展开后的长表达式重新可换行（`.vec span` 改 `.vec > span`，变量名不再被 flex 化）；变量名高亮底色代替下划线 |
| `baed985` | 变量名平时不着色，只在 hover / 按下 / 键盘聚焦时出现 3px 圆角高亮 |

### 15.2 关键实现点（以后还会踩）

- **折叠只在手机**：`@media (max-width: 767px)` 里才有 `.mtx.col-collapsed { --lbl: 88px }`；桌面端 `toggleCol()` 即使被调用也不会改变 `--lbl`（实测 1478px 宽加类后仍 210px）。宽度公式 `max(100%, calc(var(--lbl0) + var(--ncol) * var(--chw)))` + 尾巴列 `<col class="c-pad">` 吸收余量，保证收放时**通道列恒宽**。
- **可点变量名**：`litHTML()` 把 `\b(variable|query|context|temp)\.[A-Za-z0-9_]+` 包成 `<span class="vref" role="button" tabindex="0">`；`#matrix` 上委托 click/keydown(Enter·空格) → `openVars(name)`（打开弹窗 + 滚到该行 + 聚焦全选 + 该行 `.hit` 高亮）。
- **`.vref` 必须是 `display:inline` 且 `white-space:inherit`**：`.vec span { display:flex }` 曾把变量名也 flex 化，导致「展开」后不可换行、压到右邻格。修法是把那条规则收窄成 `.vec > span`。
- **变量拖动步长**：`const step = e.shiftKey ? 2 : (e.altKey ? 0.02 : 0.2);`（`index.html` 约 4150 行），纯 `clientX` 位移驱动、手停就停，**不用 rAF**（后台标签页会被挂起）。

### 15.3 部署

- 仓库 <https://github.com/hanxi507-coder/bedrock-anim-lab>，Pages = GitHub Actions。
- 工作流 `.github/workflows/static.yml` 必须**拆成两个 job**：`build`（checkout + configure-pages + `upload-pages-artifact@v3` with `path:'.'`）与 `deploy`（`needs: build` + `environment: github-pages` + `deploy-pages@v5`）。写在同一 job 里会出现 `No artifacts named "github-pages"` / 重跑后 `Multiple artifacts named "github-pages"`。
- 本地预览：`node server.js` → <http://127.0.0.1:8765/index.html>（`Cache-Control: no-store`，改完刷新即可）。
- 手机真机：HDC `hdc rport tcp:8765 tcp:8765` + `aa start -b com.huawei.hmos.browser -a MainAbility -U "URL"`；`uitest uiInput click` 用**逻辑像素**（设备像素 ÷ 3.375）。首次进入态可用 `http://localhost:8765`（与 `127.0.0.1` 的 localStorage 互不相通）。

### 15.4 仍可打磨（非阻塞）

1. `.btn.tiny` 在移动端 < 44px 触控标准；
2. 矩阵横向滚动时骨骼表头行可再冻结（现在只有首列 sticky）；
3. 拖拽无边缘自动滚动；
4. 首次进入默认浅色 —— 若要跟随"深色为主"的设计初衷，可在 `initTheme()` 里把缺省改成 `dark`。
