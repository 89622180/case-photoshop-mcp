# case-photoshop

电商手机壳卖家的 **Agent 技能（Skill）**：把透明壳产品图换进「**手指 / 手机壳 / 背景**」三层手持实拍
PSD 底图，只替换中间那层，拇指仍然压在壳前面，并保证用户手画的**红色标记被完全盖住**。

本仓库是技能本体（`case-photoshop/SKILL.md`），**不是 MCP 服务器** —— 它依赖下面的 Photoshop MCP。

## 前置：先装 Photoshop MCP

技能的全部动作都走 `photoshop_*` 工具，服务来自
<https://github.com/alisaitteke/photoshop-mcp>（非官方、社区维护，与 Adobe 无隶属）。

要求：**Photoshop 必须已经开着**（Windows 或 macOS，2012 及以上任意版本）+ **Node.js 18+**。

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

Claude Code 一行装：

```bash
claude mcp add photoshop -- npx -y @alisaitteke/photoshop-mcp
```

装完先调 `photoshop_ping` 确认连通。

## 安装技能

把 `case-photoshop/` 整个目录放进你的技能目录即可：

| 客户端 | 目标路径 |
|---|---|
| Qoder | `~/.qoder-cn/skills/case-photoshop/` |
| Claude Code | `~/.claude/skills/case-photoshop/` |

或直接下载 [Releases](../../releases) 里的 zip 解压。

## 用法

```
/case-photoshop <模板PSD路径> <手机壳透明PNG路径>
```

技能会自己判断「手机壳」层是**真实壳抠图**还是**纯色红框标记**，然后置入新壳、移到手指下方、
按红框对齐缩放旋转，最后用**原生分辨率**验证是否还有红色露出——看不到红就是覆盖到位。

## 预设机制（重要）

每张底图的对齐参数**不写死在技能里**，而是存在 PSD **同目录的同名 `.ini`**：

```
手机壳底图.psd   →   手机壳底图.ini
```

```ini
[template]
canvas=1125x1482
marker=462,434,923,1180

[case]
content=374x740
rotate=13
scale=103
center=692.5,807
```

规则：

1. 技能先 `Read` 这个 ini，命中（`canvas` + `marker` + `content` 三项全对）就**一轮到位、不再迭代**；
2. ini 不存在 → 目视迭代收敛后**首次写入**；
3. 迭代结果与 ini 不一致（换了壳图 / 模板被改过）→ **先问是否更新预设**，不静默覆盖；
4. `marker` 变了 = 模板被改，预设作废，重测。

所以 `.ini` 属于**素材侧资产**，不随技能包分发；换机器时和 PSD 一起拷走，
否则技能会当成「预设不存在」重新迭代一轮。

## 已知的坑

- `place_image` 会裁掉全透明边缘，返回的是**不透明内容**外接框，不是 PNG 画布尺寸
- 壳图右侧常有侧键凸起，外接框最右列是按键而非壳身 → 「刚好等于红框」仍会露一条细红线，需再放大 8~10%
- 红框的 bounds **反推不出倾角**（同一外接框可由无数组角度×尺寸拟合）→ 目视迭代，倾角拿不准时 `0°` 最利于覆盖
- 缩小过的预览会抹掉 2–3px 的红边，验证必须用原生分辨率

## License

未指定。
