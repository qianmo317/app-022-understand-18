# 一个字从输入到印在纸上：全链路代码走读

> 范围：`src/lib/input.ts`、`src/lib/layout.ts`、`src/lib/data.ts`、`src/lib/pinyin.ts`、
> `src/components/paint.tsx`、`src/components/PageView.tsx`、`src/lib/exportImage.tsx`、
> `src/pages/Home.tsx`、`src/pages/Editor.tsx`、`src/pages/PrintView.tsx`、`src/types.ts`。

## 0. 全景：数据在五个环节之间的形态

```
原文文本 string
  │  ① parseInput()                src/lib/input.ts
  ▼
去重后的字符数组 string[]          （Worksheet.chars，持久化到 localStorage）
  │  ② buildBlock() 逐字拆格        src/lib/layout.ts
  ▼
Block[]（每字 = 若干 Cell：例字/笔顺分解/描红/空格）
  │  ③ paginate() 贪心装行 + 切页   src/lib/layout.ts
  ▼
Page[] = Row[] = Block[]           （纯数据，每次渲染现算，不持久化）
  │  ④ RowContent / PageView 画 SVG src/components/paint.tsx, PageView.tsx
  ▼
屏幕预览 / 打印页（同一个 PageView，plain 开关只影响选中态）
  │  ⑤ pageSvgMarkup() 复用 RowContent  src/lib/exportImage.tsx
  ▼
整页 SVG 文件 或 4× PNG 文件
```

关键设计：**③④⑤ 都是无副作用的纯计算/纯渲染**。`Worksheet` 里只存 `chars` + `layout`，
页和行在预览、打印、导出、统计页数时各自重新 `paginate`，所以任何两条出口不可能不一致。

---

## 1. 文字录入与过滤去重 —— `parseInput`

代码：`src/lib/input.ts:12-30`

```ts
const CJK_RE  = /[㐀-鿿]/;                  // U+3400-U+9FFF
const VALID_RE = /[㐀-鿿A-Za-z0-9]/;

export function parseInput(text, opts = {}): string[] {
  const valid = [...text].filter((ch) => VALID_RE.test(ch));   // 过滤
  const seen = new Set();
  const out = [];
  for (const ch of valid)                                       // 去重
    if (!seen.has(ch)) { seen.add(ch); out.push(ch); }
  if (opts.sortByStrokes && opts.strokeCountOf) {               // 可选排序
    out.sort((a, b) => (ca ?? Infinity) - (cb ?? Infinity));
  }
  return out;
}
```

**输入**：用户粘贴/键入的任意字符串。**输出**：`string[]`，每个元素一个码位字符。

### 1.1 参与的字段与分支

| 步骤 | 规则 | 边界 |
|---|---|---|
| 码位切分 | `[...text]` 按 Unicode 码点展开，不用 `str.split('')` | 避免代理对拆碎（虽然下面的正则也收不到 BMP 以外的字） |
| 过滤 | 白名单 `㐀-鿿`（CJK 扩展A + 常用汉字区）、`A-Z a-z`、`0-9` | **全角标点、空格、换行、emoji、扩展B 及以上生僻字（≥U+20000）一律丢弃**；只收 ASCII 字母数字，全角Ａ/１不收 |
| 去重 | `Set` 记录，**保留首次出现位置** | `'春天花 会花春' → ['春','天','花','会']`（见 `tests/unit/layout.test.ts:7`） |
| 排序 | 仅当 `sortByStrokes && strokeCountOf` 都存在时；笔画数 `undefined` 视为 `Infinity` 沉底 | 依赖 `Array.prototype.sort` 稳定性，无数据字之间保持原序 |

笔画数来源是 `src/lib/data.ts:45` 的 `strokeCountOf`，它问的是**本地笔顺数据**
（内置 `public/data/strokes.json` + 用户导入的 custom 数据），不是字典属性。

### 1.2 三个调用点（行为有差异，值得注意）

1. **首页新建** `src/pages/Home.tsx:18,22`：输入时实时 `parseInput` 出预览计数；
   点「生成字帖」时若结果为空数组 → `setError('请至少输入一个汉字、字母或数字')`，
   **不允许创建空字帖**（但空字帖在排版层是合法的，见 §4）。
2. **编辑器原文框** `src/pages/Editor.tsx:179-184`：每次按键都 `parseInput` 整体重算
   `ws.chars`。输入框里的 `text`（带空格的展示串）和 `ws.chars`（干净数组）是两份状态，
   进编辑器时用 `ws.chars.join(' ')` 反向初始化（`Editor.tsx:102`）。
3. **右栏「替换/删除」**（`Editor.tsx:189-215`）：不走 `parseInput`，手工改数组后
   **再手写一遍 Set 去重**——因为替换进来的字可能与已有字重复。这是一处重复逻辑，
   改去重规则时两处都要改。

---

## 2. 一个字为什么被拆成若干小格 —— `buildBlock`

代码：`src/lib/layout.ts:68-83`，类型见 `src/types.ts:44-48`。

字帖里一个字不是只出现一次，而是按教学顺序排一列小格：

```ts
type CellKind = 'model' | 'step' | 'trace' | 'blank';
type Cell = { kind: CellKind; stepK?: number };
type Block = { char: string; cells: Cell[] };
```

| 格类型 | 含义 | 由谁决定数量 |
|---|---|---|
| `model` | 例字（深色完整字形 + 拼音/部首/笔画/结构信息带） | `layout.mix.model`，只能 0 或 1 |
| `step` | 笔顺分解格，画到第 k 笔，格内标数字 k | `min(mix.strokeSteps, 该字笔画数)`，k 从 1 递增 |
| `trace` | 描红格（浅色完整字形） | `layout.mix.trace` |
| `blank` | 临写空格（只有格线） | `layout.mix.blank` |

**顺序固定**：例字 → 第1笔…第n笔分解 → 描红×m → 空格×b。默认配置
`{model:1, strokeSteps:3, trace:2, blank:4}`（`layout.ts:120-129`），
一个 5 画字正好 10 格。

### 2.1 格数的两个自适应点

- **笔顺分解数 = `min(配置值, 实际笔画数)`**（`layout.ts:74`）。1 画的「一」只要 1 个
  step 格，不会出现「第 2 笔」的空数据格。
- **组合总长硬截断到一行格数**（`layout.ts:82`）：
  `cells.slice(0, Math.max(1, layout.perLine))`。
  配 8 分解 + 8 描红 + 8 空格也不会撑破一行——**从尾部丢格**（先丢空格，再丢描红，
  极端情况下分解格、例字也可能被丢；`perLine=1` 时一个字只剩一个 model 格）。
  这是「一个字的组合比一整行还宽」这一边界的**唯一防线**，在拆格阶段就解决，
  排版阶段不再处理。

### 2.2 无笔顺数据的汉字怎么对待（核心教学约束）

`layout.ts:71`：`const noStrokeHanzi = strokeCount == null && isCjk(char);`

- 是汉字但本地查不到笔顺 → **不出 step 格（§73 的 `strokeCount != null` 守卫）、
  不出 trace 描红格**（§77），只保留 `model` + `blank`。注释明说原因：
  「不提供描红，避免误教」。
- **字母/数字**同样没有笔顺数据，但 `isCjk` 为 false，所以不触发这条：
  它们照常给描红格（用字体轮廓描灰，见 §5），只是永远没有分解格。
- 例字仍然要画，画的时候 `GlyphAt` 对无数据汉字走**系统字体回退**并打
  `data-fallback="font"` 标记（`paint.tsx:135-149`，字号 82；字母数字 64），
  绝不伪造笔画路径；model 格底部再盖一枚红色「无笔顺数据」标注
  （`paint.tsx:217-223`）。
- 信息带也区别对待：`paint.tsx:204` 的条件是 `strokes || !isCjk(char)`——
  无数据汉字不显示拼音/部首那一条；字母数字照常（可显示拼音库返回空则不显示）。

笔顺数据从哪来：`getStrokes`（`src/lib/data.ts:39-43`）先查内置
`strokes.json`（hanzi-writer v1 格式），再查 localStorage 里的 `customStrokes`；
空数组/非数组一律视为无数据。用户可在编辑器「导入笔顺数据」补字
（`importStrokes`，三种 JSON 形态，`data.ts:56-73`），导入后该字下次排版即按
有数据处理。

---

## 3. 一行格数与一页行数的上限怎么算 —— 页面几何 + `clampLayout`

代码：`src/lib/layout.ts:5-62`

A4 纸面常量（mm）：

```
页宽 210，页高 297；左右边距各 5，上 8 下 8；页眉 headerMm = 12
可用宽 usableWMm   = 210 - 5 - 5            = 200
格区高 rowsAreaHMm = 297 - 8 - 8 - 12       = 269
```

```ts
maxPerLine(cellMm)        = max(1, floor(200 / cellMm))
maxLines(cellMm, gapMm)   = max(1, floor(269 / (cellMm * 1.2 + gapMm)))
```

- 行高系数 `ROW_FACTOR = 1.2`（`paint.tsx:11-12`）：每格 100 unit 高，
  上方还有 20 unit 的拼音信息带，每行 120 unit，即物理行高 = 格宽 × 1.2，
  再加 `lineGapMm` 行距才是行间距 pitch。
- 默认 20mm 格：每行 `floor(200/20)=10` 格；每页 `floor(269/(24+2))=10` 行。
- 上限 35mm 格 + 0 行距时每页 `floor(269/42)=6` 行（测试 `layout.test.ts:52`）。

`clampLayout`（`layout.ts:50-62`）在**每次排版前**把用户配置钉进合法域：

| 字段 | 合法域 | 越界处理 |
|---|---|---|
| `cellMm` | 12–35，取整 | 超界截断 |
| `lineGapMm` | 0–12，取整 | |
| `perLine` | 1 … `maxPerLine(cellMm)`，取整 | 调小格宽后每行上限变大，反之被压回 |
| `lines` | 1 … `maxLines(cellMm, gap)`，取整 | 改格宽/行距后同样重新夹逼 |
| `mix.model` | 0/1 | |
| `mix.strokeSteps/trace/blank` | 0–8 | |

编辑器所有改动经 `updateLayout → clampLayout` 入库（`Editor.tsx:176-177`），
`paginate` 内部**再 clamp 一次**（`layout.ts:94`），双保险，localStorage 里
的旧脏数据也不会算出版面溢出。

---

## 4. 换行与换页在哪一步发生 —— `paginate`

代码：`src/lib/layout.ts:89-118`。拆格和排版是同一函数内先后两步，但换行决策只认
**格数**，不认 mm。

```ts
// 第一遍：贪心装行
let current: Row = [];
let used = 0;
for (const ch of chars) {
  const block = buildBlock(ch, clamped, strokeCountOf(ch));
  if (used > 0 && used + block.cells.length > clamped.perLine) {
    rows.push(current);          // 放不下：先封行
    current = [block];           // 新字另起一行
    used = block.cells.length;
  } else {
    current.push(block);
    used += block.cells.length;
  }
}
if (current.length > 0) rows.push(current);

// 第二遍：按 lines 硬切页
for (let i = 0; i < rows.length; i += clamped.lines)
  pages.push(rows.slice(i, i + clamped.lines));

if (pages.length === 0) pages.push([]);   // 空内容也给一页
```

### 4.1 不能破的约束（测试钉死）

1. **不拆字**：一个 Block 的所有格必在同一行——换行判断发生在「放入整个 block 之前」，
   一个 block 永远整体移动（`layout.test.ts:129-139`）。
2. **不拆行跨页**：第二遍只是按 `lines` 切片，整行整体归页。
3. **行长守恒**：每行格数 ≤ `perLine`。
4. **字序守恒**：所有页拼回来的字序列与输入 `chars` 完全相等
   （`layout.test.ts:114-115`）。

### 4.2 判定式的两个细节

- `used > 0` 守卫：一行的**第一个** block 不触发换行。理论上 block 已被 §2.2 的
  slice 限到 ≤ `perLine`，所以它一定放得下；若没有这道截断，这里会出现
  「block 比整行宽 → 每行只放一个字还不停空转封行」的退化。
- 判定是**严格大于**：恰好放满不换行，下一个字自然被挤到新行。
- 块宽随笔画数变化：1 画的「一」默认配 8 格，2 画字 10 格——
  `['一','人','八']` 排版结果是三行（`layout.test.ts:141-146`），
  每行的容量是按实际块宽贪心算的，不是按字数。

### 4.3 边界清单

- **空内容**：`chars=[]` → 零行 → 兜底 `[[]]`，即一页空白格子模板，可直接打印
  （`layout.ts:116`，测试 `layout.test.ts:148-151`）。注意首页 UI 拦住了空字帖的
  创建，但删除全部字之后编辑器里仍能走到这里。
- **一个字比一行宽**：在 `buildBlock` 尾部截断解决（§2.1），排版层不感知。
  代价是被丢掉的格静默消失，UI 没有提示——配置极格数（如 35mm、每行 1–5 格）时
  用户会发现描红/空格变少。
- **尾行不满、尾页不满**：正常留白，不补空行。
- **页数变化导致页码越界**：`Editor.tsx:136-139` 用 effect 把导出页码选择器夹回
  `pageCount-1`；导出函数内部还会再夹一次（§6）。

---

## 5. 渲染：预览与打印为什么是同一份

### 5.1 PageView —— 屏幕与打印的唯一页面组件

`src/components/PageView.tsx:80-148`：

- 进入即 `clampLayout` + `paginate(strokeCountOf)`（§3/§4 的现算结果），
  映射出全部 `pages`，每页一个 `.sheet` div，尺寸直接用 `210mm×297mm` 和 mm 边距，
  靠 CSS 在打印时 1:1 落纸（打印提示要求关闭浏览器「缩放/适应页面」，
  第 1 页还画了 100mm 校验尺 `Ruler`，`PageView.tsx:10-44`）。
- 每行一个独立 SVG：宽 `perLine*cellMm mm`，高 `cellMm*1.2 mm`，
  `viewBox="0 0 perLine*100 120"`。内部坐标 1 unit = cellMm/100 mm，
  格占 y∈[20,120]，上方 20 unit 是信息带。
- **打印视图**（`src/pages/PrintView.tsx`）渲染的就是 `<PageView plain />`，
  `plain` 仅关掉选中蓝框和点击选字（`paint.tsx:244-258`），格线/字形/信息带
  一个像素都不差——「所见即打印所得」。`?autoprint=1` 时等 `document.fonts.ready`
  再延迟 300ms 调 `window.print()`，避免字体没加载完用回退字体出纸。
- 点击命中也复用块几何：`blockRanges`（`PageView.tsx:61-68`）按每块格数×100
  算 unit 区间，与绘制时的 x 推进方式一致。

### 5.2 RowContent —— 格内画什么的唯一实现

`src/components/paint.tsx:186-264`，按 block → cell 两层循环，x 每格 +100：

- 每格先铺 `GridLines`：田/米/回宫/方格/横线/拼音四线格六种
  （`paint.tsx:35-92`，`grid='line'` + `fourLine` 时画四条拼音线）。
- `model`：`GlyphAt` 深色完整字形 + `CellInfo`（拼音 15 号朱红，部首·n画·结构
  10 号灰）+ 无数据汉字的红色标注。
- `step`：`GlyphAt upto=k`——只画前 k 笔，已完成笔灰色、当前笔深色加粗 1.15×
  （`paint.tsx:117-131`），右上角标 k。
- `trace`：完整字形用 `layout.traceColor`（浅/中/深三档预设）描边。
- `blank`：只留底格，什么都不画。
- 字形定位用统一的 `glyphTransform`（`paint.tsx:29-32`）：把 hanzi-writer 的
  1024 em-box（y 向上）坐标翻成格心居中、unit 制坐标；笔顺播放器
  `StrokePlayer.tsx` 也复用同一个变换，保证三处字形完全重合。

多音字：`pinyinResolver`（`PageView.tsx:46-53`）取 `readingsOf` 的全部读音，
下标取 `worksheet.pinyinChoice[ch] ?? 0`，并对越界下标做
`Math.min(idx, len-1)` 夹回——删掉读音数据等情况下不会取到 undefined。

---

## 6. 导出整页图：与预览同一份内容的第三条出口

代码：`src/lib/exportImage.tsx`。

`pageSvgMarkup(worksheet, pageIndex)`（§21-62）做的事：

1. `clampLayout` → `paginate` → 取页，与预览**逐字相同的入参和函数**；
2. `pi = max(0, min(pageIndex, pages.length-1))` —— **页码越界夹到合法页**，
   负数归 0，超大数归末页。空内容时 `pages=[[]]]`，任何页码都导出那一页空白；
3. 页眉（标题经 `escapeXml` 转义，右侧「第 x 页 / 共 y 页」）+ 白底矩形；
4. 对该页每一行直接 `renderToStaticMarkup(<RowContent .../>)` —— **就是预览里
   那个 RowContent**——外面包 `<g transform="translate(左边距, y) scale(格宽px/100)">`
   摆到 A4 坐标，行间推进 `cellMm*1.2 + gap`，与 PageView 的 mm 几何一一对应。

所以「预览与导出的图为什么是同一份」：两者共用同一个纯函数 `paginate` 和同一个
渲染原语 `RowContent`/`GridLines`/`GlyphAt`，导出只是把浏览器里的 DOM-SVG 换成
服务端静态标记串，外加 mm→px（`96/25.4`）的坐标换算。差异只有载体相关部分：

| | 预览/打印（PageView） | 导出（pageSvgMarkup） |
|---|---|---|
| 页眉 | HTML div，第 1 页含 100mm 校验尺 | SVG `<text>`，无校验尺 |
| 页脚 | HTML `.sheet-footer` | SVG `<text>` |
| 选中态 | 可选蓝框 | 无 |
| 行容器 | 每行一个 `<svg>` | 每页一个 `<svg>`，行用 `<g>` 平移 |
| 格子与字形 | **RowContent，完全相同** | **RowContent，完全相同** |

- **SVG 导出**：标记串直接存 Blob 下载（`exportSvg`）。
- **PNG 导出**：同一标记串 → `Blob URL` → `Image` → 画到 canvas，尺寸
  `A4 px × scale`（默认 4），先铺白底再 `drawImage`（透明字形之外不留透明边），
  `toBlob` 下载；SVG 加载失败/无 2d context/编码失败都有 reject 分支。

**一个小的不一致边界**：页眉里的页码用的是夹过的 `pi+1`，但**下载文件名**用的是
未夹的原始 `pageIndex+1`（`exportImage.tsx:88,122`）。传入越界页码时，
文件名叫「第 N 页」，里面画的却是第 1 页/末页。正常 UI 流程中选择器已被
`Editor.tsx:137-139` 提前夹住，触发不到，但属于直接调用 API 时会踩到的坑。
文件名另有 `baseName` 把 `\\ / : * ? " < > |` 替换成下划线，防 Windows 非法名。

---

## 7. 端到端字段与约束速查

```
Worksheet（storage.ts 持久化，key: app022:worksheets）
  id / title / updatedAt
  chars        ← parseInput 产物；替换/删除后手工再去重
  layout       ← 入库前 clampLayout；paginate 内再 clamp
  pages        ← 只是防抖自动保存时记下的元数据（Editor.tsx:122-128），渲染不读它
  pinyinChoice ← 多音字下标，读取处夹边界
  sortByStrokes
```

不可破坏的不变量：

1. `chars` 无重复、只含白名单字符、顺序 = 首次出现序（或笔画稳定序）；
2. 每个 block 格数 ≤ `perLine`，cell 顺序 model→step→trace→blank；
3. step 格数 ≤ 真实笔画数；无笔顺数据汉字没有 step/trace，例字用字体并红标；
4. block 不跨行、row 不跨页、行格数 ≤ `perLine`、页行数 ≤ `lines`；
5. 任何配置都被 clamp 到纸面几何允许的范围内；
6. 预览/打印/导出三处由同一份 `paginate` 数据 + 同一个 `RowContent` 产出。

最容易踩的边界汇总：

| 边界 | 处置位置 | 行为 |
|---|---|---|
| 全标点/emoji/空串输入 | `parseInput` | 过滤成空数组；首页禁止创建，编辑器允许存在（渲染空白页） |
| 扩展B 生僻字、全角字母数字 | `VALID_RE` | 静默不收录 |
| 一个字组合宽于一行 | `buildBlock` slice | 从尾部静默丢格，无 UI 提示 |
| `perLine/lines` 超过纸面上限 | `clampLayout` | 压回 `maxPerLine/maxLines` |
| 格宽/行距改动使当前行数超限 | `clampLayout` | 重新分页，页码选择器自动夹回 |
| 空内容打印/导出 | `paginate` 兜底 | 至少一页（可全空） |
| 导出页码越界 | `pageSvgMarkup` | 内容夹回合法页；文件名不夹（小坑） |
| 多音字选择下标越界 | `pinyinResolver` | 夹到最后一个读音 |
| 无笔顺汉字 | buildBlock + paint + player | 无分解/无描红/红色标注/字体回退/播放器提示「无笔顺数据」 |
| 笔顺 JSON 脏数据 | `getStrokes`/`importStrokes` | 非数组或空数组视为无数据，不写入 |
