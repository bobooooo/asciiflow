# ASCIIFlow MCP Server Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将 ASCIIFlow 绘图引擎封装为 MCP Server，让 AI agent 通过工具调用从 PRD 生成 ASCII 低保真线框图。

**Architecture:** 新建 `mcp/` 目录，独立 Node.js MCP Server，直接复用 `client/layer.ts`、`client/vector.ts`、`client/draw/utils.ts` 等纯 TS 核心逻辑（无 DOM/React 依赖）。Server 维护内存中的 `Layer` 实例，暴露 7 个 MCP 工具供 agent 调用，最终通过 `canvas_export` 返回 ASCII 文本。

**Tech Stack:** TypeScript 5.8, `@modelcontextprotocol/sdk`, Node.js ESM, tsx (dev runner)

---

## Task 1: 初始化 MCP 包结构

**Files:**
- Create: `mcp/package.json`
- Create: `mcp/tsconfig.json`

**Step 1: 创建 `mcp/package.json`**

```json
{
  "name": "asciiflow-mcp",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsx src/index.ts"
  },
  "imports": {
    "#asciiflow/*": "../*"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "tsx": "^4.7.0",
    "typescript": "^5.8.0",
    "@types/node": "^22.0.0"
  }
}
```

**Step 2: 创建 `mcp/tsconfig.json`**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "paths": {
      "#asciiflow/*": ["../*"]
    }
  },
  "include": ["src/**/*"]
}
```

**Step 3: 安装依赖**

```bash
cd mcp && npm install
```

Expected: `node_modules/` 创建，`@modelcontextprotocol/sdk` 安装成功。

**Step 4: Commit**

```bash
git add mcp/package.json mcp/tsconfig.json
git commit -m "feat(mcp): init mcp package structure"
```

---

## Task 2: 实现画布核心逻辑（Canvas Engine）

**Files:**
- Create: `mcp/src/canvas.ts`

这是 MCP server 的核心，封装 Layer 操作，不依赖 React/store。

**Step 1: 创建 `mcp/src/canvas.ts`**

```typescript
import { Layer } from "#asciiflow/client/layer.js";
import { Vector } from "#asciiflow/client/vector.js";
import { layerToText } from "#asciiflow/client/text_utils.js";
import { UNICODE } from "#asciiflow/client/constants.js";
import { line } from "#asciiflow/client/draw/utils.js";
import { Box } from "#asciiflow/client/common.js";

export class Canvas {
  private layer = new Layer();

  reset() {
    this.layer = new Layer();
  }

  drawBox(x: number, y: number, w: number, h: number, label?: string) {
    const start = new Vector(x, y);
    const end = new Vector(x + w - 1, y + h - 1);
    const box = new Box(start, end);

    // 四条边
    for (let px = box.left(); px <= box.right(); px++) {
      this.layer.set(new Vector(px, box.top()), UNICODE.lineHorizontal);
      this.layer.set(new Vector(px, box.bottom()), UNICODE.lineHorizontal);
    }
    for (let py = box.top(); py <= box.bottom(); py++) {
      this.layer.set(new Vector(box.left(), py), UNICODE.lineVertical);
      this.layer.set(new Vector(box.right(), py), UNICODE.lineVertical);
    }
    // 四个角
    this.layer.set(box.topLeft(), UNICODE.cornerTopLeft);
    this.layer.set(box.topRight(), UNICODE.cornerTopRight);
    this.layer.set(box.bottomRight(), UNICODE.cornerBottomRight);
    this.layer.set(box.bottomLeft(), UNICODE.cornerBottomLeft);

    // 可选标签（居中显示在顶部边框内）
    if (label && w > 2) {
      const innerWidth = w - 2;
      const truncated = label.slice(0, innerWidth);
      const labelX = x + 1 + Math.floor((innerWidth - truncated.length) / 2);
      for (let i = 0; i < truncated.length; i++) {
        this.layer.set(new Vector(labelX + i, y), truncated[i]);
      }
    }
  }

  drawLine(x1: number, y1: number, x2: number, y2: number) {
    const lineLayer = line(new Vector(x1, y1), new Vector(x2, y2), true);
    this.layer.setFrom(lineLayer);
  }

  drawArrow(x1: number, y1: number, x2: number, y2: number) {
    this.drawLine(x1, y1, x2, y2);
    // 箭头方向取决于终点相对起点的位置
    const dx = x2 - x1;
    const dy = y2 - y1;
    let arrowChar: string;
    if (Math.abs(dx) >= Math.abs(dy)) {
      arrowChar = dx >= 0 ? UNICODE.arrowRight : UNICODE.arrowLeft;
    } else {
      arrowChar = dy >= 0 ? UNICODE.arrowDown : UNICODE.arrowUp;
    }
    this.layer.set(new Vector(x2, y2), arrowChar);
  }

  addText(x: number, y: number, text: string) {
    const lines = text.split("\n");
    for (let row = 0; row < lines.length; row++) {
      for (let col = 0; col < lines[row].length; col++) {
        const ch = lines[row][col];
        if (ch !== " ") {
          this.layer.set(new Vector(x + col, y + row), ch);
        }
      }
    }
  }

  export(): string {
    return layerToText(this.layer);
  }
}
```

**Step 2: 验证无 DOM 依赖**

检查 `client/common.ts` 是否有 DOM 引用：

```bash
grep -n "document\|window\|HTMLElement" /Users/xingbin/Documents/Study/asciiflow/client/common.ts
```

Expected: 无输出（无 DOM 依赖）。

**Step 3: Commit**

```bash
git add mcp/src/canvas.ts
git commit -m "feat(mcp): add Canvas engine wrapping Layer/Vector primitives"
```

---

## Task 3: 实现 MCP Server 入口

**Files:**
- Create: `mcp/src/index.ts`

**Step 1: 创建 `mcp/src/index.ts`**

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";
import { Canvas } from "./canvas.js";

const canvas = new Canvas();
const server = new McpServer({
  name: "asciiflow",
  version: "0.1.0",
});

server.tool(
  "canvas_new",
  "创建/重置一个空白画布",
  {},
  async () => {
    canvas.reset();
    return { content: [{ type: "text", text: "Canvas reset." }] };
  }
);

server.tool(
  "draw_box",
  "在画布上绘制一个矩形框。x/y 是左上角坐标，w/h 是宽高（字符单位）。label 可选，显示在顶边框中央。",
  {
    x: z.number().int().describe("左上角 x 坐标"),
    y: z.number().int().describe("左上角 y 坐标"),
    w: z.number().int().min(3).describe("宽度（至少 3）"),
    h: z.number().int().min(3).describe("高度（至少 3）"),
    label: z.string().optional().describe("可选标签文字"),
  },
  async ({ x, y, w, h, label }) => {
    canvas.drawBox(x, y, w, h, label);
    return { content: [{ type: "text", text: `Box drawn at (${x},${y}) size ${w}x${h}.` }] };
  }
);

server.tool(
  "draw_line",
  "在两点之间绘制一条折线（先水平后垂直）",
  {
    x1: z.number().int(),
    y1: z.number().int(),
    x2: z.number().int(),
    y2: z.number().int(),
  },
  async ({ x1, y1, x2, y2 }) => {
    canvas.drawLine(x1, y1, x2, y2);
    return { content: [{ type: "text", text: `Line drawn from (${x1},${y1}) to (${x2},${y2}).` }] };
  }
);

server.tool(
  "draw_arrow",
  "在两点之间绘制带箭头的连线，箭头指向终点",
  {
    x1: z.number().int(),
    y1: z.number().int(),
    x2: z.number().int(),
    y2: z.number().int(),
  },
  async ({ x1, y1, x2, y2 }) => {
    canvas.drawArrow(x1, y1, x2, y2);
    return { content: [{ type: "text", text: `Arrow drawn from (${x1},${y1}) to (${x2},${y2}).` }] };
  }
);

server.tool(
  "add_text",
  "在指定坐标添加文字（支持 \\n 换行）",
  {
    x: z.number().int(),
    y: z.number().int(),
    text: z.string().describe("要添加的文字，支持 \\n 换行"),
  },
  async ({ x, y, text }) => {
    canvas.addText(x, y, text);
    return { content: [{ type: "text", text: `Text added at (${x},${y}).` }] };
  }
);

server.tool(
  "canvas_export",
  "导出当前画布为 ASCII 文本",
  {},
  async () => {
    const result = canvas.export();
    return { content: [{ type: "text", text: result || "(empty canvas)" }] };
  }
);

server.tool(
  "canvas_preview",
  "预览当前画布状态（与 canvas_export 相同，用于中间检查）",
  {},
  async () => {
    const result = canvas.export();
    return { content: [{ type: "text", text: result || "(empty canvas)" }] };
  }
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

**Step 2: Commit**

```bash
git add mcp/src/index.ts
git commit -m "feat(mcp): implement MCP server with 7 drawing tools"
```

---

## Task 4: 处理模块路径问题

**Files:**
- Modify: `mcp/package.json`（添加 imports 映射）

`client/` 文件使用 `#asciiflow/*` 路径别名，Node.js ESM 需要在 `package.json` 的 `imports` 字段中配置。

**Step 1: 检查 `client/common.ts` 路径**

```bash
head -5 /Users/xingbin/Documents/Study/asciiflow/client/common.ts
```

**Step 2: 确认根 `package.json` 的 imports 配置**

根目录 `package.json` 已有：
```json
"imports": { "#asciiflow/*": "./*" }
```

`mcp/` 是子目录，需要映射到父目录。更新 `mcp/package.json` 的 imports：

```json
"imports": {
  "#asciiflow/*": "../*"
}
```

**Step 3: 构建测试**

```bash
cd mcp && npm run build 2>&1 | head -30
```

Expected: 编译成功，`dist/` 目录生成。

**Step 4: Commit**

```bash
git add mcp/package.json
git commit -m "fix(mcp): configure #asciiflow path alias for Node.js ESM"
```

---

## Task 5: 本地测试 MCP Server

**Step 1: 用 MCP Inspector 测试**

```bash
cd mcp && npx @modelcontextprotocol/inspector node dist/index.js
```

Expected: Inspector 启动，列出 7 个工具。

**Step 2: 手动验证工具调用序列**

在 Inspector 中依次调用：
1. `canvas_new` → "Canvas reset."
2. `draw_box` `{x:0, y:0, w:20, h:5, label:"Login"}` → Box drawn
3. `add_text` `{x:2, y:2, text:"Username:"}` → Text added
4. `draw_box` `{x:0, y:7, w:20, h:5, label:"Dashboard"}` → Box drawn
5. `draw_arrow` `{x:10, y:5, x2:10, y2:7}` → Arrow drawn
6. `canvas_export` → 返回 ASCII 线框图

Expected 输出类似：
```
┌──────Login─────┐
│                │
│ Username:      │
│                │
└────────────────┘
         ▼
┌────Dashboard───┐
│                │
│                │
│                │
└────────────────┘
```

**Step 3: Commit（如有修复）**

```bash
git add -A && git commit -m "fix(mcp): fix any issues found during manual testing"
```

---

## Task 6: 配置 Claude Desktop 集成

**Files:**
- Create: `mcp/README.md`（使用说明）

**Step 1: 在 Claude Desktop 配置文件中添加 MCP Server**

macOS 配置文件路径：`~/Library/Application Support/Claude/claude_desktop_config.json`

添加：
```json
{
  "mcpServers": {
    "asciiflow": {
      "command": "node",
      "args": ["/Users/xingbin/Documents/Study/asciiflow/mcp/dist/index.js"]
    }
  }
}
```

**Step 2: 重启 Claude Desktop，验证工具出现**

在 Claude Desktop 中输入：
> "帮我用 asciiflow 工具画一个简单的登录页面线框图"

Expected: Claude 调用 `canvas_new`、`draw_box`、`add_text`、`canvas_export` 等工具，返回 ASCII 线框图。

**Step 3: Commit README**

```bash
git add mcp/README.md
git commit -m "docs(mcp): add setup and usage instructions"
```

---

## 关键依赖说明

| 文件 | 用途 | DOM 依赖 |
|------|------|---------|
| `client/layer.ts` | 稀疏网格数据结构 | 无 |
| `client/vector.ts` | 2D 坐标 | 无（除 `fromPointerEvent` 静态方法，不使用） |
| `client/text_utils.ts` | Layer ↔ ASCII 文本转换 | 无 |
| `client/draw/utils.ts` | `line()` 折线绘制 | 无 |
| `client/constants.ts` | Unicode 字符集 | 无 |
| `client/common.ts` | `Box` 矩形工具类 | 需确认 |

`client/render_layer.ts` 和 `client/snap.ts` 被 `draw/line.ts` 引用，但 MCP 的 `canvas.ts` 直接调用 `draw/utils.ts` 的 `line()` 函数，绕过了这些依赖。
