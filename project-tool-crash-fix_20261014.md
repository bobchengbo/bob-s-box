# 项目研判报告工具（project-research-tool.html）致命崩溃修复

日期：2026-10-14
提交：fac59f2

## 根因（真正的"程序不跑 / 无数据"元凶）

之前一直以为是浏览器缓存旧 v1.0（v1.0/v1.2 header-badge 并存），实测并非如此——**代码本身从 v1.2 插入 8 城后就崩了**。

- v1.2 用脚本往 CITY_DB 插入 8 城（北京/上海/杭州/成都/西安/大理/丽江/苏州）时，安庆原结尾的 `};` 被保留。
- 原始 3 城时期，安庆是最后一个城市，其结尾 `  }\n};` 中：`  }` 关安庆对象、`};` 关 CITY_DB 顶层。
- 插入 8 城到 `};\n\n// 默认数据` 之前后，安庆不再是最后城市，但 `};` 仍被当成 CITY_DB 顶层关闭 → **CITY_DB 在第 1518 行就提前关闭**，后面的北京等 8 城全变成游离语句。
- 整段内联 JS 语法错误（`'北京':` Unexpected string）→ `<script>` 完全不执行 → 点"开始研判分析"无任何反应、所有模块空白。
- 此前一次修复尝试（把安庆 `};` 改成 `},`）又误留了双 `}`（安庆对象已由 `  }` 关闭，后面又多一个 `}`），雪上加霜。

## 修复

将安庆结尾由：
```
  }
},
  '北京': {
```
合并为：
```
  },
  '北京': {
```
（即 1517 行 `  }` → `  },`，删除 1518 行多余的 `},`）恢复单闭合。CITY_DB 顶层 `};` 保留在苏州之后。

## 验证（全部通过）

1. `node --check` 内联 JS：通过（此前报 `'北京' Unexpected string`）
2. 提取 `const CITY_DB` 用 node 实际求值：`Object.keys` = 11 城（景德镇/黟县/安庆/北京/上海/成都/杭州/西安/大理/丽江/苏州）全部解析成功
3. `render()` 引用的 35 个静态 DOM id + 5 个动态 `s-/b-` id 在 HTML 中全部存在（不会因 null 抛错被 try 吞掉）
4. `runAnalysis → animateLoading → generateReport → render` 流程连贯；未知城市走 DEFAULT_DATA（参考基线数据，不再全"—"）

## 当前状态

- 文件：`/Users/chengbo/.qclaw/workspace-agent-69a72c61/project-research-tool.html`（v2.0，修复后）
- 本地 http server pid 38030（端口 9876）
- serveo 隧道 pid 38047，公网地址：
  `https://9ea7c0bd3165f909-124-65-151-250.serveousercontent.com/project-research-tool.html`
- 隧道历史会挂，打不开告诉我重启即可。

## 已知未完成项（非阻断）

- v2.0 计划的 `normalize()` 函数未写入，`importAI()` 函数存在但 `importAIBtn`/`importArea` 两个 HTML 节点缺失（AI 实时数据导入 UI 未落地）。不影响内置 11 城 + DEFAULT 预览路径。
- 实时抓取架构（Agent 作数据引擎 + HTML 作呈现）已确认方向，柳州机械厂报告（reports/）为验证样例；工具内置 hardcoded 路径仅作快速预览，未知城市给参考基线。
