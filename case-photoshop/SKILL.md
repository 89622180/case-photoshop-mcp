---
name: case-photoshop
description: 把透明手机壳产品图替换进电商手持实拍 PSD 底图(mockup)。Replace a phone case inside a layered lifestyle-mockup PSD (手指/手机壳/背景) with a transparent product render, using the Photoshop MCP. Use when the user gives a 手机壳底图/模板 PSD plus a transparent case PNG, asks to 换壳/替换手机壳/换图案/盖住红色标记, or re-runs a mockup with new artwork.
argument-hint: <模板PSD路径> <手机壳透明PNG路径>
---

# 手机壳底图换壳

## Overview

电商手机壳卖家的手持实拍模板：PSD 从上到下三层 **手指 / 手机壳 / 背景**。任务是只换中间那层，
让新壳贴合原壳的倾斜与位置，并且**拇指仍然压在壳前面**。

## 前置依赖：先装 Photoshop MCP

本技能全部动作走 `photoshop_*` 工具，**没装 MCP 就不能用**（换个 agent/机器就全是 tool not found）。
服务来自 <https://github.com/alisaitteke/photoshop-mcp>（非官方、社区维护，与 Adobe 无隶属）。

前置条件（原文）：**Photoshop 必须已经开着**（Windows 或 macOS，2012 及以上任意版本）+ **Node.js 18+**。

注册到 MCP 客户端：

```json
{
  "mcpServers": {
    "photoshop": {
      "command": "npx",
      "args": ["-y", "@alisaitteke/photoshop-mcp"]
    }
  }
}
```

Claude Code 一行装：`claude mcp add photoshop -- npx -y @alisaitteke/photoshop-mcp`
（自带对话窗口：`npx -p @alisaitteke/photoshop-mcp ui`）
Qoder 侧用 `/mcp-config` 交互式写上面那段 JSON 即可。

装完先 `photoshop_ping` 确认连通；它回 `extendscript_timeout` 说明 Photoshop 正卡在脚本或弹窗上，
反复 ping 到成功再继续，别硬推下一个工具调用。

## 第一步永远是判断「手机壳」层是什么

打开 PSD 后先 `photoshop_get_preview` 看，**「手机壳」层有两种可能**：

1. **真实壳的抠图** → 它本身就是被替换对象，新壳盖住它即可。
2. **纯色圆角矩形标记（通常是正红色）** → 这是用户手画的「必须完全盖住」区域，
   不是图案。它的倾角是用户随手画的，**可能和照片里真实壳的角度不一致——以红框为准**。

判断方法：预览里看到一块无纹理的高饱和纯色矩形 = 情况 2。

红色标记的存在让验证变得极其简单：**成品里看不到红，就是覆盖到位**。不要另建测量流程。

## 主流程

1. `photoshop_open_image` 打开模板 → 记下返回的 `document_id`。
2. `photoshop_get_layers`（**必须传 `document_id`**）确认层名与顺序，读出「手机壳」层 bounds。
3. **`Read` 模板同名 `.ini` 预设**（见「预设文件」）—— 命中就直接套角度/缩放，跳过迭代。
4. `photoshop_place_image` 置入新壳 PNG。
5. `photoshop_move_layer_to_position` 把新层放到 **手指 BELOW**（关键，见下）。
6. 缩放 → 旋转 → 平移（见「变换」）。
7. 对齐期把新层 `photoshop_set_layer_opacity` 设 **50%**，能直接透视看到底下差多少；调好再设回 100%。
8. **用原生分辨率验证**：`photoshop_get_preview` 传 `max_dimension_px: 1125`（或文档实际宽度）。
   缩小过的预览会把 2–3px 的红边/原壳边直接抹掉。
9. 另存为**新文件** + 导出 JPG。

## 变换

### 五个必须知道的坑

- **`photoshop_place_image` 会裁掉全透明边缘**：返回的 bounds 是**不透明内容**的外接框，
  不是 PNG 的画布尺寸。800×800 的图常返回 374×740 之类。别按 PNG 尺寸算缩放。
- **壳身 ≠ 不透明外接框**：壳图右侧常有侧键凸起，外接框的最右列是按键而不是壳身，
  壳身向内缩十几像素（左右都可能有小凸起）。所以「外接框刚好等于红框」仍会露一条细红线 ——
  必须再放大 8~10% 把内缩吃掉。先 `Read` 那张 PNG 看清凸起在哪，比猜快。
- 置入的新层落在**当前活动层之上**。若活动层是「手机壳」，新壳会盖住手指 →
  必须 `move_layer_to_position(BELOW, "手指")`。
- **`photoshop_rotate_layer` 正值 = 顺时针**。它和 `photoshop_scale_layer(centerAnchor:true)`
  都绕层中心，**中心不变** → 所以平移量只在最后算一次。
- **红框的 bounds 反推不出倾角**：同一个外接框能被无数组（角度 × 真实尺寸）拟合出来，
  别拿它算角度。顺带一条有用的规律：**顺时针旋转会同时放大壳的 bbox，逆时针两头都缩小** ——
  倾角拿不准时 `rotate 0°` 最利于覆盖，宁可略微不自然也别露红。

### 定位公式

置入后内容尺寸 `W×H`、左上 `(0,0)`，中心即 `(W/2, H/2)`。目标中心 `(cx,cy)` 取
「手机壳」层（或红框）bounds 的中点：

```
deltaX = cx - W/2
deltaY = cy - H/2
```

顺序：`scale_layer` → `rotate_layer` → `move_layer`。

### 怎么调（别猜，按症状调）

| 症状 | 病因 | 动作 |
|---|---|---|
| 红/原边只在**一侧**露出 | 平移偏了 | `move_layer` 往那边推 |
| **左上 + 右下**同时露出（对角两侧） | 旋转角度不对 | 改角度：若加大顺时针后红边变**更粗**，说明要**逆时针**回退（负值） |
| 四边都露一点 / 整体偏小 | 尺寸不够 | `scale_layer` +3~4% |
| 单边一条细缝 | 角度和尺寸都差一点 | 先修角度再补 2~3% 尺寸 |

一轮 = 一次变换 + 一次预览。**2–3 轮就该收敛**；超过 4 轮说明层判断错了，回头重看第 1 步。

## 保存

- **绝不覆盖用户的模板**（那是复用资产）。另存为 `<模板名>-新壳.psd`；同一模板换第二张壳图时
  用 `-新壳2` 之类的后缀，别覆盖上一轮的成品。
- 「手机壳」若是红色标记层 → **保留、不删**，它已被完全遮住，下次换图还能当对齐模板。
  新壳层改名 `手机壳-新`（避免同名）。真实壳抠图那种才按用户指示删除。
- 新壳是智能对象：以后换图案用 `photoshop_replace_smart_object_contents`（保留全部变换，**零迭代**，
  只需一次原生分辨率验证）。前提是模板里已有一层 `手机壳-新` —— 这时别再 `place_image`，
  否则会在模板上叠出第二层壳。此路径不产生 `content` 尺寸，预设命中只核 `canvas` + `marker`。
- 再 `photoshop_export_as` 出一份 JPG（`quality: 95`；这个工具是 0–100 刻度，别填 11）供上架。
- 存完 `ls -l` 核对时间戳/大小，并确认模板文件没被改动。

## 预设文件（每张底图一个 .ini，技能不背预设）

预设跟着模板走，不写在本技能里：**模板同名 `.ini`，与 PSD 同目录** ——
`D:\...\手机壳底图.psd` → `D:\...\手机壳底图.ini`。

流程：

1. 打开模板、读出「手机壳」层 bounds 之后，**先 `Read` 那个 .ini**。
2. **命中判定（三条全中才直接套，否则回退目视迭代）**：
   - `canvas` == 当前文档尺寸
   - `marker` == 当前「手机壳」层 bounds
   - `place_image` 回的内容尺寸 == ini 里的 `content`
3. 命中 → 按 ini 的 `rotate` / `scale` / `center` 一轮到位，**仍要做原生分辨率验证**（预设只省迭代，不省验证）。
4. ini 不存在 → 走目视迭代；**收敛后立即 `Write` 写入 ini**（首次写入预设）。
5. 迭代结果与 ini 记的值不一致（换了别的壳图、或模板被改过）→ **先问用户「是否更新预设」，同意后再改 ini**，不要静默覆盖。

格式（键名固定，值用逗号/`x` 分隔，便于肉眼核对）：

```ini
[template]
canvas=1080x1440
marker=364,220,923,1308

[case]
content=374x740
rotate=0
scale=160
center=643.5,764
```

`marker` 变了 = 模板被改过，预设作废，重测并询问是否更新。

## 不要做的事

不要为角度/尺寸写像素分析脚本（PowerShell System.Drawing、最小二乘拟合边、求单应矩阵等）。
用户明确要求过「大致角度就可以了」「太过复杂了…简单点」，而且实拍壳的轮廓边是圆角+透视，
拟合出来的角度自相矛盾、还不如一张截图。**目视迭代就是正确做法。**

## 偶用的 JSX

读全部层 bounds（`asUnit('px')` 不存在，必须先设像素单位）：

```javascript
app.preferences.rulerUnits = Units.PIXELS;
var d = app.activeDocument, out = [];
for (var i = 0; i < d.layers.length; i++) {
  var b = d.layers[i].bounds;
  out.push({n: d.layers[i].name, l: Number(b[0]), t: Number(b[1]),
            r: Number(b[2]), b: Number(b[3])});
}
return out;
```

导出单层带透明（本版本 `ExportOptionsPNG24` 无构造函数，用 `saveAs` + `PNGSaveOptions`；
只在临时副本上 `trim(TrimType.TRANSPARENT)`，别动原文档）：

```javascript
var dup = app.activeDocument.duplicate('tmp');
app.activeDocument = dup;
var o = new PNGSaveOptions(); o.interlaced = false;
dup.saveAs(new File('<绝对路径>.png'), o, true, Extension.LOWERCASE);
dup.close(SaveOptions.DONOTSAVECHANGES);
```
